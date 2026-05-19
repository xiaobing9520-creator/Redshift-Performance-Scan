# Table Design Rules

## TD-01: Missing Distribution Key on Large Table

**Severity**: HIGH
**Category**: Table Design
**Trigger Condition**: Table size > 500MB with EVEN distribution style (diststyle = 'EVEN')
**Diagnostic Queries**: B1

**Observation Template**:
> Table `{schema}.{table}` ({size_mb} MB, {tbl_rows} rows) uses EVEN distribution. Large tables with EVEN distribution force full redistribution during joins.

**Recommendation**:
Choose a distribution key based on the most frequent join column. Use KEY distribution to co-locate rows that are commonly joined together. If the table is frequently joined with other large tables, distribute both on the join column.

**Remediation SQL**:
```sql
ALTER TABLE {schema}.{table} ALTER DISTSTYLE KEY DISTKEY ({join_column});
```

**Documentation Source**:
- https://docs.aws.amazon.com/redshift/latest/dg/c_best-practices-best-dist-key.html
- Section: "Choose the best distribution style"

---

## TD-02: High Distribution Skew

**Severity**: HIGH
**Category**: Table Design
**Trigger Condition**: skew_rows > 4.0 AND size > 100MB
**Diagnostic Queries**: B6

**Observation Template**:
> Table `{schema}.{table}` has row skew ratio of {skew_rows}x ({size_mb} MB). Data is unevenly distributed across slices, causing some slices to do disproportionately more work.

**Recommendation**:
The current distribution key has low cardinality or high value skew. Consider changing to a column with higher cardinality and more uniform distribution, or switch to EVEN distribution if no good key candidate exists.

**Remediation SQL**:
```sql
-- Check current distribution key cardinality
SELECT COUNT(DISTINCT {distkey_col}) AS distinct_values, COUNT(*) AS total_rows
FROM {schema}.{table};

-- Change distribution key
ALTER TABLE {schema}.{table} ALTER DISTSTYLE KEY DISTKEY ({new_column});
```

**Documentation Source**:
- https://docs.aws.amazon.com/redshift/latest/dg/c_best-practices-best-dist-key.html
- https://docs.aws.amazon.com/redshift/latest/dg/c_choosing_dist_sort.html

---

## TD-03: Missing Sort Key on Large Table

**Severity**: MEDIUM
**Category**: Table Design
**Trigger Condition**: Table size > 100MB with no sort key (sortkey1 IS NULL)
**Diagnostic Queries**: B2

**Observation Template**:
> Table `{schema}.{table}` ({size_mb} MB, {tbl_rows} rows) has no sort key defined. Without a sort key, Redshift cannot skip blocks during range-restricted queries.

**Recommendation**:
Add a sort key based on how the table is most commonly queried:
- Timestamp/date column if used in range filters (most common)
- Column used in equality/range predicates in WHERE clauses
- Join column if the table is frequently joined

**Remediation SQL**:
```sql
ALTER TABLE {schema}.{table} ALTER SORTKEY ({recommended_column});
```

**Documentation Source**:
- https://docs.aws.amazon.com/redshift/latest/dg/c_best-practices-sort-key.html
- Section: "Choose the best sort key"

---

## TD-04: High Unsorted Percentage

**Severity**: HIGH
**Category**: Table Design
**Trigger Condition**: unsorted > 20% AND size > 50MB
**Diagnostic Queries**: B3

**Observation Template**:
> Table `{schema}.{table}` is {unsorted}% unsorted ({size_mb} MB). Unsorted data forces Redshift to scan more blocks than necessary, degrading query performance.

**Recommendation**:
Run VACUUM SORT on the affected tables. If tables are consistently unsorted, ensure data is being loaded in sort key order, or enable automatic vacuum sort.

**Remediation SQL**:
```sql
VACUUM SORT ONLY {schema}.{table};
-- Or for full vacuum (sort + delete):
VACUUM FULL {schema}.{table};
```

**Documentation Source**:
- https://docs.aws.amazon.com/redshift/latest/dg/t_Reclaiming_storage_space202.html
- https://docs.aws.amazon.com/redshift/latest/dg/c_best-practices-sort-key-order.html
- Section: "Load data in sort key order"

---

## TD-05: Missing Compression/Encoding on Non-Sortkey Columns

**Severity**: MEDIUM
**Category**: Table Design
**Trigger Condition**: Table has >3 non-sortkey columns with encoding='none'
**Diagnostic Queries**: B5

**Observation Template**:
> Table `{tablename}` has {raw_columns} columns without compression encoding. Uncompressed columns waste storage and increase I/O during queries.

**Recommendation**:
Apply compression encoding. Use ENCODE AUTO on the table to let Redshift automatically choose optimal encoding, or use the ANALYZE COMPRESSION command to get encoding recommendations.

**Remediation SQL**:
```sql
-- Enable automatic encoding for the table
ALTER TABLE {schema}.{tablename} ALTER ENCODE AUTO;

-- Or get encoding recommendations
ANALYZE COMPRESSION {schema}.{tablename};
```

**Documentation Source**:
- https://docs.aws.amazon.com/redshift/latest/dg/c_best-practices-use-auto-compression.html
- Section: "Let COPY choose compression encodings"

---

## TD-06: VARCHAR Over-Allocation

**Severity**: LOW
**Category**: Table Design
**Trigger Condition**: VARCHAR declared size > 256 characters
**Diagnostic Queries**: B8

**Observation Template**:
> Column `{tablename}.{column}` is declared as VARCHAR({declared_size}). Over-sized VARCHAR columns waste memory during query processing (sorts, aggregations) even if actual data is shorter.

**Recommendation**:
Reduce VARCHAR size to the actual maximum length needed plus a small buffer. Redshift allocates memory based on declared size, not actual data size.

**Remediation SQL**:
```sql
-- Check actual max length
SELECT MAX(LEN({column})) FROM {schema}.{tablename};

-- Resize column (requires table recreate or ALTER)
ALTER TABLE {schema}.{tablename} ALTER COLUMN {column} TYPE VARCHAR({actual_max + buffer});
```

**Documentation Source**:
- https://docs.aws.amazon.com/redshift/latest/dg/c_best-practices-smallest-column-size.html
- Section: "Use the smallest possible column size"

---

## TD-07: ALL Distribution on Large Table

**Severity**: HIGH
**Category**: Table Design
**Trigger Condition**: Table size > 1GB with diststyle='ALL'
**Diagnostic Queries**: B1

**Observation Template**:
> Table `{schema}.{table}` ({size_mb} MB) uses ALL distribution. Large ALL-distributed tables are copied to every node, wasting storage and slowing COPY/INSERT operations.

**Recommendation**:
ALL distribution is designed for small dimension tables (< 3-5 million rows). For large tables, switch to KEY or EVEN distribution.

**Remediation SQL**:
```sql
ALTER TABLE {schema}.{table} ALTER DISTSTYLE KEY DISTKEY ({column});
-- Or:
ALTER TABLE {schema}.{table} ALTER DISTSTYLE EVEN;
```

**Documentation Source**:
- https://docs.aws.amazon.com/redshift/latest/dg/c_best-practices-best-dist-key.html
- https://docs.aws.amazon.com/redshift/latest/dg/c_choosing_dist_sort.html

---

## TD-08: Ignored Automatic Table Optimization Recommendations

**Severity**: MEDIUM
**Category**: Table Design
**Trigger Condition**: svv_alter_table_recommendations has pending entries
**Diagnostic Queries**: B7

**Observation Template**:
> {count} automatic optimization recommendations are pending for tables in this cluster. Redshift's ML-based optimizer has identified potential improvements.

**Recommendation**:
Review and apply the recommendations from SVV_ALTER_TABLE_RECOMMENDATIONS. These are generated by Redshift's machine learning based on observed query patterns. Consider enabling automatic table optimization if it's disabled.

**Remediation SQL**:
```sql
-- Apply a specific recommendation (from the 'command' column):
{command_from_recommendation}

-- Enable automatic table optimization
ALTER TABLE {schema}.{table} ALTER SORTKEY AUTO;
ALTER TABLE {schema}.{table} ALTER DISTSTYLE AUTO;
```

**Documentation Source**:
- https://docs.aws.amazon.com/redshift/latest/dg/c_autonomics.html
- https://docs.aws.amazon.com/redshift/latest/dg/c_ato-enabling-disabling-monitoring.html
- Section: "Enabling automatic table optimization"

---

## TD-09: Oversized CHAR Columns

**Severity**: LOW
**Category**: Table Design
**Trigger Condition**: CHAR columns > 10 characters declared size
**Diagnostic Queries**: B4

**Observation Template**:
> Table `{tablename}` has CHAR({size}) columns that could be VARCHAR. CHAR pads values with spaces to the declared length, wasting storage.

**Recommendation**:
Use VARCHAR instead of CHAR for variable-length strings. CHAR is only appropriate for truly fixed-length codes (e.g., 2-char country codes, 3-char currency codes).

**Documentation Source**:
- https://docs.aws.amazon.com/redshift/latest/dg/c_best-practices-smallest-column-size.html

---

## TD-10: Missing Primary Key/Foreign Key Constraints

**Severity**: LOW
**Category**: Table Design
**Trigger Condition**: Large tables (>1M rows) without PK/FK constraints defined
**Diagnostic Queries**: B1 (cross-reference with pg_constraint)

**Observation Template**:
> Several large tables lack primary key or foreign key constraints. While Redshift doesn't enforce them, the query optimizer uses these hints to generate more efficient plans.

**Recommendation**:
Define PK and FK constraints on your tables. They are informational only (not enforced) but help the optimizer eliminate unnecessary joins and generate better execution plans.

**Remediation SQL**:
```sql
ALTER TABLE {schema}.{table} ADD CONSTRAINT {table}_pk PRIMARY KEY ({column});
ALTER TABLE {schema}.{fact_table} ADD CONSTRAINT {fk_name}
  FOREIGN KEY ({column}) REFERENCES {schema}.{dim_table}({column});
```

**Documentation Source**:
- https://docs.aws.amazon.com/redshift/latest/dg/c_best-practices-defining-constraints.html
- Section: "Define primary key and foreign key constraints"

---

## TD-11: TIMESTAMP Used Where DATE Suffices

**Severity**: LOW
**Category**: Table Design
**Trigger Condition**: TIMESTAMP columns on date-only data (no time component variation)
**Diagnostic Queries**: B4

**Observation Template**:
> Table `{tablename}` has TIMESTAMP columns that may only store date-level granularity. TIMESTAMP uses 8 bytes vs DATE's 4 bytes.

**Recommendation**:
If you only need date-level precision, use the DATE data type instead of TIMESTAMP. This halves the storage for that column and improves sort/filter performance.

**Remediation SQL**:
```sql
-- Verify no time component is used
SELECT MIN({col}::time), MAX({col}::time) FROM {schema}.{table};
```

**Documentation Source**:
- https://docs.aws.amazon.com/redshift/latest/dg/c_best-practices-timestamp-date-columns.html
- Section: "Use date/time data types for date columns"

---

## TD-12: Interleaved Sort Key with Infrequent VACUUM REINDEX

**Severity**: MEDIUM
**Category**: Table Design
**Trigger Condition**: Tables with interleaved sort keys AND high unsorted percentage (>10%)
**Diagnostic Queries**: B1, B3

**Observation Template**:
> Table `{schema}.{table}` uses interleaved sort key but is {unsorted}% unsorted. Interleaved sort keys require periodic VACUUM REINDEX to maintain performance.

**Recommendation**:
Run VACUUM REINDEX on tables with interleaved sort keys regularly, especially after significant data loads. Consider switching to compound sort key if queries predominantly filter on a single leading column.

**Remediation SQL**:
```sql
VACUUM REINDEX {schema}.{table};
```

**Documentation Source**:
- https://docs.aws.amazon.com/redshift/latest/dg/t_Sorting_data-interleaved.html
- https://docs.aws.amazon.com/redshift/latest/dg/r_VACUUM_command.html

---

## TD-13: Block-Level Distribution Skew

**Severity**: HIGH
**Category**: Table Design
**Trigger Condition**: ratio_skew_across_slices > 100% (blocks unevenly distributed)
**Diagnostic Queries**: H2

**Observation Template**:
> Table `{schemaname}.{tablename}` has {ratio_skew_across_slices}% block-level skew across slices. Only {pct_slices_populated}% of slices contain data. This indicates severe distribution imbalance at the storage layer.

**Recommendation**:
Unlike row-level skew (TD-02), block-level skew means some slices store disproportionately more 1MB blocks. Choose a distribution key with higher cardinality, or switch to EVEN distribution for better parallel scan performance.

**Remediation SQL**:
```sql
ALTER TABLE {schema}.{tablename} ALTER DISTSTYLE EVEN;
-- Or choose a better distribution key:
ALTER TABLE {schema}.{tablename} ALTER DISTSTYLE KEY DISTKEY ({high_cardinality_column});
```

**Documentation Source**:
- https://docs.aws.amazon.com/redshift/latest/dg/c_best-practices-best-dist-key.html
- https://github.com/awslabs/amazon-redshift-utils/blob/master/src/AdminScripts/table_inspector.sql

---

## TD-14: Distribution Key Mismatch in INSERT...SELECT Pipelines

**Severity**: MEDIUM
**Category**: Table Design
**Trigger Condition**: Source and target tables in INSERT...SELECT have different distribution keys
**Diagnostic Queries**: H8

**Observation Template**:
> INSERT...SELECT from `{source}` (distkey: {source_dk}) into `{target}` (distkey: {target_dk}) causes data redistribution. Aligning distribution keys eliminates this overhead.

**Recommendation**:
When data flows regularly between tables via INSERT...SELECT, align the distribution keys of source and target tables to avoid redistribution during the write operation.

**Remediation SQL**:
```sql
-- Align target table's distribution key with source
ALTER TABLE {target} ALTER DISTSTYLE KEY DISTKEY ({source_dk});
```

**Documentation Source**:
- https://docs.aws.amazon.com/redshift/latest/dg/c_best-practices-best-dist-key.html
- https://github.com/awslabs/amazon-redshift-utils/blob/master/src/AdminScripts/insert_into_table_dk_mismatch.sql

---

## TD-15: Unscanned Tables Wasting Storage

**Severity**: LOW
**Category**: Table Design
**Trigger Condition**: Tables > 10MB that have never been scanned (0 queries in STL_SCAN history)
**Diagnostic Queries**: H7

**Observation Template**:
> {count} tables ({total_mb} MB total) have never been scanned by any query. These may be obsolete, staging leftovers, or incorrectly retained data consuming storage.

**Recommendation**:
Review unscanned tables for potential removal. These could be:
- Old staging tables from ETL that were never dropped
- Tables from deprecated features
- Test/dev tables in production

Archive to S3 via UNLOAD if unsure, then DROP.

**Remediation SQL**:
```sql
-- Archive before dropping
UNLOAD ('SELECT * FROM {schema}.{table}')
TO 's3://bucket/archive/{schema}/{table}/'
IAM_ROLE 'arn:aws:iam::account:role/RedshiftRole' PARQUET;

DROP TABLE {schema}.{table};
```

**Documentation Source**:
- https://docs.aws.amazon.com/redshift/latest/mgmt/working-with-clusters.html
- https://github.com/awslabs/amazon-redshift-utils/blob/master/src/AdminScripts/unscanned_table_summary.sql

---

## TD-16: Sort Key Not Aligned with Predicate Columns

**Severity**: MEDIUM
**Category**: Table Design
**Trigger Condition**: Table has predicate columns (used in WHERE) that are not the sort key
**Diagnostic Queries**: H6

**Observation Template**:
> Table `{schema}.{table}` is frequently filtered on column `{col_name}` (first used: {first_predicate_use}), but this column is not the sort key. Zone map filtering cannot be leveraged.

**Recommendation**:
Consider changing the sort key to match the most commonly used predicate column. Redshift tracks which columns are used as predicates in pg_statistic — the sort key should align with the most selective filter patterns.

**Remediation SQL**:
```sql
ALTER TABLE {schema}.{table} ALTER SORTKEY ({predicate_column});
```

**Documentation Source**:
- https://docs.aws.amazon.com/redshift/latest/dg/c_best-practices-sort-key.html
- https://github.com/awslabs/amazon-redshift-utils/blob/master/src/AdminScripts/predicate_columns.sql

---

## TD-17: Table Fragmentation (Excess Blocks)

**Severity**: MEDIUM
**Category**: Table Design
**Trigger Condition**: est_space_gain_blocks > 100 (significant reclaimable blocks from fragmentation)
**Diagnostic Queries**: H9

**Observation Template**:
> Table `{tablename}` has ~{est_space_gain_blocks} excess blocks due to fragmentation. This occurs when concurrent writes overlap with VACUUM operations.

**Recommendation**:
Defragment the table using a deep copy (CREATE TABLE AS SELECT) or VACUUM with BOOST option during a maintenance window. Fragmentation increases I/O during scans.

**Remediation SQL**:
```sql
-- Option 1: VACUUM with BOOST (uses all available resources)
VACUUM FULL {schema}.{tablename} BOOST;

-- Option 2: Deep copy (complete defragmentation)
CREATE TABLE {schema}.{tablename}_new AS SELECT * FROM {schema}.{tablename};
DROP TABLE {schema}.{tablename};
ALTER TABLE {schema}.{tablename}_new RENAME TO {tablename};
```

**Documentation Source**:
- https://docs.aws.amazon.com/redshift/latest/dg/r_VACUUM_command.html
- https://github.com/awslabs/amazon-redshift-utils/blob/master/src/AdminViews/v_fragmentation_info.sql
