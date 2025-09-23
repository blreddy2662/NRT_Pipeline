# Near Real-Time Pipeline File Config Table

## 1. Table Schema & Column Descriptions

| Column Name               | Type           | Description                                                                                                         |
|---------------------------|----------------|---------------------------------------------------------------------------------------------------------------------|
| file_id                   | BIGSERIAL PK   | Surrogate integer key for internal tracking. Unique for each source file config row.                                |
| file_key                  | VARCHAR(64)    | Logical, unique name for the file. Used by pipeline logic and dependency lists.                                     |
| source_file_name          | VARCHAR(128)   | Pattern or name for incoming files (e.g., "EGI_*.csv"). Used for matching arrivals.                                 |
| source_path               | TEXT           | Full location or pattern for file arrival (e.g., S3 bucket, FTP URI).                                               |
| region                    | VARCHAR(32)    | Region or business unit for partitioning/routing. Helps orchestrate local vs. global pipelines.                     |
| active_flag               | BOOLEAN        | If TRUE, file is processed by pipeline. If FALSE, file is ignored (maintenance/deprecation).                        |
| priority_flag             | BOOLEAN        | If TRUE, file is treated as high-priority; pipeline may expedite processing and alerting.                           |
| process_in_sequence       | BOOLEAN        | If TRUE, all files in the region/group must be processed in order; pipeline enforces strict sequencing.              |
| hold_if_prev_unprocessed  | BOOLEAN        | If TRUE, this file cannot be processed until all specified dependencies (previous files) are processed.              |
| override_dependency_flag  | BOOLEAN        | If TRUE, orchestration may bypass dependencies for this file during special runs or manual intervention.             |
| dependency_list           | JSONB          | Ordered list of dependencies (file_key or file_id) that must be processed before this file.                         |
| dependency_type           | VARCHAR(8)     | 'hard' if dependencies are mandatory (blocking), 'soft' if preferred but not required.                              |
| dependent_reports         | JSONB          | List of report IDs or names that should be refreshed when this file is successfully processed.                       |
| refresh_mode              | VARCHAR(16)    | 'immediate': refresh dependent reports as soon as file is processed; 'scheduled': batch at interval; 'deferred': wait.|
| expected_frequency        | VARCHAR(16)    | Expected arrival pattern (e.g., 'hourly', 'daily', 'ad hoc'). Used for orchestration and anomaly detection.         |
| file_format               | VARCHAR(16)    | Format of incoming file ('csv', 'parquet', 'json', etc.). Guides parsing logic.                                     |
| partition_keys            | JSONB          | List of key(s) for downstream partitioning tables. Helps manage data distribution and query efficiency.             |
| error_handling_strategy   | VARCHAR(16)    | How pipeline handles errors for this file: 'retry', 'skip', 'block', 'notify'.                                      |
| max_retries               | INT            | Maximum number of retry attempts before error is escalated or pipeline moves on.                                    |

---

## 2. DDL Statement

```sql
CREATE TABLE sc360_nrt_file_config (
  file_id BIGSERIAL PRIMARY KEY,
  file_key VARCHAR(64) NOT NULL UNIQUE,
  source_file_name VARCHAR(128) NOT NULL,
  source_path TEXT NOT NULL,
  region VARCHAR(32) NOT NULL,
  active_flag BOOLEAN NOT NULL DEFAULT TRUE,
  priority_flag BOOLEAN NOT NULL DEFAULT FALSE,
  process_in_sequence BOOLEAN NOT NULL DEFAULT FALSE,
  hold_if_prev_unprocessed BOOLEAN NOT NULL DEFAULT FALSE,
  override_dependency_flag BOOLEAN NOT NULL DEFAULT FALSE,
  dependency_list JSONB NOT NULL DEFAULT '[]',
  dependency_type VARCHAR(8) NOT NULL DEFAULT 'hard',
  dependent_reports JSONB NOT NULL DEFAULT '[]',
  refresh_mode VARCHAR(16) NOT NULL DEFAULT 'immediate',
  expected_frequency VARCHAR(16) NOT NULL DEFAULT 'hourly',
  file_format VARCHAR(16) NOT NULL,
  partition_keys JSONB NOT NULL DEFAULT '[]',
  error_handling_strategy VARCHAR(16) NOT NULL DEFAULT 'retry',
  max_retries INT NOT NULL DEFAULT 3
);
```

---

## 3. Sample Data (Tabular Output)

| file_id | file_key   | source_file_name        | source_path                    | region | active_flag | priority_flag | process_in_sequence | hold_if_prev_unprocessed | override_dependency_flag | dependency_list        | dependency_type | dependent_reports                      | refresh_mode | expected_frequency | file_format | partition_keys    | error_handling_strategy | max_retries |
|---------|------------|------------------------|-------------------------------|--------|-------------|---------------|---------------------|-------------------------|-------------------------|------------------------|-----------------|------------------------------------------|--------------|-------------------|------------|-------------------|-------------------------|-------------|
| 1       | EGI_2_0_AMS_INC        | EGI_2_0_AMS_INC_*.csv              | s3://sc360-dev-ww-nrt-bucket/LandingZone/        | WW     | true        | true          | true                | true                    | false                   |           | hard            |     | immediate    | hourly            | xlsx        | ["date_column"]    | retry                   | 3           |
| 2       | OrderBook  | OrderBook_*.parquet    | s3://data-landing/orderbook/  | WW     | true        | false         | false               | false                   | false                   | []                     | hard            | ["Report3"]                 | immediate    | hourly            | parquet    | ["date_column"]    | retry                   | 5           |
```