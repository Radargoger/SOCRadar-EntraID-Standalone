# Publishing Guide — SOCRadar Entra ID Integration

This guide is for the SOCRadar engineer who will publish this integration to the SOCRadar GitHub organization (or any other namespace) so customers can deploy from your branded repository.

The standalone bundle is currently namespaced to `orcunsami/SOCRadar-Azure-Entra-ID`. To republish under SOCRadar branding, follow the steps below.

## Prerequisites

- A GitHub organization (e.g. `SOCRadar-Inc`) with admin access
- Azure CLI installed locally (only for the publisher — customers never use Azure CLI)
- Bicep CLI ≥ 0.30 (`az bicep upgrade`)

## Step 1 — Create a new public repository

In the SOCRadar GitHub organization, create a new public repository:

```
SOCRadar-Inc/SOCRadar-Azure-Entra-ID
```

(or any name you prefer — substitute everywhere below)

## Step 2 — Copy source files into the repository

Extract this delivery bundle, then copy the following files into the new repo:

```
production/
scripts/socradar-entraid-fic.sh
README.md
CHANGELOG.md
LICENSE
PUBLISHING.md (this file — optional, internal)
```

Do **not** include `FunctionApp.zip` in the repository — it goes into a GitHub release (Step 4).

## Step 3 — Find-and-replace the GitHub namespace

The ARM template and README files contain references to `orcunsami/SOCRadar-Azure-Entra-ID`. Replace these with your namespace.

**Files to update:**

1. **`production/azuredeploy.bicep`** — the `WEBSITE_RUN_FROM_PACKAGE` default URL:
   ```bicep
   value: 'https://github.com/orcunsami/SOCRadar-Azure-Entra-ID/releases/download/v1.0.0/FunctionApp.zip'
   ```
   Change to:
   ```bicep
   value: 'https://github.com/SOCRadar-Inc/SOCRadar-Azure-Entra-ID/releases/download/v1.0.0/FunctionApp.zip'
   ```

2. **`README.md`** (top-level) — the Deploy to Azure button URL and raw template URL.

3. **`production/README.md`** — the Deploy to Azure button URL.

A quick sed pass:

```bash
cd path/to/socradar-azure-entra-id
grep -rl "orcunsami/SOCRadar-Azure-Entra-ID" . \
  | xargs sed -i '' 's|orcunsami/SOCRadar-Azure-Entra-ID|SOCRadar-Inc/SOCRadar-Azure-Entra-ID|g'
```

Verify nothing else (e.g., `b0afca82-...` test App Registration ID) was missed:

```bash
grep -rn "orcunsami" .
```

Should return zero hits after the replace.

## Step 4 — Rebuild the ARM template from Bicep

After updating the URL in `azuredeploy.bicep`, recompile to refresh `azuredeploy.json`:

```bash
cd path/to/socradar-azure-entra-id/production
az bicep build --file azuredeploy.bicep
```

Verify the new URL is in `azuredeploy.json`:

```bash
grep "FunctionApp.zip" azuredeploy.json
# Should show: https://github.com/SOCRadar-Inc/SOCRadar-Azure-Entra-ID/releases/download/v1.0.0/FunctionApp.zip
```

## Step 5 — Commit + push to your repository

```bash
git init
git add .
git commit -m "Initial release v1.0.0 — SOCRadar Entra ID Integration for Microsoft Sentinel"
git branch -M master
git remote add origin git@github.com:SOCRadar-Inc/SOCRadar-Azure-Entra-ID.git
git push -u origin master
```

## Step 6 — Create the v1.0.0 release with FunctionApp.zip

This is **critical**. The ARM template's `WEBSITE_RUN_FROM_PACKAGE` setting points to a specific URL — that URL must serve `FunctionApp.zip`.

1. Go to your GitHub repository → **Releases** → **Draft a new release**
2. Tag: `v1.0.0`, target: `master`
3. Title: `v1.0.0 — SOCRadar Entra ID Integration`
4. Description: copy from `CHANGELOG.md` for v1.0.0
5. Attach the `FunctionApp.zip` file from this delivery bundle (in the zip root)
6. **Publish release**

After publishing, verify the asset URL is accessible:

```bash
curl -sIL "https://github.com/SOCRadar-Inc/SOCRadar-Azure-Entra-ID/releases/download/v1.0.0/FunctionApp.zip" -o /dev/null -w "HTTP %{http_code}\n"
# Should return: HTTP 200
```

## Step 7 — Test the Deploy to Azure button

Click the **Deploy to Azure** button on your new repository's README. Verify the form opens with the correct ARM template URL pointing to your repo. You can cancel without deploying — the goal is just to confirm the link works.

## Step 8 — (Optional) Publish to the Azure Marketplace via Content Hub

If you want customers to install from Microsoft Sentinel's Content Hub (not just the GitHub button), use the **Content Hub Solution bundle** (`SOCRadar-EntraID-ContentHub-v1.0.0.zip`) shipped alongside this standalone bundle. The Content Hub Solution is a separate delivery and follows Microsoft's Solution Submission process — see Microsoft Sentinel partner documentation.

## What is and is NOT in this bundle

**Included:**
- `production/` — full ARM template source (Bicep + compiled JSON), Function App Python code, workbooks, and customer-facing docs.
- `scripts/socradar-entraid-fic.sh` — post-deploy helper for the reuse path (customers who provide an existing App Registration via `EntraIdClientId` and set `SkipFicCreation=true` can run this helper from Azure Cloud Shell after deployment).
- `README.md`, `CHANGELOG.md`, `LICENSE` — customer-facing project files.
- `CUSTOMER-TEST-RUNBOOK.md` — 60-minute customer acceptance test runbook (9 test users × 3 sources; forward to customers as their onboarding validation script).
- `FunctionApp.zip` — the deployable Function App package (host this on your GitHub release; the ARM template's `WEBSITE_RUN_FROM_PACKAGE` setting points to its URL).

**Features in this release:**
- 3 SOCRadar sources (Botnet / PII / VIP), configurable polling cadence
- Multi-tenant lookup (`EntraIdTenantIds` CSV) — one deployment, N tenants
- **Verified-domain allowlist** (`EntraIdVerifiedDomains` CSV) — optional gate that restricts Microsoft Graph lookups to customer-attached verified domains; off-allowlist records land in LAW with `entra_status="skipped_domain_allowlist"` (audit only, no action)
- 11 togglable remediation actions (revoke session by default; high-impact actions off)
- 4 LAW tables via DCR-based ingestion + 4 Sentinel workbooks
- Secretless auth (Workload Identity Federation)

**NOT included (intentionally):**
- `AI_KIMI.md`, `soru-cevap.txt`, `development/`, `archive/`, internal docs — these are SOCRadar dev notes.
- `scripts/deploy.sh`, `scripts/validate.sh`, `scripts/e2e_test.py`, etc. — these are SOCRadar internal test scripts, not customer-facing.
- `tests/` — internal test suite.
- `to-Radargoger/` — the delivery folder itself.

## Customer flow (after you publish)

Customers go to your repository and click **Deploy to Azure**. The Azure Portal opens with the ARM template:

| Customer profile | Deploy parameters | Manual step after deploy |
|------------------|--------------------|---------------------------|
| Cloud Application Administrator | Leave `EntraIdClientId` empty, `GrantAdminConsent=true` | 0 (zero-touch) |
| Application Administrator | Leave `EntraIdClientId` empty, `GrantAdminConsent=false` | 1 portal click (admin consent in Entra) |
| Customer with pre-existing consented App Registration | Set `EntraIdClientId=<app-id>`, `SkipFicCreation=true` | 1 CLI command (helper script in Azure Cloud Shell) |

Multi-tenant customers (MSSP / holdings) set `EntraIdTenantIds` to a CSV of tenant GUIDs and `EntraIdVerifiedDomains` to the union of every verified domain attached to those tenants. See `production/docs/multi-tenant-setup.md`.

See `production/README.md` for full parameter reference and `CUSTOMER-TEST-RUNBOOK.md` for the controlled acceptance test you can forward to customers.

## Updating to a new version

When you release a new version (e.g. v1.0.1):

1. Update `metadata version` in `production/azuredeploy.bicep`
2. Update `WEBSITE_RUN_FROM_PACKAGE` URL to the new version tag
3. Rebuild Bicep
4. Update `CHANGELOG.md`
5. Commit + push
6. Create new GitHub release with new `FunctionApp.zip`

Existing customer deployments will continue to use the old version (their `WEBSITE_RUN_FROM_PACKAGE` setting is pinned at deploy time). They redeploy to upgrade.

## Support

For questions about publishing or the integration itself, contact the original maintainer.
