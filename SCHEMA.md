# Lakehouse table schema

The audit lands **six tables** in the `pbi_security_audit` lakehouse (schema `dbo`). Five are produced by
[`NB_Semantic_Model_Access_Audit.ipynb`](NB_Semantic_Model_Access_Audit.ipynb) — one per pipeline step; the sixth
(`pbi_role_personas`) is a small **hand-curated** business-layer table.

Every column is stored as **string** (`StringType`) — the audit is metadata, so values are kept verbatim (DAX filter
text, object IDs, principal names) with no type coercion. The notebook is **DataFrame-first**: each step registers a
temp view of the same name and writes nothing until you enable `publish()`'s Delta or CSV option, so these are the
shapes of both the DataFrames and the tables/CSVs they become.

The downloadable sample report (`contoso-access-audit-pbip.zip`) ships CSVs with these exact column names, so pointing
the model at a real lakehouse is a one-line Power Query swap.

---

## `pbi_model_details`

**Grain:** one row per workspace × semantic model. **Purpose:** the audit scope — the driver table Steps 2–5 read.
**Built by:** Step 1 (from the SharePoint inventory in production, or the inline `MODELS` list in the notebook).

| Column | Type | Notes |
|---|---|---|
| `Workspace_Name` | string | Fabric/Power BI workspace display name. |
| `Semantic_Model_Name` | string | Semantic model (dataset) display name within that workspace. |

---

## `pbi_model_security_audit_details`

**Grain:** model × role × table (× column). **Purpose:** the RLS and OLS definitions — answers **Q5**.
**Built by:** Step 2 (read-only TOM walk of each role).

| Column | Type | Notes |
|---|---|---|
| `Workspace` | string | Workspace of the audited model. |
| `Semantic_Model` | string | Audited semantic model. |
| `Role_Name` | string | Security role name; `null` on an `ERROR` row. |
| `Table_Name` | string | Table the rule applies to. |
| `Column_Name` | string | OLS column; `*` for a whole-table blackout; `null` for an RLS row. |
| `Filter` | string | The RLS **DAX filter expression** (RLS rows only). |
| `Filter_Type` | string | `RLS`, `OLS`, or `ERROR` (a model the service principal could not read). |
| `OLS_Level` | string | `Table`, `Column`, or `None` (column readable). |
| `Column_Permission` | string | Raw OLS metadata permission (`None` = hidden, else `Read`). |
| `Effective_Visibility` | string | Resolved `Hidden` / `Visible` — "what does this role actually see?" |
| `Error_Message` | string | The exception text on an `ERROR` row; otherwise `null`. |

> `RowNumber` internal columns are skipped. A table with `Table_Permission = "None"` collapses to a single
> `Column_Name = "*"` row rather than one row per hidden column.

---

## `pbi_model_security_role_association_details`

**Grain:** model × role × member. **Purpose:** which principal (Entra group or user) is tagged to each role —
answers **Q2/Q3**. **Built by:** Step 3 (TOM `role.Members`, with the query-scale-out drain workaround).

| Column | Type | Notes |
|---|---|---|
| `Workspace` | string | Workspace of the model. |
| `Semantic_Model` | string | Model the role belongs to. |
| `Role_Name` | string | Security role name. |
| `Member_ID` | string | **Entra object ID** of the member — equals `Group_Object_ID` below (the join key). |

---

## `pbi_model_security_adgroup_user_details`

**Grain:** group × user. **Purpose:** expands each `pbi-sec-*` security group to its member users —
answers **Q1/Q3**. **Built by:** Step 4 (Microsoft Graph).

| Column | Type | Notes |
|---|---|---|
| `Group_Name` | string | Security group display name (`pbi-sec-*` convention). |
| `Group_Object_ID` | string | **Entra object ID** of the group — joins to `Member_ID` above. |
| `User_Name` | string | Member user's display name. |
| `User_Object_ID` | string | Member user's Entra object ID. |
| `User_Principal_Name` | string | Member user's UPN (email) — the join to workspace access. |

> `/members` returns direct members only; swap in `/transitiveMembers` for nested groups.

---

## `pbi_workspace_access_audit_details`

**Grain:** workspace × principal. **Purpose:** workspace role assignments — answers **Q4**, and surfaces the classic
gap (an RLS-restricted user who is *also* workspace Admin, so RLS never applies). **Built by:** Step 5 (Fabric REST
`roleAssignments`, with group principals expanded to users via the Step 4 table).

| Column | Type | Notes |
|---|---|---|
| `Workspace` | string | Workspace display name. |
| `Name` | string | Principal display name (user, group, or service principal). |
| `Email` | string | UPN/email; for a group principal, filled from its expanded members. |
| `Type` | string | `User`, `Group`, or `ServicePrincipal`. |
| `Role` | string | Workspace role: `Admin`, `Member`, `Contributor`, or `Viewer`. |

---

## `pbi_role_personas`

**Grain:** one row per persona. **Purpose:** the business-readable layer — describes each role's filters in plain
columns so reviewers can catch "the DAX says X but the persona sheet says Y" drift. **Built by:** hand-curated (not
generated by the notebook); joined to the AD-group/user table on the group name.

| Column | Type | Notes |
|---|---|---|
| `AD_Group` | string | The `pbi-sec-*` group this persona corresponds to (join key to `Group_Name`). |
| `Persona` | string | Plain-language persona label. |
| `Business_Unit` | string | Business unit the persona belongs to. |
| `Cost_Centers_Include` / `Cost_Centers_Exclude` | string | Cost-center scope in business terms (mirrors the RLS filter). |
| `Accounts` / `Entities` | string | Account and entity scope. |
| `Source_Systems` | string | Which source-system slices the persona may see. |
| `Active` | string | `1` for active personas (the model filters to `Active = "1"`). |

> This is the one table you populate by hand — treat the generated RLS DAX as the source of truth and keep the
> persona description in sync with it. Column names shown are representative; adapt them to your own attributes.
