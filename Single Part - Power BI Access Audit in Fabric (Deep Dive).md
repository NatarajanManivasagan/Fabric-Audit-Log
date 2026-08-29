# Who Can See What? A Complete Power BI Access Audit Solution in Microsoft Fabric

![Cover — "Who can see what? Auditing Power BI semantic model security in Microsoft Fabric." Fabric notebooks harvest RLS/OLS, role memberships, AD groups and workspace roles into a lakehouse; a Power BI report answers every access question on demand.](assets/cover-image.png)

*A deep dive: Fabric notebooks harvest RLS/OLS definitions, role memberships, AD group members, and workspace roles into a lakehouse — and a Power BI report on top answers every "who can see what?" question on demand.*

`Microsoft Fabric` · `Power BI` · `RLS / OLS` · `Security & Governance`

**Level:** Intermediate · **Read:** ~14 min · **Requires:** Microsoft Fabric + an Entra service principal · **Published:** Tuesday, September 1, 2026

| Authors | | |
|---|---|---|
| **Natarajan Manivasagan** · *Author* | [LinkedIn](https://www.linkedin.com/in/natarajan-manivasagan/) | [Fabric Community](https://community.fabric.microsoft.com/users/natarajan_m/1345926) |
| **Praful Potphode** · *Co-author* | [LinkedIn](https://www.linkedin.com/in/praful-p-912349241/) | [Fabric Community](https://community.fabric.microsoft.com/users/praful_potphode/1261729) |
| **Hardik Sri** · *Co-author* | [LinkedIn](https://www.linkedin.com/in/hardiksri98/) | [Fabric Community](https://community.fabric.microsoft.com/users/hardiksri/1586944) |

> 📦 **Grab the sample project.** The finished report and semantic model download as a **PBIP template**
> (`contoso-access-audit-pbip.zip`) wired to fabricated CSV data — it opens in Power BI Desktop with no tenant,
> no service principal and no credentials ([details below](#download-the-sample-report)). The whole pipeline is
> also one runnable notebook, **[`NB_Semantic_Model_Access_Audit.ipynb`](NB_Semantic_Model_Access_Audit.ipynb)**,
> with a built-in validation section.

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

For one model you can answer this by hand. Across a fleet of models in multiple workspaces — secured with a mix of **RLS**, **OLS**, and **Entra ID groups** — the manual answer is stale before you finish collecting it.

So we built the audit *as a data product*: Fabric notebooks harvest security metadata from every model into a lakehouse, and a Power BI report answers all five questions on demand, with a refresh timestamp. Everything below is genericized (Contoso-ified) so you can map it to your own tenant.

---

## Solution architecture

![Architecture diagram — a SharePoint inventory workbook and an Entra ID service principal feed a set of Fabric notebooks, which write five Delta tables into the pbi_security_audit lakehouse; a semantic model and a six-page audit report sit on top.](assets/architecture-diagram.png)

| # | Component | Role |
|---|---|---|
| 1 | **SharePoint `Model Inventory.xlsx`** | Scope control — one row per model to audit. |
| 2 | **Entra ID service principal** | The identity that does all the reading: XMLA/TOM, REST, Graph. |
| 3 | **Fabric notebooks** | The harvesters: `semantic-link(-labs)`, Power BI & Fabric REST, Graph. |
| 4 | **Lakehouse `pbi_security_audit`** | The audit store — one Delta table per question. |
| 5 | **Semantic model + 6-page report** | The consumption layer. |

The pipeline is one-way: the inventory drives the notebooks, each step lands one Delta table (Notebook 1 does inventory, RLS/OLS, role memberships and AD-group expansion; Notebook 2 does the workspace-role scan), and the semantic model and report sit on top.

> 🧭 **How the code is organized.** The steps below show the *essence* of each stage. The complete, runnable code lives in one notebook, **[`NB_Semantic_Model_Access_Audit.ipynb`](NB_Semantic_Model_Access_Audit.ipynb)** — a self-contained, DataFrame-first variant; the snippets show the **production** shape.

---

## Prerequisites

This is the part that takes longer than the code.

1. **App registration.** In Entra ID, note Tenant ID / Client ID and create a client secret. Application permissions (admin-consented): Graph `Group.Read.All` and `Sites.Read.All`.
2. **Tenant settings** (Fabric admin portal): *Service principals can use Fabric APIs*; **XMLA endpoint** ≥ *Read*; and — for the app-audience extension — *Service principals can access read-only admin APIs*.
3. **Workspace access.** Add the SP to every audited workspace (Contributor); missing ones are flagged instead of failing the run.
4. **Lakehouse.** Create `pbi_security_audit` **with schema support** as the notebooks' default lakehouse.
5. **Library.** `%pip install semantic-link-labs`.

> ⚠️ **Secret hygiene.** The snippets read credentials from a `config.json` to keep the walkthrough simple. In production, use **Azure Key Vault** with `notebookutils.credentials.getSecret(...)`. This is a *security audit* solution — don't let it become the finding.

**The lakehouse tables** ([full columns in `SCHEMA.md`](SCHEMA.md)):

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

Scope lives in one SharePoint workbook — one row per model. Authenticate as the service principal, pull the workbook through Graph, and land it as the scope table:

```python
# MSAL client-credentials token (Graph scope), then download the inventory workbook
headers = graph_headers(config)                 # Authorization: Bearer …
site = graph_get(".../sites/contoso.sharepoint.com:/sites/BIGovernance")
xlsx = graph_get(f".../sites/{site['id']}/drive/root:/PBI Governance/Model Inventory.xlsx:/content")

df = pd.read_excel(xlsx, sheet_name="Self Service Models", dtype=str)
#  ->  Workspace_Name | Semantic_Model_Name      (one row per model to audit)
```

> 📓 **Full code:** [Step 1 in the notebook →](NB_Semantic_Model_Access_Audit.ipynb) *(the self-contained version skips SharePoint — you list the `(workspace, model)` pairs inline in a `MODELS` variable).*

---

## Step 2 — Extract RLS and OLS with semantic-link-labs (TOM)

The heart of the solution. For every model in the inventory we open a **read-only TOM connection** under the service principal and walk `Roles`:

- **RLS:** every `TablePermission.FilterExpression` is the role's DAX filter on that table.
- **OLS:** `MetadataPermission` on tables/columns. A whole table hidden → one collapsed row (`Column_Name = "*"`); otherwise one row per column with effective visibility (`Hidden`/`Visible`).

```python
set_service_principal(config["TENANT_ID"], config["CLIENT_ID"], config["CLIENT_SECRET"])

with connect_semantic_model(dataset, workspace, readonly=True) as tom:
    for role in tom.model.Roles:
        # RLS — the DAX filter this role applies to a table
        for perm in role.TablePermissions:
            if perm.FilterExpression:
                emit(role.Name, perm.Table.Name, filter=perm.FilterExpression, kind="RLS")

        # OLS — walk every table/column, default to Read, record what's hidden
        for table in tom.model.Tables:
            for column in table.Columns:
                emit(role.Name, table.Name, column.Name,
                     visibility=effective_visibility(role, table, column), kind="OLS")
```

Each RLS row carries the DAX filter (e.g. `[Cost Center Code] IN {"CC-1001","CC-1044"}`); each OLS row carries `Effective_Visibility` (e.g. `Dim Employee[Salary]` → `Hidden` for the `Finance - All` role).

Two design choices worth calling out:

- **Errors as data.** If the SP can't read a model, we write a `Filter_Type = "ERROR"` row — "we can't see this" is itself a finding, not a silent gap.
- **Effective visibility, not raw permissions.** By walking every table/column and defaulting to `Read`, the output answers "what does this role *see*?" without the reader knowing OLS semantics.

> 📓 **Full code:** [Step 2 in the notebook →](NB_Semantic_Model_Access_Audit.ipynb) — the `emit` / `effective_visibility` logic (table-level blackouts, `RowNumber` columns skipped) and the eleven-column output schema.

---

## Step 3 — Role memberships (and the query scale-out trap)

Role definitions are half the story; we also need **who is tagged to each role**. TOM exposes `role.Members`, and each member's `MemberID` is the **Entra object ID** of the group or user — the join key for the next step.

**The trap:** our first version used DAX `INFO.ROLES()` instead of TOM. Fine interactively — until a model with **query scale-out** enabled routed reads to replicas where security metadata isn't reliably served. **The fix:** detect `queryScaleOutSettings`; if scale-out is on, set `maxReadOnlyReplicas = 0`, let the replicas drain, extract via TOM on the primary, then **restore the setting**:

```python
for workspace, dataset in models:
    settings = get_scaleout_settings(ws_id, ds_id)          # Power BI REST
    if settings.get("maxReadOnlyReplicas", 0):              # scale-out on?
        update_scaleout(ws_id, ds_id, {"maxReadOnlyReplicas": 0})
        time.sleep(300)                                     # let read replicas drain

    with connect_semantic_model(workspace, dataset, readonly=True) as tom:
        for role in tom.model.Roles:
            for member in role.Members:
                emit(role.Name, member.MemberID)            # Entra object ID = join key

    update_scaleout(ws_id, ds_id, settings)                 # ALWAYS restore
```

> 💡 Budget for the drain wait per scale-out model, and make sure the *restore* runs even on failure (`try/finally`). The PATCH needs write access on the dataset — workspace Contributor or above.

> 📓 **Full code:** [Step 3 in the notebook →](NB_Semantic_Model_Access_Audit.ipynb) — the REST helpers and per-model error handling.

---

## Step 4 — Expand AD groups into users via Microsoft Graph

Roles map to **Entra ID groups**, not individuals. A naming convention (`pbi-sec-*`) makes the lookup one filtered Graph query (`/groups?$filter=startswith(displayName,'pbi-sec')`); a second pass over each group's `/members` pulls the users, emitting `Group_Name`, `Group_Object_ID`, `User_Name`, `User_Object_ID` and `User_Principal_Name`.

The linchpin: the role member's `MemberID` (TOM, Step 3) and the group's `id` (Graph, here) are **the same Entra object ID** — the join that connects *role → group → human being*.

> 📓 **Full code:** [Step 4 in the notebook →](NB_Semantic_Model_Access_Audit.ipynb) — `graph_get_all` is the paged helper; `/members` returns *direct* members, so swap in `/transitiveMembers` for nested groups.

---

## Step 5 — Workspace role assignments via the Fabric REST API

Model-level security means nothing if someone is workspace Admin. `GET /v1/workspaces/{id}/roleAssignments` returns every principal with a role; where the principal is a group, we expand it to people using the Step 4 table:

```python
for ws in workspaces:
    for a in fabric_get(f"/v1/workspaces/{ws['id']}/roleAssignments"):
        p = a["principal"]
        emit(ws["name"], p["displayName"],
             p.get("userDetails", {}).get("userPrincipalName"), p["type"], a["role"])
# Group principals are expanded to members via the Step 4 table — Email included.
```

Each row is a workspace principal (user, group, or service principal) with its role — e.g. *Jordan Lee → Admin* on *Contoso Finance Self-Service BI*, the row that matters below.

**Scheduling:** both notebooks are idempotent, so a Fabric pipeline runs them sequentially off-hours, then refreshes the model. The scale-out workaround adds ~5 minutes per affected model.

![Fabric data pipeline canvas — a schedule trigger runs the Semantic Model Security Details notebook, then the Workspace Access notebook on success, then a semantic model refresh on success.](assets/pipeline-canvas.png)

> 📓 **Full code:** [Step 5 in the notebook →](NB_Semantic_Model_Access_Audit.ipynb) — the workspace name→id lookup, `pd.json_normalize` flattening, and the group-expansion merge.

---

## Get the notebook — and check it's correct

All five steps are compiled into one **self-contained** Fabric notebook: **[`NB_Semantic_Model_Access_Audit.ipynb`](NB_Semantic_Model_Access_Audit.ipynb)** — no SharePoint, no config file, no external module. Import it, fill in the **CONFIG** cell (`TENANT_ID` / `CLIENT_ID` / `CLIENT_SECRET` and a `MODELS` list of `(workspace, model)` pairs), run the install cell, **restart the kernel** (auth needs `semantic-link` ≥ 0.12.0), then run top to bottom. It's **DataFrame-first** — nothing is written until you flip `persist` on.

**How do you know it's right?** The notebook ends with a **validation section**: it confirms all five result sets are non-empty, runs a **referential-integrity** check (`Member_ID = Group_Object_ID`), surfaces the `ERROR` rows, and answers all five questions in SQL for one model.

**Landing the data.** The `publish()` helper carries two commented-out options: persist each result as a **Delta table** (`saveAsTable`) for a production import/Direct Lake model, or export each to a **CSV** (`to_csv`) — what the [downloadable sample](#download-the-sample-report) reads.

**What a correct run looks like.** Against the sample dataset, all five tables built (5 / 65 / 10 / 31 / 16 rows) with **zero orphans** on the referential-integrity check — the strongest single signal that Steps 3 and 4 agree — and the `ERROR` row for *Legacy Inventory Semantic Model* appeared as data. Then the check that earns its keep, joining role membership to workspace roles:

```
User_Name  | Role_Name     | Workspace                       | Workspace_Role
Jordan Lee | Finance - All | Contoso Finance Self-Service BI | Admin
```

Jordan Lee sits in an RLS role *and* holds workspace **Admin** — so that carefully-scoped role is doing nothing for them, because workspace Admins bypass RLS entirely. That single row is the whole reason this audit exists.

---

## The semantic model

The audit model is a small **import-mode** model over the lakehouse's **SQL analytics endpoint**. Every table follows the same Power Query pattern — connect, pick the table, rename to friendly names, and (on the two role-grain tables) add a composite `Key = Workspace | Model | Role`. Six tables — **Audit Details**, **Role AD Group Association**, **AD Group User Association**, **Workspace Access Details**, **Dynamic Role Persona Details** (a persona → filter-attribute matrix filtered to `Active = "1"`), and a generated **Last Updated Time** — joined by three relationships:

| From table | Joined on | To table |
|---|---|---|
| Audit Details | `Key` — the `Workspace \| Model \| Role` composite key | Role AD Group Association |
| Role AD Group Association | `Group Object ID` — **★ the linchpin** | AD Group User Association |
| Dynamic Role Persona Details | `AD_Group` (AD group name) | AD Group User Association |

![Data model diagram — Audit Details joins to Role AD Group Association on a composite Workspace-Model-Role key; Role AD Group Association joins to AD Group User Association on Group Object ID; Dynamic Role Persona Details joins to AD Group User Association on AD group name. Workspace Access Details and Last Updated Time stand alone.](assets/data-model-diagram.png)

Design notes:

- **The composite `Key`.** A relationship on `Role Name` alone breaks the moment two models both define a "Viewer" role; the composite key gives one honest relationship.
- **The Group Object ID linchpin.** TOM's `MemberID` = Graph's group `id` = the same Entra object ID. No name matching, no fuzzy joins.
- **`Last Updated Time`.** A one-row M table stamping the refresh time on every page — preempting the auditor's first question, *"as of when?"*
- **The persona matrix.** `pbi_role_personas` describes each persona in plain columns (business unit, cost centers, accounts, entities, source systems). Shown next to the raw DAX, it lets reviewers catch "the filter says X but the persona sheet says Y" drift.

> 📓 The Power Query M for every table (the renames, the `Key`, the `Last Updated Time` table) ships in the [sample PBIP](#download-the-sample-report).

---

## The report

Six pages, one layout language (title, last-updated stamp, slicer rail).

> 📷 Every page below is on the **fabricated sample data** that ships with the download — every workspace, model, role, group, filter, and person is invented.

1. **Role – Persona Details** *(landing)* — Role → AD Group → User → UPN with slicers. Answers **Q1/Q2/Q3**, showing *people*, not just group names.
2. **Security Details (RLS)** — Role → Table → Filter (the DAX), page-filtered to RLS; a bookmark-driven **RLS/OLS toggle** flips to the next page.
3. **OLS** *(via toggle)* — Role → Table → Column → Permission, pre-filtered to what's hidden. With page 2: **Q5**.
4. **View Security Details** *(drill-through)* — right-click any role → its RLS filter matrix next to the persona description.
5. **AD Group Details** — Group → User → UPN, for questions that arrive group-first. **Q3.**
6. **Workspace Access Details** — Workspace → principal → email → role. **Q4** — the page that finds the classic gap: a modest RLS role held by someone who is *also* workspace Admin.

*Role – Persona Details (landing)* — who has access, via which role, group, and persona attributes:

![Role & Persona page — a table listing each semantic model, role, AD group, business unit, region, and which source systems (ERP/Expense/HCM) the persona may see.](assets/screenshots/report-role-persona.png)

*Security Details (RLS)* — each role's DAX filter, per table:

![Security Details page — each role's RLS DAX filter shown per table, including a dynamic USERPRINCIPALNAME() rule, IN-list filters, and simple equality filters.](assets/screenshots/report-security-rls.png)

*Object-level security* — what each role can't even see:

![OLS page — hidden objects per role, mixing table-level blackouts with column-level ones such as Salary, Bonus, and Margin Amount, each marked Hidden.](assets/screenshots/report-ols.png)

*Workspace Access Details* — workspace roles, with the RLS-vs-Admin gap in plain sight:

![Workspace Access Details page — each workspace's principals (groups, users, service principals) with their email, principal type, and colour-coded workspace role (Admin, Member, Contributor, Viewer).](assets/screenshots/report-workspace-access.png)

---

## Extensions

Three further access paths, each one cell away from the same pattern (all included in the notebook's *Optional extensions* cell): **app audiences** via `admin/apps` (needs the *read-only admin APIs* tenant setting — a plain workspace SP gets 403); **direct dataset permissions**, catching users in `datasets/{id}/users` but not `groups/{id}/users` — the one-off grants audits exist to find; and **lineage**, joining `fabric.list_reports()` to `fabric.list_datasets()` to answer "which reports does this model feed?".

---

## Download the sample report

Rather than describe the report and leave you to rebuild it, here it is: **`contoso-access-audit-pbip.zip`** — the six-page report and semantic model as a **PBIP project**, with **fabricated sample data** (3 workspaces, 4 models, 10 roles, 12 AD groups, 30 users, RLS filters and both flavours of OLS) loaded from CSVs. It opens in Power BI Desktop with **no tenant, gateway, service principal or credentials**: unzip to `C:\ContosoAccessAudit`, open the `.pbip`, and hit **Refresh** (unzipped elsewhere? point the **`CsvFolder`** parameter at your `Data` folder).

![The template open in Power BI Desktop, refreshed on the sample CSV data — the AD Group page rendering with the model's tables listed in the Data pane.](assets/screenshots/template-in-desktop.png)

The CSV headers are the same snake_case names the notebooks write, so the template's Power Query is the *same* code the real solution runs — pointing it at your own lakehouse is a one-line swap (`Csv.Document(...)` → `Sql.Databases(...)`), and everything downstream stays as-is. One deliberate detail: a `Filter_Type = ERROR` row for *Legacy Inventory Semantic Model*, showing how the pipeline records "couldn't read this model" as data instead of failing.

---

## Lessons learned

1. **Query scale-out silently breaks security extraction.** Read replicas don't reliably serve role/membership metadata. Detect, drain, extract on the primary, restore — our biggest time sink.
2. **Record errors as data.** "SP not in workspace" becomes an `ERROR` row on the report — a visible finding, not a silent hole.
3. **Translate DAX for your auditors.** The persona matrix turned the report from "a developer tool" into "the thing compliance signs off on".

---

## Wrapping up

What started as an awkward compliance question — *"who can see what?"* — became a small data product: two notebooks, five Delta tables, one semantic model, six report pages. The audit that used to take days of screenshotting role dialogs is now a scheduled refresh, and every answer carries its own timestamp.

## Questions & suggestions

Questions, corrections, or a cleaner way around the scale-out drain? I'd genuinely like to hear it — leave a comment below, or reach out to the authors above. Issues and pull requests on the [companion repo](https://github.com/NatarajanManivasagan/Fabric-Audit-Log) are welcome too.

## Further reading

- [semantic-link-labs — documentation & source](https://github.com/microsoft/semantic-link-labs)
- [Row-level security (RLS) in Power BI / Fabric](https://learn.microsoft.com/fabric/security/service-admin-row-level-security)
- [Object-level security (OLS)](https://learn.microsoft.com/fabric/security/service-admin-object-level-security)
- [Roles in workspaces — why a workspace Admin bypasses RLS](https://learn.microsoft.com/fabric/fundamentals/roles-workspaces)

---

*The views in this post are my own and don't necessarily represent my employer. All workspace names, model names, group names, and IDs are fictionalized — the patterns are real.*
