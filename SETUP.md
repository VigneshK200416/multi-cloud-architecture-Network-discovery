# Setup Guide — GCP + Azure Network Diagram Tool

## 1. Install

```bash
cd network_diagram
pip install -r requirements.txt --break-system-packages
```

`--break-system-packages` is only needed on systems where pip refuses to install outside a virtualenv (common on newer Debian/Ubuntu). Using a virtualenv instead is fine too:

```bash
python -m venv venv
source venv/bin/activate          # Windows: venv\Scripts\activate
pip install -r requirements.txt
```

## 2. GCP credentials

The tool uses Application Default Credentials (ADC) — the same auth `gcloud` and every `google-cloud-*` library use.

```bash
gcloud auth application-default login
```

The account (or service account, if run in CI) needs at least **Viewer** on the target project — everything this tool does is read-only.

## 3. Azure credentials (optional — skip if you're GCP-only)

Create a Service Principal:

```bash
az ad sp create-for-rbac --name "network-diagram-reader" --role Reader \
  --scopes /subscriptions/<sub-id-1> /subscriptions/<sub-id-2>
```

This prints `appId`, `password`, and `tenant` — you'll need all three below.

**Reader** is enough; the tool never writes anything. If you want RBAC-based "who actually uses this resource" edges (see §6), the Service Principal also needs `Microsoft.Authorization/roleAssignments/read`, which Reader already includes.

## 4. Configure `.env`

Create a `.env` file in the same directory as `main.py`:

```ini
# ── GCP (required) ──
PROJECT_ID=your-gcp-project-id

# ── Azure (optional — omit entirely to stay GCP-only) ──
AZURE_TENANT_ID=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
AZURE_CLIENT_ID=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
AZURE_CLIENT_SECRET=your-client-secret-value
AZURE_SUBSCRIPTION_IDS=sub-id-1,sub-id-2

# ── Azure (optional filter) ──
# Limit discovery to specific Resource Groups. Must match the REAL
# Azure RG name exactly (case-insensitive) -- e.g. "boldsign-dev", not
# a human-friendly guess like "BoldSign Development".
AZURE_RESOURCE_GROUPS=boldsign-dev,boldsign-payments

# ── Reporting (optional — omit to keep the Executive Summary a plain
#    templated sentence instead of an AI-written one; see §7) ─────────
ENABLE_AI_SUMMARY=false
ANTHROPIC_API_KEY=
```

**`AZURE_SUBSCRIPTION_IDS` is comma-separated**, not a single ID — this matters if you have some products in a dedicated subscription and others sharing one subscription across multiple Resource Groups.

**Don't want to maintain that list?** Leave `AZURE_SUBSCRIPTION_IDS` empty, install `azure-mgmt-subscription`, and grant the Service Principal Reader at **Management Group** scope instead — the tool will auto-discover every subscription it can see.

## 5. Run it

```bash
python main.py --project your-gcp-project-id
```

(or omit `--project` if `PROJECT_ID` is set in `.env`)

Output goes to `output/<project-id>/`:
- `network.mmd` — Mermaid source
- `network.drawio` — open directly in draw.io desktop or app.diagrams.net (real Azure icons render here)
- `network.html` — browser-viewable draw.io embed
- `network_interactive.html` — fully standalone, hover/click to trace connections (works offline, no draw.io needed)
- `infra_report.docx` — resource inventory + network topology as a Word document (see §7)

## 6. Running for all projects (monthly audit)

For a monthly audit across an org, run without `--project` and add `--all-projects` instead:

```bash
python main.py --all-projects
python main.py --all-projects --workers 8   # more concurrency
```

This discovers every GCP project the authenticated account can see (`resourcemanager.projects.list`, `lifecycleState:ACTIVE`) and runs the full pipeline for each one, exactly as if you'd run `python main.py --project <id>` per project. Output stays isolated per project under `output/<project-id>/` — each project gets its own fresh run with zero shared state between projects, so nothing from one project's data can end up in another's files.

Projects run **concurrently**, `--workers` at a time (default `4`), since almost all the runtime is spent waiting on GCP API responses rather than local computation. Each project's console output is buffered and printed as one clean block when that project finishes, so logs from different projects never interleave.

Requirements:
- The account needs `resourcemanager.projects.list` (e.g. Viewer at the org or folder level) to enumerate projects, plus the normal per-project Viewer role described in §2 for each project's own discovery calls.
- If a single project fails (missing permissions, API not enabled, etc.), it's logged and skipped — the batch continues with the remaining projects rather than aborting.
- **`--workers` and API quotas**: ADC here is one caller identity shared across every project, and most GCP read quotas (Compute Engine, Resource Manager, etc.) are enforced *per caller*, not per target project — so raising `--workers` too high risks `429`/`RESOURCE_EXHAUSTED` errors across all projects rather than more speed. Start at the default and raise gradually while watching the output for rate-limit messages.
- There's no built-in include/exclude filter yet — it runs every `ACTIVE` project visible to the account. If you need to skip sandbox/test projects, filtering by name/label is a small follow-up, not currently implemented.

`--project <id>` still works exactly as before, runs sequentially with unbuffered live output, and takes precedence for a single-project run; `.env`'s `PROJECT_ID` is only used when neither flag is passed.

## 7. Infrastructure report (.docx)

Every run (single-project or `--all-projects`) also produces `infra_report.docx` — a stakeholder-readable Word document per project: resource inventory counts, per-VPC subnet tables, load balancers, and managed/PaaS services (Cloud SQL, Cloud Storage, Memorystore, Pub/Sub, Cloud Run).

**Every number and name in the report is a direct transcription of the same discovery data the diagrams are built from — no AI model is involved in that path.** Nothing there can be hallucinated because nothing there was ever generated; it's formatted, not written.

The only optional AI-touched piece is the **Executive Summary** paragraph at the top, off by default. To enable it, add to `.env`:

```ini
ENABLE_AI_SUMMARY=true
ANTHROPIC_API_KEY=sk-ant-...
```

When enabled, the model is given *only* the pre-computed resource facts as JSON and instructed to describe them and nothing else. Its output is then checked automatically: every number and every resource-name-shaped token in the generated text must already appear in those facts, or the paragraph is discarded and a plain templated sentence is used instead — this happens silently and safely, logged to the console, never as a failure. This is a mechanical filter, not a request for good behavior: it doesn't rely on the model choosing not to embellish, it verifies the output regardless. If `python-docx` isn't installed or the report otherwise fails, every other output (diagrams) is still produced — the report is one more independently fault-isolated stage, same pattern as the renderers below.

## 8. Optional features and what unlocks them

| Package | Unlocks | If missing |
|---|---|---|
| `azure-mgmt-resource` | Resource Group listing + full "Other Azure Resources" inventory (Storage Accounts, Key Vaults, App Services, SQL, etc. beyond the VM/AKS/LB subset) | Falls back to network-only Azure discovery |
| `azure-mgmt-subscription` | Auto-discovering subscriptions instead of listing them explicitly | Must set `AZURE_SUBSCRIPTION_IDS` |
| `azure-mgmt-authorization` | Real RBAC-based edges — a Storage Account/Key Vault connects to the VM/AKS whose managed identity actually has a role on it | "Other Resources" still render, just without connection edges (unless exactly one VM/AKS exists in that RG, which still gets a fallback edge) |
| `google-cloud-asset` | GCS/Pub-Sub IAM resolution fetches every resource's IAM policy in **one** Cloud Asset Inventory call instead of one live `getIamPolicy()` call per bucket/topic — a real speed win on projects with many buckets or topics, and it multiplies across `--all-projects` runs | Falls back automatically to one live API call per bucket/topic — identical results, just more API round trips |

None of these are required to run the tool — every one fails open with a console note if it's missing, never a crash.

**Enabling the Cloud Asset Inventory speedup**, beyond `pip install google-cloud-asset`:
```bash
gcloud services enable cloudasset.googleapis.com --project your-gcp-project-id
```
The account also needs `roles/cloudasset.viewer` (or just the `cloudasset.assets.searchAllIamPolicies` permission) on the project — this is a separate grant from the base Viewer role in §2. If it's missing, discovery falls back to the original per-resource calls automatically; nothing breaks, it's just slower.

## 9. Known gotchas (found the hard way — worth knowing up front)

- **`azure-mgmt-resource` import errors ("unknown location")**: this means a legacy `azure-common` package is fragmenting the `azure.*` namespace — a known conflict between `azure-common` (pre-2019 SDK style) and modern `azure-mgmt-*` packages (native namespace packages). The tool has a built-in fallback import path for this and will keep working correctly even with the conflict present (confirmed: reproduced this exact issue while testing this setup guide, and the tool auto-recovered). If you want to actually clean it up rather than rely on the fallback: `pip uninstall azure-common -y` then reinstall `azure-mgmt-resource` fresh.
- **`AZURE_RESOURCE_GROUPS` filter matches nothing**: the value must be the *actual* Azure RG name (check the Portal), not a guessed display name. `boldsign-dev` ≠ `BoldSign Development`.
- **Copy-pasting GUIDs from a wrapped terminal line**: characters get silently dropped. Copy `AZURE_CLIENT_ID`/`AZURE_CLIENT_SECRET`/`AZURE_TENANT_ID` directly from the Azure Portal's copy-icon, not by dragging across wrapped terminal text.
- **VS Code/Pylance shows import errors even after `pip install`**: check the interpreter VS Code is pointed at (bottom-right corner, or `Ctrl+Shift+P → Python: Select Interpreter`) — it may differ from the one you installed into.

## 10. Architecture (for anyone extending this)

```
main.py            orchestrator — runs every stage below in order
config.py           env vars, CLI args, constants
helpers.py           pure utility functions, no shared state
gcp_discovery.py      all GCP API calls
azure_discovery.py    all Azure API calls
connectivity.py       runs after both discoveries: cross-cloud VPN
                       matching + firewall-rule-based VM connectivity
layout_engine.py      cloud-agnostic auto-layout engine (used by both
                       draw.io and interactive HTML renderers)
render_mermaid.py     → network.mmd
render_drawio.py      → network.drawio + network.html
render_interactive.py → network_interactive.html
render_report.py      → infra_report.docx (see §7)
```

Each stage runs via `exec()` into one shared namespace (see the comment at the top of `main.py` for why — short version: this codebase has heavy cross-section shared state, and this preserves exact tested behavior rather than risking a large rename-based refactor). To add a new discovery source or connectivity signal, add a new file following the existing ones' pattern and register it in `main.py`'s `_STAGES` list in the right dependency position.