# Who Can See What? Auditing Power BI Semantic Model Security in Microsoft Fabric — Part 1: Building the Audit Pipeline

*Part 1 of 2 — this post covers the architecture, prerequisites, lakehouse design, and the Fabric notebooks that collect the security data. [Part 2](Part%202%20-%20The%20Semantic%20Model%20and%20Audit%20Report.md) covers the semantic model and the audit report built on top of it.*

`Microsoft Fabric` · `Power BI` · `RLS / OLS` · `Security & Governance` · ~12 min read

> 📦 **Grab the sample project.** The finished report and semantic model are available as a downloadable
> **PBIP template** (`contoso-access-audit-pbip.zip`), wired to fabricated CSV data so it opens in Power BI
> Desktop with no tenant, no service principal and no credentials. Unzip, hit Refresh, and click around
> while you read. Details in [Part 2](Part%202%20-%20The%20Semantic%20Model%20and%20Audit%20Report.md).
>
> 📓 The whole pipeline is also compiled into one runnable notebook —
> **[`NB_Semantic_Model_Access_Audit.ipynb`](NB_Semantic_Model_Access_Audit.ipynb)** ([get it below](#get-the-notebook--and-check-its-correct)).

---

## The problem

If you run a self-service BI platform, sooner or later someone from security, audit, or compliance walks up with a deceptively simple question:

> "Can you show me exactly who can see what in Power BI?"

And that one question fans out into five:

1. **Which users have access to each semantic model?**
2. **Which security role is each user tagged to?**
3. **Which AD (Entra ID) group gives them that access?**
4. **What workspace role do they hold?**
5. **What DAX filters (RLS) — and table/column restrictions (OLS) — does each role actually apply?**

For a single model you can answer this by opening the model in Power BI Desktop, checking *Manage roles*, then cross-checking group memberships in the Entra portal, then checking workspace access in the Service… and by the time you finish model number three, the answers for model number one are already stale.

We had this exact ask across a fleet of self-service semantic models spread over multiple workspaces, each secured with a mix of **row-level security (RLS)**, **object-level security (OLS)**, and **Entra ID groups**. Manually screenshotting role dialogs was not going to survive an audit.

So we built the audit *as a data product*: Fabric notebooks harvest the security metadata from every model, land it in a lakehouse, and a Power BI report on top answers all five questions on demand — with a refresh timestamp to prove how fresh the answer is.

This post walks through how to build it, with all names and IDs genericized (Contoso-ified) so you can map it to your own tenant.

---

## Solution architecture

![Architecture diagram — a SharePoint inventory workbook and an Entra ID service principal feed a set of Fabric notebooks, which write five Delta tables into the pbi_security_audit lakehouse; a semantic model and a six-page audit report sit on top.](assets/architecture-diagram.png)

*The audit as a one-way data product: inventory workbook + service principal → notebooks → lakehouse → semantic model → report.*

The moving parts:

| # | Component | Role in the solution |
|---|---|---|
| 1 | **SharePoint: `Model Inventory.xlsx`** | The scope control. A simple workbook listing which workspaces/semantic models to audit — maintained by the BI team, no code change needed to add a model. |
| 2 | **Entra ID app registration (service principal)** | The identity that does all the reading: semantic model metadata via XMLA, REST APIs, and Graph. |
| 3 | **Fabric notebooks** | The harvesters. Python notebooks using `semantic-link` (SemPy), `semantic-link-labs`, the Power BI & Fabric REST APIs, and Microsoft Graph. |
| 4 | **Fabric lakehouse: `pbi_security_audit`** | The audit store. Delta tables, one per question domain. |
| 5 | **Semantic model + report** | The consumption layer (covered in Part 2). |

Data flows in one direction:

```
SharePoint inventory ──►  Notebook 1: model inventory        ──►  pbi_model_details
                          Notebook 1: RLS/OLS via TOM         ──►  pbi_model_security_audit_details
                          Notebook 1: role memberships        ──►  pbi_model_security_role_association_details
                          Notebook 1: AD group expansion      ──►  pbi_model_security_adgroup_user_details
                          Notebook 2: workspace role scan     ──►  pbi_workspace_access_audit_details
                                                                        │
                                                                        ▼
                                                        Semantic model + audit report (Part 2)
```

---

## Prerequisites

This is the part that takes longer than the code. Get these right first.

### 1. App registration (service principal)

Create an app registration in Entra ID and note the **Tenant ID**, **Client ID**, and create a **client secret**.

API permissions (application permissions, admin-consented):

| API | Permission | Used for |
|---|---|---|
| Microsoft Graph | `Group.Read.All` | Expanding AD groups to user members |
| Microsoft Graph | `Sites.Read.All` | Downloading the inventory workbook from SharePoint |

> ⚠️ **Secret hygiene.** The snippets below read credentials from a `config.json` in the lakehouse *Files* area to keep the walkthrough simple. In production, store the secret in **Azure Key Vault** and read it with `notebookutils.credentials.getSecret(...)` instead of a file in your lakehouse. This is a *security audit* solution — don't let it become the finding.

### 2. Power BI / Fabric tenant settings

In the Fabric admin portal:

- **Developer settings → Service principals can use Fabric APIs** — enabled for a security group containing your SP.
- **Capacity settings → XMLA endpoint** — at least **Read** on the capacity hosting the audited models (the TOM connection below rides the XMLA endpoint).
- *(Only for the app-audience extension in Part 2)* **Admin API settings → Service principals can access read-only admin APIs**.

### 3. Workspace access

Add the service principal to **every workspace you want to audit** (Contributor is a comfortable choice; the notebook's error handling flags any workspace where it's missing rather than failing the whole run).

### 4. The lakehouse

Create a lakehouse named `pbi_security_audit` **with schema support**, and attach it as the default lakehouse of the notebooks. Everything below writes to the `dbo` schema.

### 5. Notebook environment

The main notebook uses [`semantic-link-labs`](https://github.com/microsoft/semantic-link-labs). Either add it to your workspace's Fabric environment or install it inline:

```python
%pip install semantic-link-labs
```

---

## The lakehouse tables

One Delta table per question domain — deliberately flat and boring, because the report does the joining:

| Table | Grain | Answers |
|---|---|---|
| `pbi_model_details` | workspace × semantic model | Audit scope (from the inventory workbook) |
| `pbi_model_security_audit_details` | model × role × table (× column for OLS) | Q5 — RLS DAX filters + OLS restrictions |
| `pbi_model_security_role_association_details` | model × role × member | Q2/Q3 — which AD group sits in which role |
| `pbi_model_security_adgroup_user_details` | AD group × user | Q1/Q3 — who is actually inside those groups |
| `pbi_workspace_access_audit_details` | workspace × principal | Q4 — workspace roles |
| `pbi_role_personas` | persona | (Part 2) business-friendly description of what each role filters |

---

## Step 1 — Load the model inventory from SharePoint

The audit scope lives in `Model Inventory.xlsx` (sheet `Self Service Models`, columns `Workspace_Name`, `Semantic_Model_Name`) on a SharePoint site the BI team already uses. Adding a model to the audit = adding a row to the sheet.

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

# 1. Resolve the SharePoint site, then download the workbook via Graph
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

# 2. Land the inventory sheet as a Delta table
df = pd.read_excel(local_path, sheet_name="Self Service Models", dtype=str)

(
    spark.createDataFrame(df)
    .write.format("delta")
    .mode("overwrite")
    .option("overwriteSchema", "true")
    .saveAsTable("pbi_security_audit.dbo.pbi_model_details")
)
```

*(In our environment this step is wrapped in a small utility module that also skips the download when the file hasn't changed on SharePoint — a worthwhile optimization once the pipeline is scheduled.)*

---

## Step 2 — Extract RLS and OLS definitions with semantic-link-labs (TOM)

This is the heart of the solution. For every model in the inventory, we open a **read-only Tabular Object Model (TOM)** connection using `semantic-link-labs`, impersonating the service principal via `set_service_principal`, and walk the model's `Roles` collection:

- **RLS**: every `TablePermission` with a `FilterExpression` gives us the role's DAX filter on that table.
- **OLS**: `MetadataPermission` on tables and columns tells us what the role can't even *see*. Two cases matter:
  - **Table-level OLS** (`Table_Permission = "None"`) — the whole table is hidden; we write one collapsed row (`Column_Name = "*"`) instead of exploding every column.
  - **Column-level OLS** — for readable tables we emit one row per column with its effective visibility (`Hidden`/`Visible`), so the report can show exactly which columns a role is blind to.

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
    print(f"Starting security audit for {len(model_rows)} models...")

    for r in model_rows:
        workspace, dataset = r["Workspace_Name"], r["Semantic_Model_Name"]
        try:
            with connect_semantic_model(
                dataset=dataset, workspace=workspace, readonly=True
            ) as tom:

                for role in tom.model.Roles:
                    role_name = str(role.Name)

                    # Role -> OLS lookup: {table: {permission, columns{col: permission}}}
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

                    # ---- PART A: RLS filter expressions -------------------
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

                    # ---- PART B: OLS visibility per table/column ----------
                    for table in tom.model.Tables:
                        table_name = str(table.Name)
                        sec = role_ols_map.get(
                            table_name,
                            {"Table_Permission": "Read", "Columns": {}},
                        )

                        # Case 1: whole table hidden -> one collapsed row
                        if sec["Table_Permission"] == "None":
                            security_rows.append(Row(
                                Workspace=workspace, Semantic_Model=dataset,
                                Role_Name=role_name, Table_Name=table_name,
                                Column_Name="*", Filter=None, Filter_Type="OLS",
                                OLS_Level="Table", Column_Permission="None",
                                Effective_Visibility="Hidden", Error_Message=None,
                            ))
                            continue

                        # Case 2: table readable -> check each column
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

Sample of what lands in the table (values fabricated):

| Workspace | Semantic_Model | Role_Name | Table_Name | Column_Name | Filter | Filter_Type | Effective_Visibility |
|---|---|---|---|---|---|---|---|
| Contoso Finance Self-Service BI | Finance Semantic Model | Cost Center - Dynamic | Dim Cost Center | | `[Cost Center Code] IN {"CC-1001","CC-1044"}` | RLS | |
| Contoso Finance Self-Service BI | Finance Semantic Model | Finance - All | Dim Employee | Salary | | OLS | Hidden |
| Contoso Sales Self-Service BI | Sales Semantic Model | Region - West | Dim Region | | `[Region] = "WEST"` | RLS | |

Two design choices worth calling out:

- **Error rows instead of failures.** If the SP isn't in a workspace yet, we record a row with `Filter_Type = "ERROR"` and the message. The audit report surfaces these, so "we can't see this model" is itself an audit finding rather than a silent gap.
- **Effective visibility, not raw permissions.** OLS metadata only lists *exceptions*. By walking every table/column and defaulting to `Read`, the output row set answers "what does this role see?" without the consumer needing to know OLS semantics.

---

## Step 3 — Which AD groups sit in each role (and the query scale-out trap)

Role *definitions* are only half the story — we also need role *membership*: which Entra ID group (or user) is tagged to each role. TOM exposes this via `role.Members`, and each member's `MemberID` is the **Entra object ID** of the group or user. That object ID becomes the join key to Graph data in Step 4.

### The trap we fell into first

Our first version didn't use TOM at all — it ran a DAX query with the `INFO` functions, which works nicely in an interactive session:

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

…until it hit a model with **query scale-out** enabled. Scale-out routes read queries to read-only replicas, and on a replica the security-related INFO output isn't reliable — our runs failed or returned incomplete membership for exactly those models.

### The workaround

Before connecting, we check the dataset's `queryScaleOutSettings` via the Power BI REST API. If scale-out is on, we temporarily set `maxReadOnlyReplicas` to `0`, wait for the replicas to drain, extract memberships via TOM (which is authoritative on the primary), and then **restore the original setting**:

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

> 💡 **Callout — query scale-out vs. security extraction.** If any audited model uses query scale-out, budget for the drain wait (we use 5 minutes) and make absolutely sure the *restore* runs even on failure — wrap it in `try/finally` if you extend this. The PATCH requires the SP to have write access on the dataset (workspace Contributor or above).

---

## Step 4 — Expand AD groups into actual users via Microsoft Graph

Roles are (almost always) mapped to **Entra ID groups**, not individuals — so the last link in the chain is group → members. We follow a naming convention (`pbi-sec-*`) for all groups used in Power BI security, which makes the Graph query a single filtered call:

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

# 1. All security groups following the naming convention (paginated)
url = ("https://graph.microsoft.com/v1.0/groups"
       "?$filter=startswith(displayName,'pbi-sec')&$select=id,displayName")
groups = []
while url:
    data = requests.get(url, headers=headers).json()
    groups.extend(data.get("value", []))
    url = data.get("@odata.nextLink")

print(f"Found {len(groups)} groups")

# 2. Members of each group (users only; skip nested SPs/devices)
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

Note what just happened across Steps 3 and 4: the role member's `Member_ID` (from TOM) and the group's `id` (from Graph) are **the same Entra object ID**. That single fact is what lets the report join *role → group → human being* — it becomes the key relationship in the semantic model in Part 2.

> 💡 If you don't have a naming convention for BI security groups: this project is a very good excuse to introduce one. The alternative is enumerating every group in the tenant.
>
> Also note `/members` returns *direct* members only — if you nest groups inside groups, switch to `/transitiveMembers`.

---

## Step 5 — Workspace role assignments via the Fabric REST API

Model-level security means nothing if someone is an Admin on the workspace itself. The Fabric REST API (`GET /v1/workspaces/{id}/roleAssignments`) returns every principal (user, group, or SP) with a workspace role. We resolve workspace names to IDs, pull assignments, and — where the principal is a group — expand it to people using the Step 4 table:

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

# 1. Resolve workspace names -> IDs
all_ws = requests.get(f"{FABRIC}/v1/workspaces", headers=headers).json()["value"]
lookup = {w["displayName"]: w["id"] for w in all_ws}
workspaces = [{"id": lookup[n], "name": n} for n in WORKSPACE_NAMES if n in lookup]

# 2. Role assignments per workspace
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

# 3. Where the principal is a group, expand to individual users
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

Sample output (fabricated):

| Workspace | Name | Email | Type | Role |
|---|---|---|---|---|
| Contoso Finance Self-Service BI | pbi-sec-finance-developers | alex.chen@contoso.com | Group | Contributor |
| Contoso Finance Self-Service BI | pbi-sec-finance-developers | sam.rivera@contoso.com | Group | Contributor |
| Contoso Finance Self-Service BI | Jordan Lee | jordan.lee@contoso.com | User | Admin |
| Contoso Sales Self-Service BI | svc-pbi-audit | | ServicePrincipal | Contributor |

---

## Scheduling the pipeline

Both notebooks are idempotent (`overwrite` + `overwriteSchema`), so orchestration is simple:

1. A **Fabric data pipeline** with two sequential notebook activities (model security → workspace access), scheduled daily off-hours — the scale-out workaround can add ~5 minutes per scale-out model, so don't run it at 8:59 before a 9:00 review.
2. A **semantic model refresh** activity at the end, so the report (Part 2) always shows the latest audit with its refresh timestamp.

![Fabric data pipeline canvas — a schedule trigger runs the Semantic Model Security Details notebook, then the Workspace Access notebook on success, then a semantic model refresh on success.](assets/pipeline-canvas.png)

*The scheduled pipeline: two notebook activities chained on success, then a semantic model refresh so the report always carries a fresh timestamp.*

---

## Get the notebook — and check it's correct

All five steps above are compiled into one **self-contained** Fabric notebook:
**[`NB_Semantic_Model_Access_Audit.ipynb`](NB_Semantic_Model_Access_Audit.ipynb)** — no SharePoint, no config
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

Point a model at those CSVs instead — exactly what the downloadable **sample report** does (see [Part 2](Part%202%20-%20The%20Semantic%20Model%20and%20Audit%20Report.md#download-the-sample-report)): its Power Query reads the same snake_case CSVs, and switching that model to a live lakehouse is a one-line change.

### What a correct run looks like

I ran those checks against the fabricated sample dataset that ships with the template — the same schemas the notebook writes. Real output:

```
pbi_model_details                                     5 rows   ok
pbi_model_security_audit_details                     65 rows   ok
pbi_model_security_role_association_details          10 rows   ok
pbi_model_security_adgroup_user_details              31 rows   ok
pbi_workspace_access_audit_details                   16 rows   ok
```

**Zero orphans** on the referential-integrity check — every `Member_ID` resolved to a `Group_Object_ID`, which is the strongest single signal that Steps 3 and 4 agree with each other. The `Filter_Type = "ERROR"` row for *Legacy Inventory Semantic Model* appeared as data, exactly as intended.

Asking "who can see `Finance Semantic Model`?" returned 10 rows — four roles, resolved through their groups to actual people:

```
Role_Name             | Group_Name                         | User_Name      | User_Principal_Name
Cost Center - Dynamic | pbi-sec-finance-costcenter-dynamic | Alex Chen      | alex.chen@contoso.com
Cost Center - West    | pbi-sec-finance-costcenter-west    | Dana Whitfield | dana.whitfield@contoso.com
Entity - EMEA         | pbi-sec-finance-entity-emea        | Elena Petrova  | elena.petrova@contoso.com
Finance - All         | pbi-sec-finance-all                | Jordan Lee     | jordan.lee@contoso.com
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

## What we can answer so far

With five Delta tables in place, four of the five audit questions are now answerable with plain SQL — try it in the lakehouse SQL endpoint:

```sql
-- "Who can see the Finance Semantic Model, through which role and group?"
SELECT DISTINCT
    r.Workspace, r.Semantic_Model, r.Role_Name,
    u.Group_Name, u.User_Name, u.User_Principal_Name
FROM pbi_security_audit.dbo.pbi_model_security_role_association_details r
JOIN pbi_security_audit.dbo.pbi_model_security_adgroup_user_details u
  ON r.Member_ID = u.Group_Object_ID
WHERE r.Semantic_Model = 'Finance Semantic Model';
```

But nobody from compliance wants to run SQL. In [**Part 2**](Part%202%20-%20The%20Semantic%20Model%20and%20Audit%20Report.md), we put a semantic model and a six-page Power BI report on top: RLS and OLS explorer pages, role → group → user drill-through, workspace access review, and a persona matrix that translates DAX filters into business language.

---

## Questions & suggestions

Questions, corrections, or a cleaner way around the scale-out drain? I'd genuinely like to hear it — leave a comment below, or reach out to the authors directly. Issues and pull requests on the [companion repo](https://github.com/NatarajanManivasagan/Fabric-Audit-Log) are welcome too.

## About the authors

| Role | Name | Connect |
|---|---|---|
| **Author** | Natarajan Manivasagan | [LinkedIn](https://www.linkedin.com/in/natarajan-manivasagan/) · [Fabric Community](https://community.fabric.microsoft.com/users/natarajan_m/1345926) |
| **Co-author** | Praful Potphode | [LinkedIn](https://www.linkedin.com/in/praful-p-912349241/) · [Fabric Community](https://community.fabric.microsoft.com/users/praful_potphode/1261729) |
| **Co-author** | Hardik Sri | [LinkedIn](https://www.linkedin.com/in/hardiksri98/) · [Fabric Community](https://community.fabric.microsoft.com/users/hardiksri/1586944) |

## Further reading

- [semantic-link-labs — documentation & source](https://github.com/microsoft/semantic-link-labs)
- [Row-level security (RLS) in Power BI / Fabric](https://learn.microsoft.com/fabric/security/service-admin-row-level-security)
- [Object-level security (OLS)](https://learn.microsoft.com/fabric/security/service-admin-object-level-security)
- [Roles in workspaces — why a workspace Admin bypasses RLS](https://learn.microsoft.com/fabric/fundamentals/roles-workspaces)

---

> **This series** · **Part 1: Building the Audit Pipeline** *(you are here)* · [Part 2: The Semantic Model and Audit Report →](Part%202%20-%20The%20Semantic%20Model%20and%20Audit%20Report.md)

*The views in this post are my own and don't necessarily represent my employer. All workspace names, model names, group names, and IDs are fictionalized — the patterns are real.*
