# SOCRadar Entra ID Integration — Customer Acceptance Test Runbook

**Audience:** Customer IT / SOC engineer running the integration for the first time.
**Goal:** Confirm the integration ingests SOCRadar alarms, matches them to Microsoft Entra ID identities, takes the configured remediation actions, and lands records in Log Analytics — end to end — using a controlled set of 3 throwaway test users per source (botnet / PII / VIP).
**Duration:** ~60 minutes (10 min setup at customer side, 10 min setup at SOCRadar side, 30 min deploy + observation, 10 min cleanup).

---

## 0. What's exchanged before the test

| Item | Provided by | Sent to |
|------|-------------|---------|
| List of 9 test usernames (3 per source) — e.g. `entratest-botnet-1@<customer-domain>`, `entratest-pii-1@<customer-domain>`, `entratest-vip-1@<customer-domain>`, etc. | Customer | SOCRadar |
| 9 passwords for those users | **Customer keeps internally** (NEVER emailed). Customer types each password locally when creating the user. SOCRadar does NOT need plaintext passwords for this test — the dummy alarms use placeholder strings. | — |
| `EntraIdClientId` (App Registration object ID — the pre-consented one) | SOCRadar / customer admin | Deploy parameter |
| `SocradarApiKey` + `SocradarCompanyId` | SOCRadar account team | Deploy parameter |

**Why passwords stay at the customer:** the integration looks up users by email in Microsoft Graph and takes account-level actions (`revoke_session`, `disable_account`, etc.). It does NOT verify passwords against SOCRadar's leaked-credential payloads in this test (ROPC is off by default). The customer can rotate / delete the test passwords after the run; SOCRadar never sees them.

---

## 1. Customer side — create 9 test users in Entra ID (10 min)

Use the same primary domain that will be in `EntraIdVerifiedDomains` (if you plan to set that parameter). One-time setup:

| User UPN | Source it will exercise |
|----------|------------------------|
| `entratest-botnet-1@<your-domain>` | Botnet |
| `entratest-botnet-2@<your-domain>` | Botnet |
| `entratest-botnet-3@<your-domain>` | Botnet |
| `entratest-pii-1@<your-domain>`    | PII |
| `entratest-pii-2@<your-domain>`    | PII |
| `entratest-pii-3@<your-domain>`    | PII |
| `entratest-vip-1@<your-domain>`    | VIP |
| `entratest-vip-2@<your-domain>`    | VIP |
| `entratest-vip-3@<your-domain>`    | VIP |

**Steps (Azure Portal):**
1. Sign in as **User Administrator** (or higher).
2. **Microsoft Entra ID → Users → New user → Create new user**.
3. Set the UPN from the table above, give a strong unique password, **uncheck** "Block sign-in", leave usage location set.
4. Repeat for all 9 users.
5. (Recommended) Put all 9 users into a security group (e.g. `SOCRadar-IntegrationTest`) so cleanup is one click later.
6. Sign in once with at least one user (e.g. `entratest-botnet-1`) and accept the first-sign-in prompts so its `accountEnabled` + `signInActivity` is populated for Graph.

**Send to SOCRadar:** the list of 9 UPNs (emails only — no passwords).

---

## 2. SOCRadar side — seed 9 dummy alarms on preprod (10 min)

SOCRadar operations team creates 3 dummy alarms in each source for the customer's company ID:

| Source | Alarm count | Sample payload fields |
|--------|-------------|-----------------------|
| Botnet Data | 3 | `email` (one of `entratest-botnet-{1,2,3}`), `password` (any placeholder), `url`, `deviceIP`, `country`, `logDate` within last 30 days, `isEmployee: true` |
| PII Exposure | 3 | `email` (one of `entratest-pii-{1,2,3}`), exposure source URL, `isEmployee: true` |
| VIP Protection | 3 | `email` (one of `entratest-vip-{1,2,3}`) |

Each alarm gets a unique `alarmId` so resolve-alarm semantics work later.

Verify the alarms appear in the SOCRadar API:

```
GET https://preprod.socradar.com/api/company/{COMPANY_ID}/dark-web-monitoring/botnet-data/v2?startDate=YYYY-MM-DD
Header: API-Key: <preprod-key>
```

— should return 3 records for the customer.

Confirm to customer: "Alarms seeded. Ready for deploy."

---

## 3. Customer side — deploy via Deploy to Azure (15 min)

1. Open the repo: `https://github.com/Radargoger/SOCRadar-EntraID-Standalone`
2. Click **Deploy to Azure** in the README.
3. Fill the form:
   - **Workspace Name** — pick your existing Sentinel workspace, or any free name for a new one.
   - **Socradar Api Key** — preprod key from SOCRadar.
   - **Socradar Company Id** — your preprod company ID.
   - **EntraIdClientId** — App Registration object ID provided by SOCRadar.
   - **EntraIdVerifiedDomains** — your primary domain (e.g. `acme.com`). Optional but recommended.
   - **EnableBotnetSource / EnablePiiSource / EnableVipSource** — leave all true.
   - **EnableRevokeSession** — true. Other actions optional for this test (start minimal).
4. Review + Create. Wait for `Succeeded` (~5–8 minutes).
5. If addFic deployment script fails with "Insufficient privileges", that's the **reuse path** — run the helper:
   ```bash
   bash scripts/socradar-entraid-fic.sh <YOUR_RG_NAME>
   ```
   (download `scripts/socradar-entraid-fic.sh` from the repo, run from Azure Cloud Shell).
6. The helper restarts the Function App and triggers the first run.

---

## 4. Customer side — verify in Log Analytics (15 min)

Wait ~5 minutes after the trigger fires, then open **Log Analytics workspace → Logs** and run:

### A. Per-record results — all 9 test users found?

```kql
SOCRadar_PII_CL
| where TimeGenerated > ago(30m)
| project TimeGenerated, email, entra_status, actions_taken, entra_tenant_id
| sort by TimeGenerated desc
```

```kql
SOCRadar_Botnet_CL | where TimeGenerated > ago(30m) | project TimeGenerated, email, entra_status, actions_taken
```

```kql
SOCRadar_VIP_CL | where TimeGenerated > ago(30m) | project TimeGenerated, email, entra_status, actions_taken
```

**Expected:**
- 3 rows per source (9 total)
- `entra_status` = `found` for every test user
- `actions_taken` includes `revoke_session` (and any other enabled actions)
- `entra_tenant_id` = your Entra ID Tenant GUID

### B. Per-source summary

```kql
SOCRadar_EntraID_Audit_CL
| where TimeGenerated > ago(30m)
| sort by TimeGenerated desc
| project TimeGenerated, source, total_records, found_count, not_found_count, error_count, actions_taken
```

**Expected:**
- One row per source per run
- `found_count = 3` for each source
- `error_count = 0`
- `actions_taken = 3 × number-of-enabled-actions`

### C. Domain allowlist working? (only if `EntraIdVerifiedDomains` was set)

```kql
union SOCRadar_Botnet_CL, SOCRadar_PII_CL, SOCRadar_VIP_CL
| where TimeGenerated > ago(30m)
| summarize Count=count() by entra_status
```

**Expected:** `found=9`, no `skipped_domain_allowlist` rows (because all 9 test users live in the verified domain). To negatively prove the filter:

1. Azure Portal → Function App → **Configuration** → set `ENTRA_ID_VERIFIED_DOMAINS` to a non-matching domain (e.g. `nomatch.example`).
2. Function App → **Overview → Restart**.
3. Re-run the trigger from the SCM/admin endpoint OR wait for the next timer cycle.
4. The same 9 records now land with `entra_status="skipped_domain_allowlist"` and `actions_taken=[]`. No Graph lookups happen.
5. Set the parameter back to your real domain afterwards.

### D. Workbooks (visual confirmation)

**Microsoft Sentinel → Workbooks → My workbooks** — should list:
- SOCRadar Entra ID — Botnet
- SOCRadar Entra ID — PII
- SOCRadar Entra ID — VIP
- SOCRadar Entra ID — Combined Dashboard

Open the Combined workbook. The `TenantId` dropdown should show your tenant. Tiles should show 9 compromised users total, 3 per source.

### E. Verify the actual remediation hit Entra ID

For each test user, open **Microsoft Entra ID → Users → `<user>` → Sign-in logs**. The most recent entry should show a forced sign-out / revoked session within the time window of the function run. (Optional: try signing in with a test user that you signed in with earlier — it should re-prompt for credentials because the session was revoked.)

---

## 5. Cleanup (5 min)

**Customer side:**
1. **Microsoft Entra ID → Users** → delete the 9 test users (or remove the `SOCRadar-IntegrationTest` group + members).
2. (Optional) Tear down the resource group created by the Deploy to Azure flow if this was a one-off test environment.
3. (If you changed `ENTRA_ID_VERIFIED_DOMAINS` for the negative test) Restore it to your real domain in the Function App configuration.

**SOCRadar side:**
1. Mark the 9 dummy alarms `RESOLVED` (or delete) on preprod so they don't bleed into the next test.

---

## 6. Pass / Fail criteria

The test is **PASS** when:
- [ ] 9 LAW records (3 per source) with `entra_status="found"`
- [ ] Audit table shows `found_count=3, error_count=0` for each of the 3 sources
- [ ] At least one Entra ID test user shows a revoked-session event in its sign-in logs
- [ ] (If verified-domain test was run) records with non-matching domain land as `entra_status="skipped_domain_allowlist"` and zero Graph calls occur

If any expectation fails, capture the LAW row and the Function App's App Insights trace, then open a ticket with SOCRadar referencing the deployment's resource group + UTC timestamp.

---

## Common issues seen during first-run test

| Symptom | Likely cause | Fix |
|---------|--------------|-----|
| `entra_status="skipped_no_token"` for every record | UAMI federated credential not attached to App Reg | Re-run `socradar-entraid-fic.sh` |
| `entra_status="lookup_permission_denied"` | Admin consent missing on Graph permissions | Microsoft Entra ID → App registrations → `<app>` → API permissions → **Grant admin consent** |
| `entra_status="not_found"` for some test users | UPN mismatch — alarm email differs from Entra UPN | Make sure SOCRadar dummy alarms use the exact UPN (including the tenant domain suffix), not a display name |
| Audit `total_records=0` for every source | Function App pulled too narrow a lookback window | Set `INITIAL_LOOKBACK_MINUTES=86400` (60 days) in App Settings; restart; trigger; reset the storage-account `EntraIDState` rows |
| LAW shows nothing 10 minutes after trigger | Custom-table first-write ingestion latency | Wait another 5–10 minutes; if still empty, check Function App **Log stream** for ingestion errors |
