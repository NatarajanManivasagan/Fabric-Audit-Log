# Who Can See What? A Complete Power BI Access Audit Solution in Microsoft Fabric

*A deep dive: Fabric notebooks harvest RLS/OLS definitions, role memberships, AD group members, and workspace roles into a lakehouse — and a Power BI report on top answers every "who can see what?" question on demand.*

`Microsoft Fabric` · `Power BI` · `RLS / OLS` · `Security & Governance` · ~18 min read

> 📦 **Grab the sample project.** The finished report and semantic model are downloadable as a **PBIP template**
> (`contoso-access-audit-pbip.zip`) wired to fabricated CSV data — it opens in Power BI Desktop with no tenant,
> no service principal and no credentials. Unzip to `C:\ContosoAccessAudit`, open the `.pbip`, hit Refresh, and
> click around while you read. ([Details below.](#download-the-sample-report))
>
> 📓 The whole pipeline is also compiled into one runnable notebook —
> **[`NB_Semantic_Model_Access_Audit.ipynb`](assets/NB_Semantic_Model_Access_Audit.ipynb)** with a built-in validation section
> ([get it below](#get-the-notebook--and-check-its-correct)).

**Contents:** [The problem](#the-problem) · [Architecture](#solution-architecture) · [Prerequisites](#prerequisites) · [Step 1 — Inventory](#step-1--load-the-model-inventory-from-sharepoint) · [Step 2 — RLS/OLS](#step-2--extract-rls-and-ols-with-semantic-link-labs-tom) · [Step 3 — Memberships](#step-3--role-memberships-and-the-query-scale-out-trap) · [Step 4 — Graph](#step-4--expand-ad-groups-into-users-via-microsoft-graph) · [Step 5 — Workspace roles](#step-5--workspace-role-assignments-via-the-fabric-rest-api) · [Notebook & testing](#get-the-notebook--and-check-its-correct) · [Semantic model](#the-semantic-model) · [The report](#the-report) · [Extensions](#extensions) · [Download](#download-the-sample-report) · [Lessons learned](#lessons-learned)

---

## The problem

If you run a self-service BI platform, sooner or later someone from security, audit, or compliance asks a deceptively simple question:

> "Can you show me exactly who can see what in Power BI?"

That one question fans out into five:

1. **Which users have access to each semantic model?**
2. **Which security role is each user tagged to?**
3. **Which AD (Entra ID) group gives them that access?**
4. **What workspace role do they hold?**
5. **What DAX filters (RLS) — and table/column restrictions (OLS) — does each role actually apply?**

For one model you can answer this manually: open *Manage roles* in Desktop, cross-check group membership in the Entra portal, check workspace access in the Service. Across a fleet of self-service models in multiple workspaces, secured with a mix of **row-level security**, **object-level security**, and **Entra ID groups**, the manual answer is stale before you finish collecting it.

So we built the audit *as a data product*: Fabric notebooks harvest security metadata from every model, land it in a lakehouse, and a Power BI report answers all five questions on demand — with a refresh timestamp to prove how fresh the answer is. This post walks through the whole build, with all names and IDs genericized (Contoso-ified) so you can map it to your own tenant.

---

## Solution architecture

![Architecture diagram — a SharePoint inventory workbook and an Entra ID service principal feed a set of Fabric notebooks, which write five Delta tables into the pbi_security_audit lakehouse; a semantic model and a six-page audit report sit on top.](assets/architecture-diagram.png)

*The audit as a one-way data product: inventory workbook + service principal → notebooks → lakehouse → semantic model → report.*

| # | Component | Role |
|---|---|---|
| 1 | **SharePoint: `Model Inventory.xlsx`** | Scope control — a workbook listing the workspaces/models to audit. Adding a model = adding a row. |
| 2 | **Entra ID app registration (service principal)** | The identity that does all the reading: XMLA/TOM, REST APIs, Graph. |
| 3 | **Fabric notebooks** | The harvesters: `semantic-link` (SemPy), `semantic-link-labs`, Power BI & Fabric REST, Microsoft Graph. |
| 4 | **Lakehouse `pbi_security_audit`** | The audit store — one Delta table per question domain. |
| 5 | **Semantic model + 6-page report** | The consumption layer. |

```
SharePoint inventory ──►  Notebook 1: model inventory        ──►  pbi_model_details
                          Notebook 1: RLS/OLS via TOM         ──►  pbi_model_security_audit_details
                          Notebook 1: role memberships        ──►  pbi_model_security_role_association_details
                          Notebook 1: AD group expansion      ──►  pbi_model_security_adgroup_user_details
                          Notebook 2: workspace role scan     ──►  pbi_workspace_access_audit_details
                                                                        │
                                                                        ▼
                                                          Semantic model + audit report
```

---

## Prerequisites

This is the part that takes longer than the code.

**1. App registration.** Create it in Entra ID; note Tenant ID / Client ID, create a client secret. Application permissions (admin-consented): Microsoft Graph `Group.Read.All` (group expansion) and `Sites.Read.All` (inventory download).

> ⚠️ **Secret hygiene.** The snippets read credentials from a `config.json` in lakehouse *Files* to keep the walkthrough simple. In production, use **Azure Key Vault** with `notebookutils.credentials.getSecret(...)`. This is a *security audit* solution — don't let it become the finding.

**2. Tenant settings** (Fabric admin portal): *Service principals can use Fabric APIs* enabled for a group containing the SP; **XMLA endpoint** at least *Read* on the capacity; and — only for the app-audience extension near the end — *Service principals can access read-only admin APIs*.

**3. Workspace access.** Add the SP to every audited workspace (Contributor is comfortable; the error handling below flags missing ones instead of failing the run).

**4. Lakehouse.** Create `pbi_security_audit` **with schema support** and attach it as the notebooks' default lakehouse.

**5. Library.** `%pip install semantic-link-labs` (or add it to your Fabric environment).

### The lakehouse tables

| Table | Grain | Answers |
|---|---|---|
| `pbi_model_details` | workspace × model | audit scope |
| `pbi_model_security_audit_details` | model × role × table (× column) | Q5 — RLS + OLS |
| `pbi_model_security_role_association_details` | model × role × member | Q2/Q3 — role → AD group |
| `pbi_model_security_adgroup_user_details` | group × user | Q1/Q3 — group → people |
| `pbi_workspace_access_audit_details` | workspace × principal | Q4 — workspace roles |
| `pbi_role_personas` | persona | business description of each role's filters |

---

## Step 1 — Load the model inventory from SharePoint

```python
import msal, requests, json
import pandas as pd

with open("/lakehouse/default/Files/pbi_security_audit/config.json") as f:
    config = json.load(f)

app = msal.ConfidentialClientApplication(
    config["CLIENT_ID"],
    authority=f"https://login.microsoftonline.com/{config['TENANT_ID']}",
    client_credential=config["CLIENT_SECRET"],
)
token = app.acquire_token_for_client(
    scopes=["https://graph.microsoft.com/.default"]
)["access_token"]
headers = {"Authorization": f"Bearer {token}"}

# Resolve the SharePoint site, download the workbook via Graph
site = requests.get(
    "https://graph.microsoft.com/v1.0/sites/contoso.sharepoint.com:/sites/BIGovernance",
    headers=headers,
).json()

content = requests.get(
    f"https://graph.microsoft.com/v1.0/sites/{site['id']}/drive/root:"
    f"/PBI Governance/Model Inventory.xlsx:/content",
    headers=headers,
).content

local_path = "/lakehouse/default/Files/pbi_security_audit/Model Inventory.xlsx"
with open(local_path, "wb") as f:
    f.write(content)

df = pd.read_excel(local_path, sheet_name="Self Service Models", dtype=str)
(
    spark.createDataFrame(df)
    .write.format("delta").mode("overwrite")
    .option("overwriteSchema", "true")
    .saveAsTable("pbi_security_audit.dbo.pbi_model_details")
)
```

*(In our environment this is wrapped in a utility that skips the download when the file hasn't changed — worth adding once scheduled.)*

---

## Step 2 — Extract RLS and OLS with semantic-link-labs (TOM)

The heart of the solution. For every model in the inventory we open a **read-only TOM connection** under the service principal and walk `Roles`:

- **RLS:** every `TablePermission.FilterExpression` = the role's DAX filter on that table.
- **OLS:** `MetadataPermission` on tables/columns. Table-level `None` → one collapsed row (`Column_Name = "*"`); otherwise one row per column with effective visibility (`Hidden`/`Visible`).

```python
import json
from sempy.fabric import set_service_principal
from sempy_labs.tom import connect_semantic_model
from pyspark.sql import Row
from pyspark.sql.types import StructType, StructField, StringType

with open("/lakehouse/default/Files/pbi_security_audit/config.json") as f:
    config = json.load(f)

with set_service_principal(
    tenant_id=config["TENANT_ID"],
    client_id=config["CLIENT_ID"],
    client_secret=config["CLIENT_SECRET"],
):
    model_rows = spark.sql("""
        SELECT DISTINCT Workspace_Name, Semantic_Model_Name
        FROM pbi_security_audit.dbo.pbi_model_details
    """).collect()

    security_rows = []

    for r in model_rows:
        workspace, dataset = r["Workspace_Name"], r["Semantic_Model_Name"]
        try:
            with connect_semantic_model(
                dataset=dataset, workspace=workspace, readonly=True
            ) as tom:

                for role in tom.model.Roles:
                    role_name = str(role.Name)

                    role_ols_map = {
                        str(tp.Table.Name): {
                            "Table_Permission": str(tp.MetadataPermission),
                            "Columns": {
                                str(cp.Column.Name): str(cp.MetadataPermission)
                                for cp in tp.ColumnPermissions
                            },
                        }
                        for tp in role.TablePermissions
                    }

                    # ---- RLS filter expressions ---------------------------
                    for perm in role.TablePermissions:
                        expr = perm.FilterExpression
                        if expr and str(expr).strip():
                            security_rows.append(Row(
                                Workspace=workspace, Semantic_Model=dataset,
                                Role_Name=role_name,
                                Table_Name=str(perm.Table.Name),
                                Column_Name=None,
                                Filter=str(expr).strip(), Filter_Type="RLS",
                                OLS_Level=None, Column_Permission=None,
                                Effective_Visibility=None, Error_Message=None,
                            ))

                    # ---- OLS visibility per table/column ------------------
                    for table in tom.model.Tables:
                        table_name = str(table.Name)
                        sec = role_ols_map.get(
                            table_name,
                            {"Table_Permission": "Read", "Columns": {}},
                        )

                        if sec["Table_Permission"] == "None":   # whole table hidden
                            security_rows.append(Row(
                                Workspace=workspace, Semantic_Model=dataset,
                                Role_Name=role_name, Table_Name=table_name,
                                Column_Name="*", Filter=None, Filter_Type="OLS",
                                OLS_Level="Table", Column_Permission="None",
                                Effective_Visibility="Hidden", Error_Message=None,
                            ))
                            continue

                        for column in table.Columns:
                            if str(column.Type) == "RowNumber":
                                continue
                            col = str(column.Name)
                            col_perm = sec["Columns"].get(col, "Read")
                            security_rows.append(Row(
                                Workspace=workspace, Semantic_Model=dataset,
                                Role_Name=role_name, Table_Name=table_name,
                                Column_Name=col, Filter=None, Filter_Type="OLS",
                                OLS_Level="Column" if col in sec["Columns"] else "None",
                                Column_Permission=col_perm,
                                Effective_Visibility="Hidden" if col_perm == "None" else "Visible",
                                Error_Message=None,
                            ))

        except Exception as e:
            print(f"Error processing {dataset}: {e} "
                  "— check that the service principal is added to the workspace")
            security_rows.append(Row(
                Workspace=workspace, Semantic_Model=dataset, Role_Name=None,
                Table_Name=None, Column_Name=None, Filter=None,
                Filter_Type="ERROR", OLS_Level=None, Column_Permission=None,
                Effective_Visibility=None, Error_Message=str(e),
            ))

    schema = StructType([
        StructField(c, StringType(), True) for c in [
            "Workspace", "Semantic_Model", "Role_Name", "Table_Name",
            "Column_Name", "Filter", "Filter_Type", "OLS_Level",
            "Column_Permission", "Effective_Visibility", "Error_Message",
        ]
    ])

    (
        spark.createDataFrame(security_rows, schema=schema)
        .write.format("delta").mode("overwrite")
        .option("overwriteSchema", "true")
        .saveAsTable("pbi_security_audit.dbo.pbi_model_security_audit_details")
    )
```

Sample output (fabricated):

| Workspace | Semantic_Model | Role_Name | Table_Name | Column_Name | Filter | Filter_Type | Effective_Visibility |
|---|---|---|---|---|---|---|---|
| Contoso Finance Self-Service BI | Finance Semantic Model | Cost Center - Dynamic | Dim Cost Center | | `[Cost Center Code] IN {"CC-1001","CC-1044"}` | RLS | |
| Contoso Finance Self-Service BI | Finance Semantic Model | Finance - All | Dim Employee | Salary | | OLS | Hidden |
| Contoso Sales Self-Service BI | Sales Semantic Model | Region - West | Dim Region | | `[Region] = "WEST"` | RLS | |

Two design choices worth calling out:

- **Error rows instead of failures.** If the SP isn't in a workspace, we record `Filter_Type = "ERROR"` with the message — "we can't see this model" is itself an audit finding, not a silent gap.
- **Effective visibility, not raw permissions.** OLS metadata only lists exceptions; by walking every table/column and defaulting to `Read`, the output answers "what does this role see?" without the consumer knowing OLS semantics.

---

## Step 3 — Role memberships (and the query scale-out trap)

Role definitions are half the story; we also need **who is tagged to each role**. TOM exposes `role.Members`, and each member's `MemberID` is the **Entra object ID** of the group or user — the join key for the next step.

**The trap:** our first version used DAX `INFO` functions instead of TOM:

```dax
EVALUATE
VAR _Roles =
    SELECTCOLUMNS(INFO.ROLES(),
        "RoleID", [ID], "Role_Name", [Name], "Model_Permission", [ModelPermission])
VAR _Members =
    SELECTCOLUMNS(INFO.ROLEMEMBERSHIPS(),
        "RoleID", [RoleID], "Member_Name", [MemberName], "Member_ID", [MemberID])
RETURN
    NATURALLEFTOUTERJOIN(_Roles, _Members)
```

Great interactively — until it hit a model with **query scale-out** enabled. Scale-out routes reads to read-only replicas, where security metadata isn't reliably served; runs failed or returned incomplete membership for exactly those models.

**The workaround:** check `queryScaleOutSettings` via the Power BI REST API; if scale-out is on, temporarily set `maxReadOnlyReplicas = 0`, wait for replicas to drain, extract via TOM on the primary, then **restore the original setting**:

```python
import msal, requests, json, time
import pandas as pd
from sempy.fabric import set_service_principal
from sempy_labs.tom import connect_semantic_model

with open("/lakehouse/default/Files/pbi_security_audit/config.json") as f:
    config = json.load(f)

app = msal.ConfidentialClientApplication(
    config["CLIENT_ID"],
    authority=f"https://login.microsoftonline.com/{config['TENANT_ID']}",
    client_credential=config["CLIENT_SECRET"],
)
token = app.acquire_token_for_client(
    scopes=["https://analysis.windows.net/powerbi/api/.default"]
)["access_token"]
headers = {"Authorization": f"Bearer {token}", "Content-Type": "application/json"}
PBI = "https://api.powerbi.com/v1.0/myorg"


def get_workspace_id(name):
    groups = requests.get(f"{PBI}/groups", headers=headers).json()["value"]
    return next((g["id"] for g in groups if g["name"] == name), None)

def get_dataset_id(ws_id, name):
    ds = requests.get(f"{PBI}/groups/{ws_id}/datasets", headers=headers).json()["value"]
    return next((d["id"] for d in ds if d["name"] == name), None)

def get_scaleout_settings(ws_id, ds_id):
    r = requests.get(f"{PBI}/groups/{ws_id}/datasets/{ds_id}", headers=headers)
    return r.json().get("queryScaleOutSettings", {})

def update_scaleout(ws_id, ds_id, settings):
    requests.patch(
        f"{PBI}/groups/{ws_id}/datasets/{ds_id}", headers=headers,
        data=json.dumps({"queryScaleOutSettings": settings}),
    ).raise_for_status()


model_rows = spark.sql("""
    SELECT DISTINCT Workspace_Name, Semantic_Model_Name
    FROM pbi_security_audit.dbo.pbi_model_details
""").collect()

all_results = []

for row in model_rows:
    workspace, dataset = row["Workspace_Name"], row["Semantic_Model_Name"]
    print(f"Processing: {workspace} | {dataset}")
    try:
        ws_id = get_workspace_id(workspace)
        ds_id = get_dataset_id(ws_id, dataset)
        if not ws_id or not ds_id:
            print("  Workspace or dataset not found — skipping.")
            continue

        original = get_scaleout_settings(ws_id, ds_id)
        scaleout_enabled = original.get("maxReadOnlyReplicas", 0) != 0

        if scaleout_enabled:
            print("  Scale-out enabled — disabling temporarily...")
            update_scaleout(ws_id, ds_id, {"maxReadOnlyReplicas": 0})
            time.sleep(300)  # allow replicas to sync out

        with set_service_principal(
            tenant_id=config["TENANT_ID"],
            client_id=config["CLIENT_ID"],
            client_secret=config["CLIENT_SECRET"],
        ):
            with connect_semantic_model(
                workspace=workspace, dataset=dataset, readonly=True
            ) as tom:
                for role in tom.model.Roles:
                    for member in role.Members:
                        all_results.append({
                            "Workspace": workspace,
                            "Semantic_Model": dataset,
                            "Role_Name": str(role.Name),
                            "Member_ID": str(member.MemberID),  # Entra object ID
                        })

        if scaleout_enabled:
            print("  Restoring original scale-out settings...")
            update_scaleout(ws_id, ds_id, original)

    except Exception as e:
        print("  Error:", e)

if all_results:
    (
        spark.createDataFrame(pd.DataFrame(all_results)).distinct()
        .write.format("delta").mode("overwrite")
        .option("overwriteSchema", "true")
        .saveAsTable("pbi_security_audit.dbo.pbi_model_security_role_association_details")
    )
```

> 💡 Budget for the drain wait per scale-out model, and make sure the *restore* runs even on failure (`try/finally` if you extend this). The PATCH needs write access on the dataset — workspace Contributor or above.

---

## Step 4 — Expand AD groups into users via Microsoft Graph

Roles map to **Entra ID groups**, not individuals. We follow a naming convention (`pbi-sec-*`) for all BI security groups, so one filtered Graph query finds them all, and a second pass pulls members:

```python
import msal, requests, json

with open("/lakehouse/default/Files/pbi_security_audit/config.json") as f:
    config = json.load(f)

app = msal.ConfidentialClientApplication(
    config["CLIENT_ID"],
    authority=f"https://login.microsoftonline.com/{config['TENANT_ID']}",
    client_credential=config["CLIENT_SECRET"],
)
token = app.acquire_token_for_client(
    scopes=["https://graph.microsoft.com/.default"]
)["access_token"]
headers = {"Authorization": f"Bearer {token}"}

url = ("https://graph.microsoft.com/v1.0/groups"
       "?$filter=startswith(displayName,'pbi-sec')&$select=id,displayName")
groups = []
while url:
    data = requests.get(url, headers=headers).json()
    groups.extend(data.get("value", []))
    url = data.get("@odata.nextLink")

rows = []
for g in groups:
    url = (f"https://graph.microsoft.com/v1.0/groups/{g['id']}/members"
           "?$select=id,displayName,userPrincipalName")
    while url:
        data = requests.get(url, headers=headers).json()
        rows += [
            {
                "Group_Name": g["displayName"],
                "Group_Object_ID": g["id"],
                "User_Name": m.get("displayName"),
                "User_Object_ID": m.get("id"),
                "User_Principal_Name": m.get("userPrincipalName"),
            }
            for m in data.get("value", [])
            if m.get("@odata.type") == "#microsoft.graph.user"
        ]
        url = data.get("@odata.nextLink")

if rows:
    (
        spark.createDataFrame(rows)
        .write.format("delta").mode("overwrite")
        .option("overwriteSchema", "true")
        .saveAsTable("pbi_security_audit.dbo.pbi_model_security_adgroup_user_details")
    )
```

Note what happened across Steps 3 and 4: the role member's `MemberID` (TOM) and the group's `id` (Graph) are **the same Entra object ID** — the join that lets the report connect *role → group → human being*.

> 💡 No naming convention for BI security groups yet? This project is a good excuse to introduce one — the alternative is enumerating the whole tenant. And `/members` returns *direct* members only; for nested groups use `/transitiveMembers`.

---

## Step 5 — Workspace role assignments via the Fabric REST API

Model-level security means nothing if someone is workspace Admin. `GET /v1/workspaces/{id}/roleAssignments` returns every principal with a role; where the principal is a group we expand it to people using the Step 4 table:

```python
import msal, requests, json
import pandas as pd

with open("/lakehouse/default/Files/pbi_security_audit/config.json") as f:
    config = json.load(f)

WORKSPACE_NAMES = [
    "Contoso Finance Self-Service BI",
    "Contoso Sales Self-Service BI",
    "Contoso Operations Self-Service BI",
]

app = msal.ConfidentialClientApplication(
    config["CLIENT_ID"],
    authority=f"https://login.microsoftonline.com/{config['TENANT_ID']}",
    client_credential=config["CLIENT_SECRET"],
)
token = app.acquire_token_for_client(
    scopes=["https://api.fabric.microsoft.com/.default"]
)["access_token"]
headers = {"Authorization": f"Bearer {token}"}
FABRIC = "https://api.fabric.microsoft.com"

all_ws = requests.get(f"{FABRIC}/v1/workspaces", headers=headers).json()["value"]
lookup = {w["displayName"]: w["id"] for w in all_ws}
workspaces = [{"id": lookup[n], "name": n} for n in WORKSPACE_NAMES if n in lookup]

frames = []
for ws in workspaces:
    data = requests.get(
        f"{FABRIC}/v1/workspaces/{ws['id']}/roleAssignments", headers=headers
    ).json().get("value", [])
    if not data:
        continue
    df = pd.json_normalize(data).rename(columns={
        "principal.displayName": "Name",
        "principal.type": "Type",
        "principal.userDetails.userPrincipalName": "Email",
        "role": "Role",
    })
    df["Workspace"] = ws["name"]
    frames.append(df[["Workspace", "Name", "Email", "Type", "Role"]])

combined = pd.concat(frames, ignore_index=True)

# Expand group principals to individual users
df_ad = spark.sql("""
    SELECT Group_Name, User_Principal_Name AS Group_Member_Email
    FROM pbi_security_audit.dbo.pbi_model_security_adgroup_user_details
""").toPandas()

combined = combined.merge(
    df_ad, how="left", left_on="Name", right_on="Group_Name"
).drop(columns=["Group_Name"])
combined["Email"] = combined["Group_Member_Email"].combine_first(combined["Email"])
combined = combined.drop(columns=["Group_Member_Email"])

(
    spark.createDataFrame(combined)
    .write.format("delta").mode("overwrite")
    .option("overwriteSchema", "true")
    .saveAsTable("pbi_security_audit.dbo.pbi_workspace_access_audit_details")
)
```

| Workspace | Name | Email | Type | Role |
|---|---|---|---|---|
| Contoso Finance Self-Service BI | pbi-sec-finance-developers | alex.chen@contoso.com | Group | Contributor |
| Contoso Finance Self-Service BI | Jordan Lee | jordan.lee@contoso.com | User | Admin |
| Contoso Sales Self-Service BI | svc-pbi-audit | | ServicePrincipal | Contributor |

**Scheduling:** both notebooks are idempotent (`overwrite`), so a Fabric data pipeline runs them sequentially off-hours, followed by a semantic model refresh. The scale-out workaround can add ~5 minutes per affected model — plan accordingly.

![Fabric data pipeline canvas — a schedule trigger runs the Semantic Model Security Details notebook, then the Workspace Access notebook on success, then a semantic model refresh on success.](assets/pipeline-canvas.png)

*The scheduled pipeline: two notebook activities chained on success, then a semantic model refresh so the report always carries a fresh timestamp.*

---

## Get the notebook — and check it's correct

All five steps above are compiled into one **self-contained** Fabric notebook:
**[`NB_Semantic_Model_Access_Audit.ipynb`](assets/NB_Semantic_Model_Access_Audit.ipynb)** — no SharePoint, no config
file, no external module. Import it (*Workspace → Import → Notebook*), then fill in a single **CONFIG** cell: your
service-principal `TENANT_ID` / `CLIENT_ID` / `CLIENT_SECRET`, and a `MODELS` list of the `(workspace, semantic model)`
pairs to audit. Run the install cell, **restart the kernel** (service-principal auth needs `semantic-link` ≥ 0.12.0),
then run top to bottom. It's **DataFrame-first** — each step builds a Spark DataFrame you can eyeball, and nothing is
written to the lakehouse until you flip `persist` on.

**How do you know the output is right?** The notebook ends with a **validation section** that:

- confirms all five result sets built and are non-empty;
- runs a **referential-integrity** check — every role member (group) should resolve to an AD group via `Member_ID = Group_Object_ID` (a non-zero count is expected only for individual-user members, or groups outside the `pbi-sec-*` convention);
- surfaces the `Filter_Type = "ERROR"` rows, so a model the service principal *couldn't* read is a visible finding, not a silent gap;
- answers all five audit questions in SQL for one model — if those return sensible rows, the pipeline is correct end to end.

Testing on a single model first is as easy as putting one pair in `MODELS`.

### Two ways to land the data — tables or CSV

The pipeline builds five DataFrames; how you *land* them decides how the semantic model reads them. The notebook's
`publish()` helper carries both options, commented out (it writes nothing by default):

**Production — Delta tables.** Keep scope in an **Excel inventory** workbook and credentials in **`config.json`** (the
fuller pattern shown in Steps 1–5), then persist each result as a managed Delta table:

```python
df.write.format("delta").mode("overwrite").option("overwriteSchema", "true").saveAsTable(f"{LH}.{name}")
```

Point an **import or Direct Lake** semantic model at those tables and build the report.

**Portable — CSV.** Or stay DataFrame-first and export each result to a CSV:

```python
df.toPandas().to_csv(f"/lakehouse/default/Files/audit_csv/{name}.csv", index=False)
```

Point a model at those CSVs instead — exactly what the [downloadable sample report](#download-the-sample-report) does: its Power Query reads the same snake_case CSVs, and switching that model to a live lakehouse is a one-line change.

### What a correct run looks like

I ran those checks against the fabricated sample dataset that ships with the template — the same schemas the notebook writes. Real output:

```
pbi_model_details                                     5 rows   ok
pbi_model_security_audit_details                     65 rows   ok
pbi_model_security_role_association_details          10 rows   ok
pbi_model_security_adgroup_user_details              31 rows   ok
pbi_workspace_access_audit_details                   16 rows   ok
```

**Zero orphans** on the referential-integrity check — every `Member_ID` resolved to a `Group_Object_ID`, the strongest single signal that Steps 3 and 4 agree with each other. The `Filter_Type = "ERROR"` row for *Legacy Inventory Semantic Model* appeared as data, exactly as intended.

Asking "who can see `Finance Semantic Model`?" returned 10 rows — four roles, resolved through their groups to actual people. The RLS check returned the five filters that model applies, including the dynamic one:

```
Role_Name             | Table_Name               | Filter
Cost Center - Dynamic | Dim Security Cost Center | [User Principal Name] = USERPRINCIPALNAME()
Cost Center - West    | Dim Cost Center          | [Cost Center Code] IN {"CC-1001","CC-1044","CC-1099"}
Entity - EMEA         | Dim Entity               | [Entity Code] IN {"EMEA-01","EMEA-02"}
...
```

And then the check that earns its keep — joining role membership to workspace roles:

```
User_Name  | Role_Name     | Workspace                       | Workspace_Role
Jordan Lee | Finance - All | Contoso Finance Self-Service BI | Admin
```

Jordan Lee sits in an RLS role *and* holds workspace **Admin** — which means that carefully-scoped role is doing nothing for them, because workspace Admins bypass RLS entirely. That single row is the whole reason this audit exists.

> 💡 Writing this section caught a bug in my own query: selecting only `Workspace, Name, Type, Role` from the workspace table renders a group expanded to four members as four identical-looking rows. Include `Email`. The notebook has the fixed version.

---

## The semantic model

The audit model is a small **import-mode** model over the lakehouse's **SQL analytics endpoint**. Every table uses the same Power Query pattern — connect, pick table, rename to friendly names:

```powerquery-m
let
    Source = Sql.Databases("xxxxxxxx.datawarehouse.fabric.microsoft.com"),
    pbi_security_audit = Source{[Name="pbi_security_audit"]}[Data],
    audit = pbi_security_audit{[Schema="dbo",Item="pbi_model_security_audit_details"]}[Data],
    Renamed = Table.RenameColumns(audit, {
        {"Workspace", "Workspace Name"}, {"Semantic_Model", "Semantic Model Name"},
        {"Role_Name", "Role Name"}, {"Table_Name", "Table Name"},
        {"Filter_Type", "Filter Type"}, {"Column_Name", "Column Name"},
        {"Column_Permission", "Column Permission"}, {"OLS_Level", "OLS Level"},
        {"Effective_Visibility", "Visibility"}
    }),
    // Composite key used to relate audit rows to role membership rows
    WithKey = Table.AddColumn(Renamed, "Key",
        each [Workspace Name] & " | " & [Semantic Model Name] & " | " & [Role Name])
in
    WithKey
```

Six tables — **Audit Details**, **Role AD Group Association**, **AD Group User Association**, **Workspace Access Details**, **Dynamic Role Persona Details** (a persona → filter-attribute matrix filtered to `Active = "1"`), and a generated **Last Updated Time** — joined by three relationships:

```
Audit Details[Key] ─────────────── Role AD Group Association[Key]
        (Workspace | Model | Role composite key)

Role AD Group Association[Group Object ID] ─── AD Group User Association[Group Object ID]
        ★ the linchpin

Dynamic Role Persona Details[AD_Group] ─────── AD Group User Association[Group Name]
```

![Data model diagram — Audit Details joins to Role AD Group Association on a composite Workspace-Model-Role key; Role AD Group Association joins to AD Group User Association on Group Object ID; Dynamic Role Persona Details joins to AD Group User Association on AD group name. Workspace Access Details and Last Updated Time stand alone.](assets/data-model-diagram.png)

*Three many-to-many, single-direction relationships. The `Group Object ID` join (★) is the linchpin that connects a role to the people inside it.*

Design notes:

- **The composite `Key`.** Both role-grain tables get `Key = Workspace | Model | Role` in Power Query. A relationship on `Role Name` alone breaks the moment two models both define a role called "Viewer"; the composite key gives one honest relationship.
- **The Group Object ID linchpin.** TOM's `MemberID` = Graph's group `id` = the same Entra object ID. No name matching, no fuzzy joins.
- **`Last Updated Time`.** A one-row M table stamping the refresh time — every page displays it, preempting the auditor's first question (*"as of when?"*):

```powerquery-m
let
    LocalTime  = DateTimeZone.SwitchZone(DateTimeZone.UtcNow(), -6),
    Formatted  = DateTime.ToText(DateTimeZone.RemoveZone(LocalTime),
                                 "yyyy-MM-dd HH:mm:ss") & " CST",
    Source     = #table({"Last Updated"}, {{Formatted}})
in
    Source
```

- **The persona matrix.** Raw RLS DAX is hostile to business readers. `pbi_role_personas` describes each persona in plain columns (business unit, cost centers include/exclude, accounts, entities, source-system slices). Shown next to the raw DAX, it lets reviewers catch "the filter says X but the persona sheet says Y" drift — exactly what an access audit exists to find.

---

## The report

Six pages, one layout language (title, last-updated stamp, slicer rail).

> 📷 Every page below is shown on the **fabricated sample data** that ships with the downloadable template
> ([details below](#download-the-sample-report)) — so every workspace, model, role, group, filter, and person here is invented.

1. **Role – Persona Details** *(landing)* — Role → AD Group → User → UPN with workspace/role/group/user slicers. Answers **Q1/Q2/Q3** in one glance, showing *people*, not just group names.
2. **Security Details (RLS)** — Role → Table → Filter (the DAX), page-filtered to `Filter Type = "RLS"`. A bookmark-driven **RLS/OLS toggle** (two buttons) flips between this page and the next.
3. **OLS** *(hidden; via toggle)* — Role → Table → Column → Permission, pre-filtered to `Column Permission = "None"` so the default view shows only what's hidden; an `OLS Level` slicer separates table-level blackouts (`*` rows) from column-level ones. Together with page 2: **Q5**.
4. **View Security Details** *(hidden drill-through)* — right-click any role → consolidated view: RLS filter matrix next to the persona description. One role, complete picture.
5. **AD Group Details** — Group → User → UPN, for questions that arrive group-first ("who's in `pbi-sec-finance-costcenter-viewers`?"). **Q3.**
6. **Workspace Access Details** — Workspace → principal → email → workspace role. **Q4** — and the page that regularly finds the classic gap: a user with a modest RLS role who is *also* workspace Admin, meaning their RLS doesn't apply at all.

**Role – Persona Details** *(landing)* — who has access, via which role, group, and persona attributes:

![Role & Persona page — a table listing each semantic model, role, AD group, business unit, region, and which source systems (ERP/Expense/HCM) the persona may see.](assets/screenshots/report-role-persona.png)

**Security Details (RLS)** — each role's DAX filter, per table:

![Security Details page — each role's RLS DAX filter shown per table, including a dynamic USERPRINCIPALNAME() rule, IN-list filters, and simple equality filters.](assets/screenshots/report-security-rls.png)

**OLS** — what each role can't even see (table- and column-level):

![OLS page — hidden objects per role, mixing table-level blackouts (entire table) with column-level ones such as Salary, Bonus, and Margin Amount, each marked Hidden.](assets/screenshots/report-ols.png)

**AD Group Details** — group → user membership:

![AD Group Details page — each pbi-sec-* security group expanded to its member users and their principal names.](assets/screenshots/report-ad-group-details.png)

**Workspace Access Details** — workspace roles, with the RLS-vs-Admin gap in plain sight:

![Workspace Access Details page — each workspace's principals (groups, users, service principals) with their email, principal type, and colour-coded workspace role (Admin, Member, Contributor, Viewer).](assets/screenshots/report-workspace-access.png)

---

## Extensions

**A. App audiences (Admin API).** Apps add another access path. `admin/apps` + `admin/apps/{id}/users` enumerate audiences — but workspace access is *not* enough for `admin/*` routes; the SP must be covered by the tenant setting *Service principals can access read-only admin APIs* (we learned via 403).

```python
apps = requests.get(
    "https://api.powerbi.com/v1.0/myorg/admin/apps?$top=5000",
    headers=headers,
).json()["value"]

for app_info in [a for a in apps if a.get("workspaceId") == workspace_id]:
    users = requests.get(
        f"https://api.powerbi.com/v1.0/myorg/admin/apps/{app_info['id']}/users",
        headers=headers,
    ).json()["value"]
    # -> pbi_security_audit.dbo.pbi_app_audience_details
```

**B. Direct dataset permissions.** Individual Build/Read grants on the dataset are a third access route. Anyone in `datasets/{id}/users` but not in `groups/{id}/users` has a **direct grant** — exactly the one-off access audits exist to find:

```python
ws_users = requests.get(f"{PBI}/groups/{ws_id}/users", headers=headers).json()["value"]
ds_users = requests.get(f"{PBI}/groups/{ws_id}/datasets/{ds_id}/users",
                        headers=headers).json()["value"]

ws_ids = {u.get("identifier") for u in ws_users}
direct_grants = [u for u in ds_users if u.get("identifier") not in ws_ids]
```

**C. Lineage (bonus).** `fabric.list_reports()` includes each report's dataset ID; join to `fabric.list_datasets()` and you have report → model lineage — useful when an RLS change is proposed and someone asks "what does this break?".

---

## Download the sample report

Rather than describe the report and leave you to rebuild it, here it is: **`contoso-access-audit-pbip.zip`** —
the complete six-page report and semantic model as a **PBIP project**, shipping with **fabricated sample data**
(3 workspaces, 4 models, 10 roles, 12 AD groups, 30 users, RLS filters and both flavours of OLS) loaded from CSVs.
It opens in Power BI Desktop with **no tenant, no gateway, no service principal and no credentials**:

1. Unzip to `C:\ContosoAccessAudit`.
2. Open `Contoso Semantic Model Security Details.pbip`.
3. Click **Refresh**.

Unzipped elsewhere? Point the **`CsvFolder`** parameter at your `Data` folder (*Home → Transform data → Edit
parameters*) and refresh.

![The template open in Power BI Desktop, refreshed on the sample CSV data — the AD Group page rendering with the model's tables listed in the Data pane.](assets/screenshots/template-in-desktop.png)

*The downloadable template open in Power BI Desktop, refreshed on the fabricated sample data — no tenant or credentials required.*

The CSV headers are deliberately the same snake_case names the notebooks above write (`Workspace`,
`Semantic_Model`, `Role_Name`, …), so the template's Power Query is the *same* code the real solution runs.
Pointing it at your own lakehouse is a one-step change — each table's M opens with a comment showing the swap:

```powerquery-m
// Sample (the template):
Source = Csv.Document(File.Contents(CsvFolder & "\pbi_model_security_audit_details.csv"),
                      [Delimiter=",", Encoding=65001, QuoteStyle=QuoteStyle.Csv]),
dbo_pbi_model_security_audit_details = Table.PromoteHeaders(Source, [PromoteAllScalars=true]),

// Production (your Fabric lakehouse):
Source   = Sql.Databases("<your-endpoint>.datawarehouse.fabric.microsoft.com"),
Database = Source{[Name="pbi_security_audit"]}[Data],
dbo_pbi_model_security_audit_details = Database{[Schema="dbo",Item="pbi_model_security_audit_details"]}[Data],
```

Everything downstream — the renames, the `Key` column, the relationships, all six pages — stays exactly as-is.

> One deliberate detail in the sample: a `Filter_Type = ERROR` row for *Legacy Inventory Semantic Model*, so you
> can see how the pipeline records "the service principal couldn't read this model" as data instead of failing the run.

---

## Lessons learned

1. **Query scale-out silently breaks security extraction.** Read replicas don't reliably serve role/membership metadata. Detect, drain, extract on the primary, restore. Our biggest time sink.
2. **INFO functions are great interactively, wrong for this pipeline.** `INFO.ROLES()` worked — until scale-out. TOM via `semantic-link-labs` is authoritative and exposes OLS cleanly too.
3. **Record errors as data.** "SP not in workspace" becomes an `ERROR` row on the report — a visible finding, not a silent hole.
4. **Object IDs beat names as join keys.** Names get renamed; object IDs don't.
5. **Admin APIs are a different permission tier.** `groups/*` needs workspace access; `admin/*` needs the tenant setting. Budget the admin-team conversation.
6. **Translate DAX for your auditors.** The persona matrix turned the report from "a developer tool" into "the thing compliance signs off on".

---

## Wrapping up

What started as an awkward compliance question — *"who can see what?"* — became a small data product: two notebooks, five Delta tables, one semantic model, six report pages. The audit that used to take days of screenshotting role dialogs is now a scheduled refresh, and every answer carries its own timestamp.

If you build something similar (or find a better way around the scale-out drain), I'd love to hear about it in the comments.

*All workspace names, model names, group names, and IDs in this post are fictionalized. The patterns are real.*
