# Power BI Semantic Model Access Audit — in Microsoft Fabric

Companion code and write-up for the blog **"Who Can See What? Auditing Power BI Semantic Model Security in Microsoft Fabric."**

![Visual summary — the "who can see what?" question flows through a SharePoint inventory, an Entra ID service principal, Fabric notebooks, a lakehouse of five Delta tables, and a six-page audit report; with the RLS, OLS, Entra ID group and workspace-role motifs and the "a workspace Admin bypasses RLS" gap the audit catches.](assets/blog-summary-visual.png)

A self-service BI platform eventually gets the audit question: *"who can see what in Power BI?"* This solution answers it
as a **data product** — Fabric notebooks harvest RLS/OLS definitions, role memberships, Entra ID (AD) group members and
workspace roles into a lakehouse, and a Power BI report answers five questions on demand:

1. Which users have access to each semantic model?
2. Which security role is each user tagged to?
3. Which Entra ID (AD) group grants that access?
4. What workspace role do they hold?
5. What DAX filters (RLS) — and table/column restrictions (OLS) — does each role apply?

## What's here

| | |
|---|---|
| 📓 **[NB_Semantic_Model_Access_Audit.ipynb](NB_Semantic_Model_Access_Audit.ipynb)** | The Fabric notebook. Paste service-principal credentials, list the `(workspace, model)` pairs to audit, and run top to bottom. It's **DataFrame-first** — writes nothing until you choose to persist to Delta tables or export CSVs. |
| 📄 **[Read the deep-dive](Single%20Part%20-%20Power%20BI%20Access%20Audit%20in%20Fabric%20%28Deep%20Dive%29.md)** | The whole story in one post. Also as a two-part series: **[Part 1 — the pipeline](Part%201%20-%20Building%20the%20Power%20BI%20Access%20Audit%20Pipeline%20in%20Fabric.md)** · **[Part 2 — the model & report](Part%202%20-%20The%20Semantic%20Model%20and%20Audit%20Report.md)**. |
| 📊 **[contoso-access-audit-pbip.zip](contoso-access-audit-pbip.zip)** | The sample Power BI report + model (PBIP). Unzip to `C:\ContosoAccessAudit`, open the `.pbip` in Power BI Desktop, hit **Refresh** — it opens on fabricated data, with no tenant or credentials required. |
| 🗂️ **[SCHEMA.md](SCHEMA.md)** | The six lakehouse tables — every column, type, and grain — so you can see what the notebook produces without reading the code. |
| ✅ **[TESTING.md](TESTING.md)** | How the code is verified: offline checks against the sample data, plus a live smoke-test checklist and the known gotchas. |

## Quick start (the notebook)

1. Import `NB_Semantic_Model_Access_Audit.ipynb` into a Fabric workspace (*Workspace → Import → Notebook*).
2. Run the first cell — `%pip install -U semantic-link semantic-link-labs` — then **restart the kernel**
   (service-principal auth needs `semantic-link` ≥ 0.12.0).
3. Fill in the **CONFIG** cell: your `TENANT_ID` / `CLIENT_ID` / `CLIENT_SECRET`, and a `MODELS` list of the
   `(workspace, semantic model)` pairs to audit.
4. Run top to bottom. Each step builds a DataFrame you can inspect; the validation section confirms the output is
   correct and answers the five questions for one model.

## Requirements

- Microsoft Fabric with a lakehouse attached; Power BI Desktop for the sample report.
- An Entra **app registration (service principal)** with Microsoft Graph **`Group.Read.All`**, added to every audited
  workspace, plus the tenant setting *Service principals can use Fabric APIs* and an XMLA endpoint set to at least *Read*.

## License

Released under the [MIT License](LICENSE) — the code and sample are free to use, adapt, and build on.

## Note

Every workspace name, model, role, group, user and ID in this repository is **fabricated ("Contoso")** — it is a
genericized reference implementation, not an export of any real environment.
