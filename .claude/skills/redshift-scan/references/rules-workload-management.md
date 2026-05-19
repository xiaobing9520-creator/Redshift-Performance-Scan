# Workload Management Rules

## WL-01: Single WLM Queue (No Workload Separation)

**Severity**: MEDIUM
**Category**: Workload Management
**Trigger Condition**: Only 1 user-defined service class (service_class > 4) in stv_wm_service_class_config
**Diagnostic Queries**: D1

**Observation Template**:
> Only 1 WLM queue is configured. All queries share the same queue regardless of workload type, priority, or resource requirements.

**Recommendation**:
Create separate queues for different workload types:
- Short interactive queries (high concurrency, lower memory)
- Long-running ETL/reporting queries (low concurrency, higher memory)
- Dashboard/BI queries (medium concurrency)

Consider using Automatic WLM which dynamically manages memory and concurrency.

**Documentation Source**:
- https://docs.aws.amazon.com/redshift/latest/dg/cm-c-implementing-workload-management.html
- https://docs.aws.amazon.com/redshift/latest/dg/tutorial-configuring-workload-management.html

---

## WL-02: High Query Abort Rate

**Severity**: HIGH
**Category**: Workload Management
**Trigger Condition**: aborted_queries > 5% of total_queries for any service class
**Diagnostic Queries**: D2

**Observation Template**:
> WLM service class {service_class} has {aborted_queries} aborted queries out of {total_queries} total ({pct}%). Queries may be hitting timeout limits or QMR abort rules.

**Recommendation**:
Investigate why queries are being aborted:
1. Check if max_execution_time is too restrictive
2. Review QMR (Query Monitoring Rules) abort thresholds
3. Look for resource contention causing timeouts
4. Consider increasing timeout for the queue or moving long queries to a dedicated queue

**Remediation SQL**:
```sql
-- Check which queries were aborted and why
SELECT query, querytxt, aborted, starttime, endtime
FROM stl_query
WHERE aborted = 1 AND starttime > DATEADD(day, -7, GETDATE())
ORDER BY starttime DESC LIMIT 20;
```

**Documentation Source**:
- https://docs.aws.amazon.com/redshift/latest/dg/cm-c-wlm-query-monitoring-rules.html
- https://docs.aws.amazon.com/redshift/latest/dg/cm-c-implementing-workload-management.html

---

## WL-03: High SQA Eviction Rate

**Severity**: MEDIUM
**Category**: Workload Management
**Trigger Condition**: SQA evicted queries > 30% of SQA completed queries
**Diagnostic Queries**: D4

**Observation Template**:
> Short Query Acceleration (SQA) evicted {sqa_evicted} queries (vs {sqa_completed} completed). Many queries predicted to be short are actually taking longer than expected.

**Recommendation**:
High eviction rates indicate that many queries are being misclassified as "short". This can happen when:
- Table statistics are stale (causing optimizer misestimates)
- Query complexity varies significantly
- Consider adjusting the SQA max runtime threshold

**Documentation Source**:
- https://docs.aws.amazon.com/redshift/latest/dg/wlm-short-query-acceleration.html
- https://docs.aws.amazon.com/redshift/latest/dg/cm-c-implementing-workload-management.html

---

## WL-04: Frequent Query Monitoring Rule Violations

**Severity**: MEDIUM
**Category**: Workload Management
**Trigger Condition**: > 50 QMR violations per day
**Diagnostic Queries**: D3

**Observation Template**:
> Rule `{rule}` triggered {violation_count} times with action `{action}` in the last 7 days. Frequent violations may indicate rules are too restrictive or queries need optimization.

**Recommendation**:
Review whether QMR thresholds are appropriate:
- If queries are being logged but not hopped/aborted, consider if the threshold is useful
- If queries are being hopped frequently, the source queue may need more memory
- If queries are being aborted, they may need to be routed to a dedicated queue with higher limits

**Documentation Source**:
- https://docs.aws.amazon.com/redshift/latest/dg/cm-c-wlm-query-monitoring-rules.html
- Section: "Defining a query monitoring rule"

---

## WL-05: Concurrency Scaling Underutilized

**Severity**: LOW
**Category**: Workload Management
**Trigger Condition**: Queue wait times > 5s (QP-05 triggered) AND no concurrency scaling usage in C7 results
**Diagnostic Queries**: C6, C7

**Observation Template**:
> Queries are experiencing queue waits but concurrency scaling is not being used. Concurrency scaling can burst to additional clusters during peak demand.

**Recommendation**:
Enable concurrency scaling on the queues experiencing high wait times. This allows Redshift to automatically add transient clusters during demand spikes.

**Remediation SQL**:
```sql
-- Enable concurrency scaling via WLM parameter group
-- Set max_concurrency_scaling_clusters > 0 in parameter group
```

**Documentation Source**:
- https://docs.aws.amazon.com/redshift/latest/dg/concurrency-scaling.html
- Section: "Configuring concurrency scaling queues"

---

## WL-06: Manual WLM When Automatic WLM Available

**Severity**: LOW
**Category**: Workload Management
**Trigger Condition**: Manual WLM configuration detected (fixed concurrency values in stv_wm_service_class_config)
**Diagnostic Queries**: D1

**Observation Template**:
> Cluster is using manual WLM configuration with fixed concurrency slots. Automatic WLM dynamically adjusts memory and concurrency based on workload.

**Recommendation**:
Consider migrating to Automatic WLM (Auto WLM). It uses machine learning to dynamically manage query resources, eliminating the need to manually tune concurrency and memory allocation. Auto WLM typically provides better overall throughput.

**Documentation Source**:
- https://docs.aws.amazon.com/redshift/latest/dg/automatic-wlm.html
- https://docs.aws.amazon.com/redshift/latest/dg/cm-c-implementing-workload-management.html
- Section: "Switching WLM mode"
