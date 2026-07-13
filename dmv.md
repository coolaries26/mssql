# Dynamic Management Views (DMVs) 
are built-in system views in Microsoft SQL Server that return internal state, real-time health, and performance data. Database Administrators (DBAs) and developers use them to troubleshoot performance bottlenecks, monitor active user connections, diagnose high CPU or memory use, and find missing indexes.
## Key Rules of DMVs
**Schema:** 
All DMVs live in the sys schema and their names begin with dm_ (e.g., sys.dm_exec_requests).**Naming Constraint:** 
You must use at least two-part naming (sys.dm_name) to query them.
**Production Caution:**
 Never use SELECT * in production scripts because Microsoft occasionally adds new columns to DMVs, which can break application code.
**DMV vs DMF:** DMVs behave like normal tables or views. Dynamic Management Functions (DMFs) look similar but require you to pass input parameters via CROSS APPLY (e.g., passing a query handle to get the query text).
---
# Practical DMV Examples
1. See What Queries Are Running Right Now
This query identifies currently running requests, how long they have been executing, whether they are blocked, and the exact SQL code being run. It joins a DMV with a DMF (sys.dm_exec_sql_text).
```sql
SELECT 
    r.session_id,
    r.status,
    r.cpu_time,
    r.total_elapsed_time,
    r.blocking_session_id,
    t.text AS query_text
FROM sys.dm_exec_requests r
CROSS APPLY sys.dm_exec_sql_text(r.sql_handle) t
WHERE r.session_id > 50; -- Filters out background system processes
```

2. Find the Most Resource-Intensive Queries (Top 5 by CPU)This statement queries cached execution statistics to find which historical queries have consumed the most CPU processing power since the SQL Server instance last restarted.
```sql
SELECT TOP 5
    st.text AS query_text,
    qs.execution_count,
    qs.total_worker_time AS total_cpu_time,
    qs.total_worker_time / qs.execution_count AS avg_cpu_time,
    qs.total_elapsed_time / qs.execution_count AS avg_execution_duration
FROM sys.dm_exec_query_stats qs
CROSS APPLY sys.dm_exec_sql_text(qs.sql_handle) st
ORDER BY qs.total_worker_time DESC;
```
3. Monitor Active User Connections and SessionsUse this to see who is logged onto your SQL Server instance, what application they are connecting from, and their client IP address.
```sql
SELECT 
    s.session_id,
    s.login_name,
    s.host_name,
    s.program_name,
    c.client_net_address,
    c.connect_time
FROM sys.dm_exec_sessions s
JOIN sys.dm_exec_connections c 
    ON s.session_id = c.session_id
WHERE s.is_user_process = 1; -- Shows user connections only
```

4. Find Top System Bottlenecks (Wait Statistics)SQL Server tracks exactly why queries are waiting (e.g., waiting for storage disks, waiting for memory, or network lag). This query filters out trivial system background waits to show you real performance roadblocks.
```sql
SELECT TOP 10
    wait_type,
    waiting_tasks_count,
    wait_time_ms,
    max_wait_time_ms,
    signal_wait_time_ms
FROM sys.dm_os_wait_stats
WHERE wait_type NOT IN (
    'CLR_SEMAPHORE','LAZYWRITER_SLEEP','RESOURCE_QUEUE','SLEEP_TASK',
    'SLEEP_SYSTEMTASK','SQLTRACE_BUFFER_FLUSH','WAITFOR','LOGMGR_QUEUE',
    'CHECKPOINT_QUEUE','REQUEST_FOR_DEADLOCK_SEARCH','XE_TIMER_EVENT'
)
ORDER BY wait_time_ms DESC;
```

5. Find Missing Indexes Recommended by SQL ServerThe query optimizer logs when a query would have executed faster if a specific index existed. You can extract these recommendations to optimize database index design.
```sql
SELECT 
    id.statement AS table_name,
    id.equality_columns,
    id.inequality_columns,
    id.included_columns,
    gs.unique_compiles,
    gs.user_seeks,
    gs.user_scans
FROM sys.dm_db_missing_index_details id
JOIN sys.dm_db_missing_index_groups ig 
    ON id.index_handle = ig.index_handle
JOIN sys.dm_db_missing_index_group_stats gs 
    ON ig.group_handle = gs.group_handle
ORDER BY gs.user_seeks DESC;
```

**Required Permissions**
To execute DMV queries, your user database account requires elevated server permissions.
- **SQL Server 2019 and older:**
     Requires VIEW SERVER STATE (for server-level views) or VIEW DATABASE STATE (for database-level views)
- **SQL Server 2022 and newer:**
     Microsoft refined security permissions to use VIEW SERVER PERFORMANCE STATE or VIEW DATABASE PERFORMANCE STATE.
```sql
     -- Example: Granting permission to a DBA user
GRANT VIEW SERVER STATE TO [YourUsername];
```

