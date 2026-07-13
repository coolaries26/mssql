This is exactly the type of question a client or Change Advisory Board (CAB) may ask:

> **"Which recommendations can be implemented online and which require downtime?"**

As a SQL Server DBA, categorize them into:

* ✅ **No Downtime / Online**
* ⚠️ **Short Maintenance Window Recommended**
* 🔴 **Major Change / Planned Downtime Project**

***

# ✅ Can Be Done Without Downtime

## 1. Update Cost Threshold for Parallelism

Current:

```text
5
```

Recommended:

```text
40-50
```

Command:

```sql
EXEC sp_configure 'cost threshold for parallelism',50;
RECONFIGURE;
```

Downtime?

✅ No

Impact:

* Takes effect immediately for new query compilations.

***

# 2. Change MAXDOP

Current:

```text
4
```

Command:

```sql
EXEC sp_configure 'max degree of parallelism',4;
RECONFIGURE;
```

Downtime?

✅ No

Impact:

* New queries use the new setting.
* Running queries are unaffected.

***

# 3. Create Missing Indexes

Example:

```sql
CREATE NONCLUSTERED INDEX IX_MBMASTP
ON dbo.MBMASTP(...);
```

Downtime?

✅ Usually No

If Enterprise Edition:

```sql
WITH (ONLINE = ON)
```

can be used for large indexes.

***

# 4. Create New Covering Indexes

Example:

```sql
CREATE INDEX IX_INDLYIP_NEW
ON dbo.INDLYIP(...)
```

Downtime?

✅ No

May generate CPU and I/O load during creation.

***

# 5. Update Statistics

```sql
EXEC sp_updatestats;
```

or

```sql
UPDATE STATISTICS dbo.INSATDP;
```

Downtime?

✅ No

***

# 6. Remove Unused Indexes

Example:

```sql
DROP INDEX IX_UNUSED
ON dbo.TEST;
```

Downtime?

✅ No

But verify first.

Incorrectly dropping an index can slow applications.

***

# 7. Resource Governor Configuration

Create workload groups.

```sql
ALTER RESOURCE GOVERNOR RECONFIGURE;
```

Downtime?

✅ No

***

# 8. Enable Query Store (if not enabled)

```sql
ALTER DATABASE CUSTOM
SET QUERY_STORE = ON;
```

Downtime?

✅ No

Recommended for identifying problematic queries.

***

# 9. Memory Configuration Changes

Current:

```text
Max Server Memory = 48000 MB
```

Possible:

```text
55000 MB
```

Command:

```sql
EXEC sp_configure 'max server memory',55000;
RECONFIGURE;
```

Downtime?

✅ No

***

# 10. Configure Backup Compression

```sql
EXEC sp_configure 'backup compression default',1;
RECONFIGURE;
```

Downtime?

✅ No

***

# 11. Enable Instant File Initialization

At Windows level:

```text
Perform Volume Maintenance Tasks
```

Downtime?

✅ No SQL downtime

⚠️ SQL Service restart may be needed depending on how the privilege is granted.

***

# 12. Query Tuning

Rewriting:

* joins
* predicates
* filters

Downtime?

✅ No

Deployment can occur through normal change process.

***

# 13. Create Summary/Aggregate Tables

Example:

```text
INSATDP Summary Table
```

Downtime?

✅ No

Can be built alongside production.

***

# ⚠️ Usually Requires Maintenance Window

## 1. TempDB Reconfiguration

Your system:

```text
2600 VLFs
```

Need:

```text
8 Data Files
Proper pre-sizing
Larger autogrowth
```

Commands:

```sql
ALTER DATABASE tempdb
ADD FILE ...
```

File additions:

✅ Immediate

However:

To resize/restructure TempDB properly:

```text
SQL Service Restart
```

is normally required.

Downtime?

⚠️ Short outage

Usually 5-15 minutes.

***

# 2. Large Index Rebuilds

Example:

```sql
ALTER INDEX ALL
ON dbo.INSATDP
REBUILD;
```

Enterprise Edition:

```sql
ONLINE = ON
```

possible.

However:

377M rows

or

2.1B rows

may generate:

* heavy IO
* blocking moments
* significant log generation

Downtime?

⚠️ Prefer maintenance window.

***

# 3. Reorganizing Data Files

Moving MDF/NDF files.

Downtime?

⚠️ Usually requires maintenance window.

***

# 🔴 Major Changes Requiring Planned Project / Downtime

## 1. Table Partitioning

Examples:

```text
INSATDP
INSATSP
INCAFIF
INDLYIP
tbl_eft_sales_transaction_details
```

For existing large tables:

Creating partitioned structures requires:

* new partition function
* new partition scheme
* index movement

For 377M rows and 2.1B rows:

This is a project.

Downtime?

🔴 Usually yes.

Can be minimized but not considered a simple online change.

***

# 2. Changing Clustered Index Structure

Example:

```text
PK_AU_INDLYIP
```

Changing key order.

Most likely:

```sql
DROP EXISTING
```

or

```sql
DROP PK
CREATE PK
```

Downtime?

🔴 Maintenance window recommended.

***

# 3. In-Memory OLTP Filegroups

Recommendation:

```text
Create Memory Optimized Filegroup
```

Downtime?

🔴 Requires planning and testing.

Not usually a same-day change.

***

# 4. Reporting Server / Log Shipping Deployment

Recommendation:

```text
Dedicated Reporting Server
```

Requires:

* New server
* Log shipping
* Validation
* Power BI connection changes

Downtime?

🔴 Project activity.

Application cutover typically needed.

***

# 5. CPU/Storage Upgrade

Hardware changes.

Downtime?

🔴 Depends on virtualization and storage architecture.

Usually change window required.

***

# What I Would Recommend Implementing First (No Downtime Quick Wins)

If I were leading this engagement, I'd tell the client to first implement:

### Phase 1 (Zero Downtime)

```text
✔ Increase Cost Threshold (5 → 50)
✔ Review MAXDOP
✔ Update Statistics
✔ Create Missing Indexes
✔ Remove Unused Indexes
✔ Tune Top Queries
✔ Enable Resource Governor
✔ Increase Max Memory
✔ Enable Query Store
✔ Enable IFI
```

Expected improvement:

```text
20-50% performance gain
```

in many environments before touching hardware.

### Phase 2 (Short Maintenance Window)

```text
✔ Fix TempDB
✔ Reduce VLF Count
✔ Review Large Index Rebuild Strategy
```

### Phase 3 (Strategic Project)

```text
✔ Partition Large Tables
✔ Reporting Server
✔ Hardware Upgrade
✔ In-Memory OLTP (if justified)
```

In your meeting, emphasize that **the highest ROI, lowest-risk items are Cost Threshold, statistics, indexing, query tuning, TempDB correction, and separating reporting workloads from CDC processing.** These are the recommendations most likely to resolve report timeouts without major downtime or infrastructure investment.
