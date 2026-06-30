---
name: splunk-workload-mgmt
description: >-
  Splunk workload management and search capacity — all tasks. Trigger for:
  workload management, workload pools, workload rules, admission rules,
  workload_pools.conf, workload_policy.conf, search admission control, search
  concurrency limits, search quotas, search head capacity planning, search
  scheduler, scheduler.conf, search job priorities, auto-summarization limits,
  real-time search limits, search head CPU/memory resource allocation, search
  cannibalism, search head cluster scheduler, search head pooling (legacy),
  Monitoring Console search activity, search performance tuning at the SH level.
  Default: Splunk Enterprise 9.x.
---

# Splunk Workload Management

Router only. Read `references/workload-mgmt.md` for all workload management tasks.

## Routing

| Task | Action |
|---|---|
| Define workload pools and rules | Read `references/workload-mgmt.md` |
| Admission rules — user, role, app, search type | Read `references/workload-mgmt.md` |
| Search concurrency limits, quotas | Read `references/workload-mgmt.md` |
| Scheduler tuning, scheduled search priority | Read `references/workload-mgmt.md` |
| Indexer-side resource tuning | Defer to `splunk-distributed` |
| SPL query optimization | Defer to `splunk-spl` |

## Shared conventions

- Define a `default` workload pool as a catch-all before creating specific pools — every search must land somewhere.
- Workload rules match in order; put narrow rules (specific user/role) before broad ones (app-wide).
- `guaranteed_cpu_perc` across all pools must not exceed 100%; leave headroom for system processes (≤90% total).
- Limit ad-hoc search concurrency per role in `authorize.conf` (`srchJobsQuota`); use workload rules to enforce CPU caps.
- Real-time searches bypass the scheduler — cap with `max_rt_search_multiplier` in `limits.conf`.
- Scheduled searches: set `priority` (0–10) in `savedsearches.conf`; use `max_concurrent` to cap parallel runs.
- Monitor `index=_internal sourcetype=scheduler` for skipped searches and `run_time` outliers.
- Apply workload management on the search head (or deployer for SHC) — not on indexers or HFs.
- Test rule changes in a non-production SH first; incorrect pool allocation can starve critical scheduled searches.

## Output format

Fenced `ini` for conf files. Decision table for pool design. ⚠️ callouts for starvation, scheduler skipping, and over-allocation risks.
