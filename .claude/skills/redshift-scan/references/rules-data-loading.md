# Data Loading Rules

## DL-01: COPY File Count Less Than Slice Count

**Severity**: MEDIUM
**Category**: Data Loading
**Trigger Condition**: Number of input files < number of slices in the cluster
**Diagnostic Queries**: F3

**Observation Template**:
> COPY operation (query {query}) loaded from {num_files} files but the cluster has {num_slices} slices. When file count < slice count, some slices sit idle during the load.

**Recommendation**:
Split input data into at least as many files as slices in your cluster (ideally a multiple of the slice count). This ensures all slices participate in parallel loading, maximizing throughput.

**Remediation**:
- Split large files before loading: use `split` command or partition in your ETL pipeline
- Use a manifest file with evenly-sized splits
- Compress files with gzip/lzo for additional performance

**Documentation Source**:
- https://docs.aws.amazon.com/redshift/latest/dg/c_loading-data-best-practices.html
- Section: "Split your load data into multiple files"
- https://docs.aws.amazon.com/redshift/latest/dg/c_best-practices-use-multiple-files.html

---

## DL-02: Frequent Load Errors

**Severity**: HIGH
**Category**: Data Loading
**Trigger Condition**: > 100 load errors per day in stl_load_errors
**Diagnostic Queries**: F2

**Observation Template**:
> {error_count} load errors in the last 7 days. Most common: `{err_reason}` ({count} occurrences). Load errors indicate data quality issues or schema mismatches.

**Recommendation**:
Investigate and fix the root cause of load errors:
- Data type mismatches: fix source data or adjust target column types
- Encoding issues: use ACCEPTINVCHARS or fix source encoding
- Delimiter issues: verify COPY command delimiter matches file format
- NULL handling: use BLANKSASNULL or EMPTYASNULL as appropriate

**Remediation SQL**:
```sql
-- Get details on recent errors
SELECT query, filename, line_number, colname, type, err_reason,
       SUBSTRING(raw_line, 1, 200) AS sample_data
FROM stl_load_errors
WHERE starttime > DATEADD(day, -1, GETDATE())
ORDER BY starttime DESC
LIMIT 20;
```

**Documentation Source**:
- https://docs.aws.amazon.com/redshift/latest/dg/c_loading-data-best-practices.html
- https://docs.aws.amazon.com/redshift/latest/dg/r_STL_LOAD_ERRORS.html

---

## DL-03: Single Large File Load (No Parallelism)

**Severity**: MEDIUM
**Category**: Data Loading
**Trigger Condition**: COPY from single file with > 10 million rows
**Diagnostic Queries**: F1, F3

**Observation Template**:
> COPY operation loaded {total_rows} rows from a single file ({file_path}). Single-file loads cannot be parallelized across slices, severely limiting throughput.

**Recommendation**:
Split large files into multiple smaller files before loading. The optimal number of files is a multiple of the number of slices in your cluster. Each file should be approximately the same size (100MB-1GB compressed).

**Remediation**:
```bash
# Split a large file into multiple parts (example for CSV)
split -l 1000000 large_file.csv part_
gzip part_*

# Or use S3 multipart upload with prefix-based manifest
```

**Documentation Source**:
- https://docs.aws.amazon.com/redshift/latest/dg/c_loading-data-best-practices.html
- Section: "Split your load data into multiple files"
- Section: "Use a single COPY command to load from multiple files"

---

## DL-04: Low COPY Throughput (S3 Transfer Rate)

**Severity**: MEDIUM
**Category**: Data Loading
**Trigger Condition**: COPY throughput < 10 MB/s for loads > 100MB
**Diagnostic Queries**: H4

**Observation Template**:
> COPY operation (query {query}) transferred {size_mb}MB in {time_seconds}s ({mb_per_s} MB/s) from {n_files} files. Transfer rates below 10 MB/s indicate suboptimal COPY configuration.

**Recommendation**:
Improve COPY throughput:
1. Compress files (GZIP, LZO, ZSTD) to reduce transfer volume
2. Split into more files to parallel-load across slices
3. Use columnar format (Parquet) for better compression
4. Ensure S3 bucket is in the same region as the cluster
5. Use manifest files for controlled parallel loads

**Documentation Source**:
- https://docs.aws.amazon.com/redshift/latest/dg/c_loading-data-best-practices.html
- Section: "Compress your data files" and "Split your load data"
- https://github.com/awslabs/amazon-redshift-utils/blob/master/src/AdminScripts/copy_performance.sql
