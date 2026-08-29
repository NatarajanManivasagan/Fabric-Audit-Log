# Testing the audit

The harvesting code (`sempy` TOM, Microsoft Graph, Power BI / Fabric REST) only runs against a **live tenant**, so
"correct" is proven in two layers: what can be checked offline, and a live smoke test in Fabric.

## 1. Offline checks (no tenant required)

These run on any machine with Python — they prove the notebook is syntactically sound and that the audit *logic*
(the joins and the five questions) is right, using the fabricated CSVs in `contoso-access-audit-pbip.zip`.

**Notebook compiles.** Every code cell parses as valid Python (Jupyter magics stripped):

```
code cells: 16
RESULT: ALL CELLS COMPILE
```

**Validation logic on the sample data.** Loading the sample CSVs and running the notebook's Section 7–8 checks
reproduces the expected result exactly:

```
pbi_model_details                                5 rows   ok
pbi_model_security_audit_details                65 rows   ok
pbi_model_security_role_association_details     10 rows   ok
pbi_model_security_adgroup_user_details         31 rows   ok
pbi_workspace_access_audit_details              16 rows   ok

referential integrity (Member_ID -> Group_Object_ID): 0 orphans   ✔
ERROR row recorded as data: "Legacy Inventory Semantic Model"     ✔
governance gap reproduced: Jordan Lee holds Finance-All RLS AND workspace Admin ✔
```

The zero-orphan referential-integrity result is the strongest single signal that the role-membership step and the
Graph group-expansion step agree with each other. The `ERROR` row proves coverage gaps surface as **data**, not
silent failures.

## 2. Live smoke test in Fabric

1. **Import** `NB_Semantic_Model_Access_Audit.ipynb` into a workspace with a lakehouse attached.
2. Run the first cell — `%pip install -U semantic-link semantic-link-labs` — then **restart the kernel**.
3. Fill the **CONFIG** cell: `TENANT_ID` / `CLIENT_ID` / `CLIENT_SECRET`, and **one** `(workspace, model)` pair in
   `MODELS` (start with a single model you know has roles defined).
4. **Run top to bottom.** Expected: each step's `display()` shows a non-empty DataFrame; the Section 7 checks print
   row counts and `0` (or explained) orphans; Section 8 answers all five questions for your model.
5. Widen `MODELS` to the full fleet once one model looks right.

### Expected per step

| Step | Expected |
|---|---|
| 1 · scope | one row per `(workspace, model)` you listed |
| 2 · RLS/OLS | RLS rows carry a DAX `Filter`; OLS rows carry `Effective_Visibility`; unreadable models appear as `Filter_Type = "ERROR"` |
| 3 · members | one row per role member; `Member_ID` is an Entra object ID |
| 4 · groups | `pbi-sec-*` groups expanded to users; `Group_Object_ID` matches Step 3's `Member_ID` |
| 5 · workspace | every workspace principal with a role; groups expanded to people |

## Known gotchas

| Symptom | Cause | Fix |
|---|---|---|
| `cannot import name 'set_service_principal' from 'sempy.fabric'` | `semantic-link` older than **0.12.0** on the runtime | run the install cell, then **restart the kernel** |
| Graph group expansion returns nothing / 403 | app registration missing **`Group.Read.All`** (application permission, admin-consented) | grant + consent it in Entra ID |
| TOM connection fails for a model | service principal not added to that workspace, or **XMLA endpoint** not set to *Read* | add the SP (Contributor); set XMLA to Read on the capacity |
| Membership incomplete on one model | **query scale-out** serving from a read replica | Step 3 detects it, drains replicas, extracts on the primary, restores — budget ~5 min per affected model |
| `admin/*` routes 403 (extensions) | `admin/*` needs a different tier than `groups/*` | enable *Service principals can access read-only admin APIs* |

*All names and IDs in the sample data are fabricated.*
