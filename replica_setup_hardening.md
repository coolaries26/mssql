Given your background (**strong DB2 DBA, but comparatively new to SQL Server administration**), I would recommend approaching the replica/reporting server setup using a **DB2 HADR mindset**:

* Primary Server = Production
* Secondary Server = Reporting Replica
* Transaction Log Shipping = Similar to DB2 Log Shipping/HADR replay
* Read-only reporting workload offloaded to secondary
* Production remains dedicated to CDC refresh, application transactions, and ETL processes

For your client discussion, below is a practical **SQL Server Reporting Replica Build & Hardening Playbook**.

***

# SQL Server Reporting Replica Setup Playbook

## Objective

Build a dedicated reporting server to:

* Offload PowerBI reports
* Offload ad-hoc reporting
* Reduce production I/O contention
* Improve CDC performance
* Improve report response time

***

# Phase 1 - Infrastructure Build

## Server Sizing

Current Production

```text
CPU          : 12 Core
Memory       : 64GB
DB Size      : ~1TB
Growth Trend : High
```

Recommended Reporting Server

```text
CPU          : 16 Core Minimum
Memory       : 96GB or 128GB
Storage      : SSD/NVME
OS           : Windows Server 2022
SQL Version  : SQL Server 2019 Enterprise / 2022 Enterprise
```

Reason:

Reporting workload is read intensive.

***

# Disk Layout

Avoid single drive architecture.

Recommended:

```text
C: OS

D: Data Files(.mdf/.ndf)

E: Log Files(.ldf)

F: TempDB

G: Backup
```

***

# NTFS Allocation Size

Format using:

```text
64 KB Allocation Unit Size
```

Verify:

```powershell
fsutil fsinfo ntfsinfo d:
```

***

# Windows Hardening

## Patch Level

Install latest:

```text
Windows Updates
.NET Updates
SQL Server CU
```

***

## Power Plan

Current default may be:

```text
Balanced
```

Change to:

```text
High Performance
```

Verify:

```powershell
powercfg /L
```

***

## Antivirus Exclusions

Exclude:

```text
MDF
NDF
LDF
BAK
TRN
TempDB
SQL Binn Folder
```

Example:

```text
D:\SQLData
E:\SQLLogs
F:\TempDB
```

***

# SQL Server Installation

## Service Accounts

Do not use:

```text
LocalSystem
Administrator
```

Use dedicated domain account.

Example:

```text
svc_SQLProd
svc_SQLAgent
```

***

## Enable Instant File Initialization

Windows Policy:

```text
Perform Volume Maintenance Tasks
```

Grant to SQL service account.

Verify:

```sql
SELECT SERVERPROPERTY('IsInstantFileInitializationEnabled');
```

Expected:

```text
1
```

***

# SQL Server Post Installation Hardening

***

## Memory Configuration

Server Memory:

```text
128 GB
```

Recommended:

```text
Max Memory = 118 GB
```

Leave memory for:

```text
OS
Monitoring
Backup Tools
AV
```

Configure:

```sql
EXEC sp_configure 'max server memory';
```

***

## MAXDOP

For 16 core server:

```text
MAXDOP = 8
```

Configure:

```sql
EXEC sp_configure 'max degree of parallelism',8;
RECONFIGURE;
```

***

## Cost Threshold For Parallelism

Default:

```text
5
```

Recommended:

```text
50
```

Configure:

```sql
EXEC sp_configure 'cost threshold for parallelism',50;
RECONFIGURE;
```

***

## Backup Compression

Enable:

```sql
EXEC sp_configure 'backup compression default',1;
RECONFIGURE;
```

***

## Optimize For Ad Hoc Workloads

Enable:

```sql
EXEC sp_configure 'optimize for ad hoc workloads',1;
RECONFIGURE;
```

Very useful for reporting environments.

***

# TempDB Configuration

Most forgotten SQL Server tuning area.

***

## Number of Data Files

For 16 CPU Server

Start with:

```text
8 TempDB Data Files
1 TempDB Log File
```

***

## Equal Size Files

Example

```text
tempdev1 4GB
tempdev2 4GB
tempdev3 4GB
tempdev4 4GB
tempdev5 4GB
tempdev6 4GB
tempdev7 4GB
tempdev8 4GB
```

***

## Auto Growth

Avoid

```text
64MB
```

Use

```text
512MB
```

or

```text
1GB
```

***

# Security Hardening

***

## Disable sa Login

```sql
ALTER LOGIN sa DISABLE;
```

If policy permits.

***

## Password Policy

Enforce:

```text
Complex Password
Password Expiry
```

***

## Remove Unused Logins

Review:

```sql
SELECT * FROM sys.server_principals;
```

***

## Audit Sysadmin Access

```sql
SELECT
name,
type_desc
FROM sys.server_principals
WHERE IS_SRVROLEMEMBER('sysadmin',name)=1;
```

***

# Monitoring Setup

Install monitoring before production cutover.

Track:

```text
CPU
Memory
Blocking
Deadlocks
Disk Latency
Replication Delay
Job Failures
```

Tools:

```text
SQL Monitor
Dynatrace
Prometheus
Grafana
```

(You already use these in UST.)

***

# Reporting Replica Configuration

Two options.

***

# Option 1 (Recommended)

## Log Shipping

Production

```text
Backup Logs
Copy Logs
Restore Logs
```

Replica

```text
Read Only
```

Advantages:

```text
Simple
Low Cost
Stable
Minimal Production Impact
```

***

# Option 2

## Always On Availability Group

Advantages

```text
Near Real Time
Readable Secondary
Automatic Failover
```

Disadvantages

```text
More Licensing
More Complexity
```

***

# Reporting Workload Isolation

Move these to Reporting Server:

```text
PowerBI

SSRS

AdHoc Reporting

Long Running Reports

Analytical Queries
```

Keep on Production:

```text
CDC Refresh

AS400 Loads

Inserts

Updates

Deletes

ETL
```

***

# Database-Level Optimizations After Restore

Execute:

## Update Statistics

```sql
EXEC sp_updatestats;
```

***

## Check Fragmentation

```sql
SELECT *
FROM sys.dm_db_index_physical_stats
(
DB_ID(),
NULL,
NULL,
NULL,
'SAMPLED'
);
```

***

## Verify Missing Indexes

```sql
SELECT *
FROM sys.dm_db_missing_index_details;
```

***

# Validation Checklist Before Go-Live

### Infrastructure

✅ Windows patched

✅ SQL latest CU

✅ SSD storage

✅ 64KB allocation

✅ Dedicated service accounts

***

### SQL Configuration

✅ IFI Enabled

✅ MAXDOP configured

✅ CTP = 50

✅ TempDB optimized

✅ Max memory configured

✅ Backup compression enabled

✅ Ad hoc optimization enabled

***

### Security

✅ SA disabled/restricted

✅ Password policy enabled

✅ Least privilege access

✅ Audit enabled

***

### Reporting

✅ Log Shipping working

✅ Restore latency validated

✅ PowerBI redirected

✅ Reporting workload tested

✅ CDC performance measured after cutover

***

# Questions You Should Be Ready For

### Why not replicate PowerBI workload on existing server?

Because report queries are generating heavy I/O against tables containing:

* 377M rows
* 117M rows
* 2.14B rows

Offloading reporting reduces contention for:

* CPU
* Memory
* Disk
* TempDB

and improves CDC refresh performance.

### Why log shipping instead of replication?

Reporting workload needs complete database consistency and minimal production overhead.

Log shipping is often simpler and more reliable for read-only reporting than transactional replication.

### How much performance improvement can be expected?

From similar environments:

```text
20-40% improvement
```

on Production CPU/I/O contention.

Often:

```text
50%+ improvement
```

in reporting response times after workload isolation.

As a DBA coming from DB2, focus your discussion around **workload isolation, log shipping, storage layout, TempDB tuning, memory sizing, and parallelism settings**. These are the SQL Server equivalents of the same performance principles you already apply in enterprise DB2 environments.
