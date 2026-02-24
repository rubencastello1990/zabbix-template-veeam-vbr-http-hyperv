# Changes vs official template (Zabbix 7.2-1)

## Summary

One item modified: `veeam.get.metrics` JavaScript.  
All UUIDs regenerated. Template renamed for parallel import.  
Everything else is identical to the official template.

---

## Template metadata

| Field | Official | This fork |
|-------|----------|-----------|
| `template:` | `Veeam Backup and Replication by HTTP` | `Veeam Backup and Replication by HTTP (Hyper-V enhanced)` |
| `name:` | same | same new name |
| `vendor.version` | `7.2-1` | `7.2-1-hyperv` |
| All UUIDs | original | regenerated (deterministic, uuid5) |

---

## Item `veeam.get.metrics` — JavaScript diff

### Added: `requestSafe(url)` method

```javascript
// NEW method — returns null on HTTP error instead of throwing.
// Allows optional/fallback endpoint calls without aborting the collection.
requestSafe: function (url) {
    var response, request = new HttpRequest(), status;
    // ... (proxy / auth headers same as request()) ...
    status = request.getStatus();
    if (status !== 200 || response === null) {
        Zabbix.log(4, '[ VEEAM ] requestSafe: ' + url + ' returned HTTP ' + status + '.');
        return null;
    }
    try { return JSON.parse(response); }
    catch (error) {
        Zabbix.log(4, '[ VEEAM ] requestSafe: failed to parse response from ' + url + '.');
        return null;
    }
},
```

### Modified: `getMetricsData()` — appended after primary fetch loop

The five primary endpoint calls are **unchanged**. After them, this block runs:

```javascript
// 1. Log primary job count
var primaryJobs = data.jobs_states?.data ?? [];
Zabbix.log(4, '[ VEEAM ] api/v1/jobs/states returned N job(s).');

// 2. Build set of already-known job IDs
var primaryIds = {};  // {jobId: true}

// 3. Fetch api/v1/jobs (safe — Hyper-V jobs appear here in VBR 12.1+)
var jobsList = Veeam.requestSafe(endpoint + 'api/v1/jobs');
var jobsById = {};  // {jobId: jobObject}

// 4. Index sessions already collected by jobId (NO extra HTTP call)
var sessionsByJob = {};  // {jobId: [session, ...]}

// 5. Candidates = jobs in api/v1/jobs OR sessions, minus primaryIds
var candidateIds = {};

// 6. For each candidate:
//    - Sort its sessions descending by creationTime
//    - Derive status:     running / disabled / inactive
//    - Derive lastResult: None / Success / Warning / Failed
//    - Derive workload:   job.workload || job.platform || 'HyperV'
//    - Build {id, name, type, workload, status, lastResult, repositoryName}

// 7. Concat newJobs into data.jobs_states.data
//    → LLD rule reads $.jobs_states.data unchanged
```

---

## No other changes

| Artifact | Status |
|----------|--------|
| Discovery rule `veeam.job.state.discovery` | Unchanged (JSONPath, macros, filters) |
| Item prototypes (jobs, proxies, repos, sessions) | Unchanged |
| Trigger prototypes | Unchanged (expressions updated only to use new template name) |
| Proxies / Repositories / Sessions discovery rules | Unchanged |
| All macros | Unchanged |
| Valuemaps | N/A (template has none) |
