# Veeam Backup and Replication by HTTP — Hyper-V enhanced

Zabbix 7.2 template based on the official **"Veeam Backup and Replication by HTTP"**,
with added Hyper-V backup job discovery support.

Designed for **parallel testing**: UUIDs and template name are different, so this
template can coexist with the official one in the same Zabbix instance.

---

## Problem: why Hyper-V jobs were not discovered

The official template uses `api/v1/jobs/states` as the **sole** source for the
**"Jobs states discovery"** LLD rule:

```
GET api/v1/jobs/states  →  $.jobs_states.data  →  LLD
```

In Veeam Backup & Replication (REST API port 9419, `x-api-version: 1.0-rev2`),
**`api/v1/jobs/states` only returns VMware vSphere backup jobs**.  
Hyper-V backup jobs are managed by a different internal subsystem and are simply
not exposed through that endpoint in VBR ≤ 12.0 and several 12.1 builds.

**Symptom**: proxies, repositories and sessions are discovered correctly (their
endpoints are platform-agnostic), but `Jobs states discovery` stays empty.

---

## Solution implemented

A **minimal, backward-compatible** change to the `veeam.get.metrics` JavaScript item.
All other rules, item prototypes, triggers and macros are **unchanged**.

### New method: `requestSafe(url)`

Identical to `request()` but returns `null` on HTTP error instead of throwing.
Used for optional/fallback endpoints that must not abort the full collection.

### Modified method: `getMetricsData()` — Hyper-V block

After the five primary endpoint calls (unchanged), the script now:

1. Logs how many jobs `api/v1/jobs/states` returned (`Zabbix.log` level 4).
2. Calls **`api/v1/jobs`** safely → in VBR 12.1+ this endpoint includes Hyper-V jobs.
3. **Indexes sessions by `jobId`** from the already-fetched `api/v1/sessions` data —
   **zero extra HTTP requests**.
4. Collects **candidate IDs**: jobs in `api/v1/jobs` OR sessions whose `jobId` is
   not already in `jobs/states` (deduplication against VMware jobs).
5. For each candidate derives:
   - `status` — `running` / `disabled` / `inactive` from latest session state +
     job metadata.
   - `lastResult` — `None / Success / Warning / Failed` from latest session result.
6. Pushes synthetic job objects (`{id, name, type, workload, status, lastResult,
   repositoryName}`) into `data.jobs_states.data` — the same array the LLD reads.
7. Logs a summary of how many jobs were added by the fallback.

**VMware environments**: `jobs/states` returns all jobs → `candidateIds` is empty →
zero synthetic jobs added → behaviour identical to the official template.

---

## Compatibility

| VBR version | VMware jobs | Hyper-V jobs | Notes |
|-------------|-------------|--------------|-------|
| ≤ 12.0      | ✅ `jobs/states` | ✅ sessions fallback | `api/v1/jobs` may be VMware-only; sessions fill the gap |
| 12.1        | ✅ `jobs/states` | ✅ `api/v1/jobs` | `api/v1/jobs` includes HyperV |
| 13.x        | ✅ `jobs/states` | ✅ `api/v1/jobs` | Full support |
| Zabbix      | 7.2+        | 7.2+         | Uses Script item + DEPENDENT items |
| API version | `1.0-rev2`  | `1.0-rev2`   | No change required |

---

## Setup

Same process as the official template:

1. Create a read-only Veeam account (or reuse an existing one).
2. Import **`template_veeam_backup_replication_http_hyperv.yaml`** in Zabbix
   (Configuration → Templates → Import).
3. Link the template to a host.
4. Configure macros:

| Macro | Example | Description |
|-------|---------|-------------|
| `{$VEEAM.API.URL}` | `https://vbr.corp:9419` | VBR REST API base URL |
| `{$VEEAM.USER}` | *(secret)* | Veeam account username |
| `{$VEEAM.PASSWORD}` | *(secret)* | Veeam account password |
| `{$VEEAM.DATA.TIMEOUT}` | `10` | Per-request timeout (seconds) |
| `{$CREATED.AFTER}` | `7` | Session lookback window (days) |

---

## How to test (curl)

Replace `<VBR>`, `<USER>`, `<PASS>` with real values. Add `-k` for self-signed certs.

### 1 — Obtain token
```bash
TOKEN=$(curl -sk -X POST "https://<VBR>:9419/api/oauth2/token" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -H "x-api-version: 1.0-rev2" \
  -d "grant_type=password&username=<USER>&password=<PASS>" \
  | jq -r .access_token)
echo "Token: ${TOKEN:0:30}..."
```

### 2 — Original endpoint (VMware jobs)
```bash
# On Hyper-V environments this returns: {"data":[],"pagination":...}
curl -sk "https://<VBR>:9419/api/v1/jobs/states" \
  -H "Authorization: Bearer $TOKEN" \
  -H "x-api-version: 1.0-rev2" \
  | jq '{total: (.data|length), sample: [.data[0] | {name,type,workload,status,lastResult}]}'
```

### 3 — Fallback endpoint (Hyper-V jobs)
```bash
# Should return Hyper-V backup jobs with id, name, type fields
curl -sk "https://<VBR>:9419/api/v1/jobs" \
  -H "Authorization: Bearer $TOKEN" \
  -H "x-api-version: 1.0-rev2" \
  | jq '{total: (.data|length), jobs: [.data[] | {id, name, type}]}'
```

### 4 — Sessions with jobId (used for state derivation)
```bash
# Verify sessions carry jobId — required for the fallback to work
curl -sk "https://<VBR>:9419/api/v1/sessions" \
  -H "Authorization: Bearer $TOKEN" \
  -H "x-api-version: 1.0-rev2" \
  | jq '[.data[] | select(.jobId != null) | {jobId, name, sessionType, state, result: .result.result}] | .[0:5]'
```

### 5 — Validate merged output
After the template runs (check item `veeam.get.metrics` value in Zabbix):
```bash
# jobs_states.data should contain entries for ALL job types
echo '<paste veeam.get.metrics value here>' | jq '{
  vmware_jobs: [.jobs_states.data[] | select(.workload == "VMware") | .name],
  hyperv_jobs:  [.jobs_states.data[] | select(.workload == "HyperV")  | .name],
  total: (.jobs_states.data | length)
}'
```

---

## Files

| File | Description |
|------|-------------|
| `template_veeam_backup_replication_http_hyperv.yaml` | Zabbix 7.2 template — import this |
| `README.md` | This document |
| `CHANGES.md` | Line-by-line diff vs official template |

---

## Base template

- **Source**: Official Zabbix 7.2 `Veeam Backup and Replication by HTTP`
- **Upstream**: https://git.zabbix.com/projects/ZBX/repos/zabbix/browse/templates/app/veeam/backup_replication_http?at=release/7.2
- **Changes**: Only `veeam.get.metrics` JavaScript (`params` field) + all UUIDs regenerated + template renamed
