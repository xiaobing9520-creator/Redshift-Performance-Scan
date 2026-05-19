# Cluster Configuration Rules

## CC-01: Single Node Cluster with Large Data

**Severity**: MEDIUM
**Category**: Cluster Configuration
**Trigger Condition**: Only 1 compute node detected AND total data size > 500GB
**Diagnostic Queries**: A1, A2

**Observation Template**:
> Cluster has a single compute node with {total_data_gb}GB of data. A single-node cluster cannot leverage Redshift's MPP (Massively Parallel Processing) architecture effectively.

**Recommendation**:
Scale to a multi-node cluster to benefit from parallel query processing across slices on multiple nodes. Each additional node adds both compute capacity and memory for query processing. Consider:
- RA3 nodes for flexible compute/storage scaling
- At minimum 2 nodes for any production workload

**Documentation Source**:
- https://docs.aws.amazon.com/redshift/latest/mgmt/working-with-clusters.html
- Section: "Clusters and nodes in Amazon Redshift"
- https://docs.aws.amazon.com/redshift/latest/mgmt/resizing-cluster.html
- https://docs.aws.amazon.com/redshift/latest/dg/c_high_level_system_architecture.html

---

## CC-02: Disk Usage Near Capacity

**Severity**: CRITICAL
**Category**: Cluster Configuration
**Trigger Condition**: Any node with pct_used > 85%
**Diagnostic Queries**: A2

**Observation Template**:
> Node {node} is at {pct_used}% disk capacity. When disk usage exceeds ~90%, Redshift may become unable to process queries that require temp space, and maintenance operations will fail.

**Recommendation**:
Take immediate action:
1. VACUUM DELETE to reclaim ghost rows
2. Drop or archive unused tables
3. Resize cluster (add nodes via elastic resize)
4. For RA3 nodes: data is in managed storage, but local cache pressure still impacts performance

**Documentation Source**:
- https://docs.aws.amazon.com/redshift/latest/mgmt/working-with-clusters.html
- https://docs.aws.amazon.com/redshift/latest/mgmt/resizing-cluster.html
- Section: "Elastic resize"

---

## CC-03: Result Caching Disabled

**Severity**: LOW
**Category**: Cluster Configuration
**Trigger Condition**: enable_result_cache_for_session = 'off' in pg_settings
**Diagnostic Queries**: A3

**Observation Template**:
> Result caching is disabled. Repeated identical queries will be re-executed instead of returning cached results. For analytical workloads with repeated queries (dashboards, BI tools), this misses significant optimization.

**Recommendation**:
Enable result caching unless the workload requires real-time data on every query execution. Result cache automatically invalidates when underlying data changes.

**Remediation SQL**:
```sql
-- Enable for current session
SET enable_result_cache_for_session = on;

-- To enable cluster-wide, modify the parameter group
-- AWS Console: Clusters → Parameter Groups → Edit → enable_result_cache_for_session = true
```

**Documentation Source**:
- https://docs.aws.amazon.com/redshift/latest/dg/c_challenges_achieving_high_performance_queries.html
- Section: "Result caching"
