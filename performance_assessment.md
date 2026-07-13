As a **Senior SQL Server DBA**, when you walk into a performance assessment meeting, you should have a standard checklist of **SQL Server Instance, Database, Storage, Query, and OS-level settings** to review.

Below is a practical checklist with the exact commands to verify configurations and, where applicable, commands to change them.

# 1. Server-Level Configuration

## Check All Important SQL Server Configurations

```sql
EXEC sp_configure;
```

More readable:

```sql
SELECT name,
       value,
       value_in_use
FROM sys.configurations
ORDER BY name;
```

***

## Max Server Memory

### Check

```sql
EXEC sp_configure 'max server memory';
```

### Change

Example:

```sql
EXEC sp_configure 'show advanced options',1;
RECONFIGURE;

EXEC sp_configure 'max server memory',55000;
RECONFIGURE;
```

### Recommendation for your server

```text
Total RAM = 61 GB

Current Max Memory = 48 GB

Recommended:
52-55 GB
```

keeping enough memory for OS.

***

# 2. Parallelism Settings

## Check MAXDOP

```sql
SELECT *
FROM sys.configurations
WHERE name='max degree of parallelism';
```

or

```sql
EXEC sp_configure 'max degree of parallelism';
```

### Change

```sql
EXEC sp_configure 'max degree of parallelism',4;
RECONFIGURE;
```

***

## Check Cost Threshold for Parallelism

```sql
EXEC sp_configure 'cost threshold for parallelism';
```

Current:

```text
5
```

Recommended starting point:

```text
40-50
```

Change:

```sql
EXEC sp_configure 'cost threshold for parallelism',50;
RECONFIGURE;
```

***

# 3. TempDB Configuration

## Check TempDB Files

```sql
SELECT
name,
physical_name,
size/128 AS SizeMB,
growth,
type_desc
FROM sys.master_files
WHERE database_id=2;
```

***

## Check Number of Files

```sql
USE tempdb;

SELECT name
FROM sys.database_files;
```

For 12 CPUs recommend:

```text
8 TempDB data files
1 TempDB log file
```

***

## Check VLF Count

SQL 2019:

```sql
DBCC LOGINFO('tempdb');
```

or

```sql
DBCC LOGINFO;
```

If count is very large (your case \~2600):

```text
Needs correction.
```

***

## Increase Log Growth

```sql
ALTER DATABASE tempdb
MODIFY FILE
(
NAME=tempdev,
FILEGROWTH=512MB
);
```

For log:

```sql
ALTER DATABASE tempdb
MODIFY FILE
(
NAME=templog,
FILEGROWTH=512MB
);
```

***

# 4. Instant File Initialization

IFI is an OS-level privilege.

### Check

```sql
SELECT serverproperty('IsInstantFileInitializationEnabled');
```

Result:

```text
1 = Enabled
0 = Disabled
```

### Enable

Windows Server:

```text
Perform Volume Maintenance Tasks
```

Grant this right to SQL Server Service Account.

***

# 5. Statistics

## Check Auto Update Statistics

```sql
SELECT
name,
is_auto_create_stats_on,
is_auto_update_stats_on
FROM sys.databases;
```

***

## Last Statistics Update

```sql
SELECT
OBJECT_NAME(object_id) TableName,
name StatsName,
STATS_DATE(object_id,stats_id) LastUpdated
FROM sys.stats
ORDER BY LastUpdated;
```

***

## Update Statistics

```sql
EXEC sp_updatestats;
```

Large tables:

```sql
UPDATE STATISTICS dbo.INSATDP
WITH FULLSCAN;
```

***

# 6. Missing Index Analysis

## Check Missing Indexes

```sql
SELECT
DB_NAME(database_id) DatabaseName,
avg_user_impact,
user_seeks,
statement
FROM sys.dm_db_missing_index_details
MID
JOIN sys.dm_db_missing_index_groups MIG
ON MID.index_handle=MIG.index_handle
JOIN sys.dm_db_missing_index_group_stats MGS
ON MIG.index_group_handle=MGS.group_handle
ORDER BY avg_user_impact DESC;
```

***

# 7. Unused Index Analysis

## Check Unused Indexes

```sql
SELECT
OBJECT_NAME(i.object_id) TableName,
i.name IndexName,
s.user_seeks,
s.user_scans,
s.user_lookups,
s.user_updates
FROM sys.indexes i
LEFT JOIN sys.dm_db_index_usage_stats s
ON i.object_id=s.object_id
AND i.index_id=s.index_id
WHERE i.index_id > 0;
```

Review carefully before dropping.

***

# 8. Identify Expensive Queries

## Top CPU Queries

```sql
SELECT TOP 20
total_worker_time/execution_count AvgCPU,
execution_count,
total_worker_time,
SUBSTRING(st.text,
(qs.statement_start_offset/2)+1,
((CASE qs.statement_end_offset
WHEN -1 THEN DATALENGTH(st.text)
ELSE qs.statement_end_offset END
- qs.statement_start_offset)/2)+1)
AS QueryText
FROM sys.dm_exec_query_stats qs
CROSS APPLY sys.dm_exec_sql_text(qs.sql_handle) st
ORDER BY AvgCPU DESC;
```

***

## Top Duration Queries

```sql
SELECT TOP 20
total_elapsed_time/execution_count AvgDuration,
execution_count,
st.text
FROM sys.dm_exec_query_stats qs
CROSS APPLY sys.dm_exec_sql_text(qs.sql_handle) st
ORDER BY AvgDuration DESC;
```

***

# 9. Wait Statistics

Most important performance area.

## Check Waits

```sql
SELECT TOP 20
wait_type,
wait_time_ms/1000 WaitSeconds,
signal_wait_time_ms/1000 SignalSeconds
FROM sys.dm_os_wait_stats
ORDER BY wait_time_ms DESC;
```

Interpretation:

| Wait                  | Meaning                         |
| --------------------- | ------------------------------- |
| PAGEIOLATCH           | Storage issue                   |
| CXPACKET              | Parallelism issue               |
| CXCONSUMER            | Parallel query workload         |
| WRITELOG              | Transaction log bottleneck      |
| ASYNC\_NETWORK\_IO    | Application slow consuming data |
| SOS\_SCHEDULER\_YIELD | CPU pressure                    |

***

# 10. Blocking Analysis

## Current Blocking

```sql
SELECT
session_id,
blocking_session_id,
wait_type,
wait_time
FROM sys.dm_exec_requests
WHERE blocking_session_id <> 0;
```

***

# 11. Database Growth Analysis

## Database Size

```sql
EXEC sp_spaceused;
```

Per file:

```sql
SELECT
name,
size/128 SizeMB
FROM sys.database_files;
```

***

# 12. Fragmentation

## Check Fragmentation

```sql
SELECT
OBJECT_NAME(object_id) TableName,
index_id,
avg_fragmentation_in_percent
FROM sys.dm_db_index_physical_stats
(
DB_ID(),
NULL,
NULL,
NULL,
'SAMPLED'
);
```

### Reorganize

```sql
ALTER INDEX IX_Test
ON dbo.TableName
REORGANIZE;
```

### Rebuild

```sql
ALTER INDEX IX_Test
ON dbo.TableName
REBUILD;
```

***

# 13. Disk I/O Bottlenecks

## Check File-Level I/O

```sql
SELECT
DB_NAME(vfs.database_id) DatabaseName,
mf.name FileName,
num_of_reads,
io_stall_read_ms,
num_of_writes,
io_stall_write_ms
FROM sys.dm_io_virtual_file_stats(NULL,NULL) vfs
JOIN sys.master_files mf
ON vfs.database_id=mf.database_id
AND vfs.file_id=mf.file_id
ORDER BY io_stall_read_ms DESC;
```

This is the exact DMV that would validate the high D:\data I/O issue mentioned in your assessment.

***

# 14. Memory Pressure

## Check Memory Usage

```sql
SELECT
physical_memory_kb/1024 PhysicalMemoryMB,
available_physical_memory_kb/1024 AvailableMemoryMB
FROM sys.dm_os_sys_memory;
```

***

## PLE (Page Life Expectancy)

```sql
SELECT cntr_value
FROM sys.dm_os_performance_counters
WHERE counter_name='Page life expectancy';
```

Low PLE generally indicates memory pressure.

***

# 15. Resource Governor

## Check Resource Governor

```sql
SELECT
name,
is_enabled
FROM sys.resource_governor_resource_pools;
```

Enable:

```sql
ALTER RESOURCE GOVERNOR RECONFIGURE;
```

***

# Most Likely Improvements in Your Environment

Based on the assessment, I would prioritize:

### Immediate (0-2 Weeks)

1. Increase Cost Threshold from 5 → 50
2. Fix TempDB growth and VLF issue
3. Enable IFI
4. Update statistics
5. Create missing indexes
6. Remove unused indexes after validation

### Short Term (2-6 Weeks)

1. Tune top resource-consuming queries
2. Correct index design on INDLYIP
3. Create summary table for INSATDP
4. Configure Resource Governor

### Long Term

1. Partition 377M and 2.1B row tables
2. Move reporting workload to separate reporting server
3. Consider CPU/storage upgrades if bottlenecks remain

If the client challenges you technically, focus on **Wait Statistics, Query Plans, Missing Indexes, TempDB, Parallelism, and I/O latency**—those are the areas most directly tied to the problems described in your assessment.
