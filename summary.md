For this meeting, your goal is not to explain every SQL Server concept. Your goal is to demonstrate that:

1. **You understand the root causes of the performance issues.**
2. **You can prioritize recommendations based on business impact.**
3. **You know the risks, effort, and expected outcome of each recommendation.**
4. **You can answer "Why?" behind every recommendation.**

***

# Executive Summary You Can Open With

"I found that the performance issue is not caused by a single bottleneck. It is a combination of large tables (up to 2.1 billion rows), suboptimal indexing, aggressive parallelism settings, TempDB configuration issues, high I/O waits, and reporting workloads competing with CDC replication. My recommendations are divided into immediate quick wins, medium-term optimizations, and long-term architectural improvements."

This immediately positions you as a consultant rather than an administrator.

***

# Be Ready For These Questions

## Q1. Why are reports timing out?

### Your Answer

There are multiple contributing factors:

### Database Size

* INSATDP = 377 million rows
* INDLYIP = 117 million rows
* tbl\_eft\_sales\_Transaction\_Details = 2.14 billion rows

Queries against these tables are processing enormous volumes of data.

### Missing/Improper Indexes

Examples found:

* Missing index on MBMASTP
* Inefficient index ordering on INDLYIP
* PK\_AU\_INDLYIP column order not aligned with predicates

Result:

* Table scans
* High CPU
* High I/O
* Long query duration

### Parallelism

Cost Threshold = 5

This is extremely low.

Many relatively small queries become parallelized unnecessarily.

Result:

* Excess CPU consumption
* CXPACKET/CXCONSUMER waits
* Resource contention

### High Disk I/O

CONNECTCDC:

* 48 TB Reads
* 20 million seconds read wait

CUSTOM:

* 63 TB Reads

This clearly indicates an I/O-bound workload.

### Concurrent Workload

Peak usage:

* 6142 processes between 7 AM and 8 AM

Multiple large reporting queries running simultaneously create additional contention.

***

## Q2. Why do you recommend increasing Cost Threshold for Parallelism from 5?

### Your Answer

Current value:

```text
CTP = 5
```

SQL Server considers parallel plans even for relatively inexpensive queries.

For a server with:

```text
12 CPUs
Large databases
Mixed OLTP + Reporting
```

A starting point of:

```text
CTP = 40 or 50
```

would be more appropriate.

Benefits:

* Fewer unnecessary parallel plans
* Lower CPU pressure
* Better throughput
* More CPU available for genuinely expensive queries

I would increase gradually and monitor wait statistics and query duration.

***

## Q3. Why should MAXDOP be changed?

### Your Answer

Current configuration is inconsistent:

Database Level:

```text
MAXDOP = 0
```

Server Level:

```text
MAXDOP = 4
```

Observed query:

```text
Using 4 CPUs
CPU reached 80%
Runtime 4.38 minutes
```

For a 12-core server:

```text
MAXDOP 4
```

is generally acceptable.

However:

* Database-level settings should be reviewed.
* Parallelism should be controlled together with Cost Threshold.

Changing MAXDOP alone will not solve the issue.

***

## Q4. Why is TempDB a concern?

### Your Answer

Current problem:

```text
TempDB log growth = 64 MB
```

Result:

```text
2600 VLFs
```

Excessive VLFs can cause:

* Slow recovery
* Slower log operations
* Checkpoint delays

Recommendations:

### TempDB Data Files

Create:

```text
8 equally sized data files
```

because server has 12 CPUs.

### Log Growth

Increase:

```text
64 MB → 512 MB or 1024 MB
```

### Pre-size TempDB

Avoid repeated autogrowth.

Expected Result:

* Less allocation contention
* Lower TempDB waits
* Better query performance

***

## Q5. Why enable Instant File Initialization (IFI)?

### Your Answer

Currently disabled.

Without IFI:

When SQL Server grows a data file it performs zero initialization.

Large file growth can take significant time.

IFI allows immediate allocation of data file space.

Benefits:

* Faster autogrowth
* Faster restore
* Faster database creation

Very low-risk configuration change.

***

## Q6. Why is partitioning recommended?

### Your Answer

Some tables contain hundreds of millions or billions of rows.

Examples:

| Table                                 | Rows         |
| ------------------------------------- | ------------ |
| INSATDP                               | 377 million  |
| INDLYIP                               | 117 million  |
| tbl\_eft\_sales\_Transaction\_Details | 2.14 billion |

Without partitioning:

A query may scan very large portions of the table.

Partitioning provides:

* Partition elimination
* Faster maintenance
* Faster index rebuilds
* Improved reporting performance

Example:

```sql
WHERE transaction_date BETWEEN ...
```

If partitioned by date:

Only relevant partitions are read.

Instead of scanning 2.14 billion rows.

***

## Q7. Why suggest a summary table for INSATDP?

### Your Answer

The table has:

```text
377 million rows
```

But

```text
TDWHSE distinct values = 19
```

Many reports only require summarized warehouse-level information.

Instead of scanning:

```text
377 million rows
```

A summary table may contain:

```text
19 warehouse aggregates
```

This dramatically reduces:

* Reads
* CPU
* Execution time

***

## Q8. Why add more CPUs when average CPU is only 20-30%?

### Your Answer

Average CPU doesn't tell the whole story.

Observed:

```text
Spikes up to 80%
```

A single query consumed:

```text
4 CPUs
```

for

```text
4.38 minutes
```

During peak hour:

```text
6000+ concurrent processes
```

When multiple heavy reports run concurrently, CPUs become saturated.

I would first optimize:

1. Queries
2. Indexes
3. Parallelism

Then reassess CPU requirement.

Hardware should not be the first fix.

Clients typically like this answer.

***

## Q9. Why recommend Resource Governor?

### Your Answer

Current environment combines:

* CDC processing
* Power BI reporting
* User queries
* ETL activities

All workloads compete equally.

Resource Governor can:

* Limit reporting queries
* Reserve CPU for CDC
* Prevent one report from affecting other workloads

Business benefit:

Ensures CDC replication continues even during reporting peaks.

***

## Q10. Why recommend a reporting server?

This is probably the most important question.

### Your Answer

Current server performs:

* AS400 CDC refresh
* Report queries
* Power BI workloads
* Application activity

These workloads compete for:

* CPU
* Memory
* Disk I/O

The biggest bottleneck observed is I/O wait.

Reporting workloads are primarily read-heavy.

A dedicated reporting server allows:

```text
Production Server:
CDC refresh
DML activity
Transactional workload

Reporting Server:
Power BI
Reports
Analytics
```

Benefits:

* Reduced locking
* Reduced I/O contention
* Better report response times
* Better CDC performance

This recommendation provides the highest long-term business value.

***

# Questions You Should Ask Them

To sound like an architect, ask these:

### Business Questions

1. Which reports are timing out most frequently?

2. What is the acceptable report runtime?

```text
30 seconds?
2 minutes?
5 minutes?
```

3. What are the reporting SLAs?

4. What is business impact when reports timeout?

***

### Technical Questions

1. Are the large tables archived?

2. How often are statistics updated?

3. Are index maintenance jobs running?

4. Any Availability Groups?

5. Is storage SSD or SAN?

6. What's the average CDC latency?

7. Can PowerBI use a dedicated reporting replica?

***

# Prioritized Action Plan

### Phase 1 (Immediate)

✅ Enable IFI

✅ Fix TempDB

✅ Increase Cost Threshold

✅ Review MAXDOP

✅ Create missing indexes

✅ Remove unused indexes

✅ Update statistics

***

### Phase 2 (Short-Term)

✅ Index redesign

✅ Query tuning

✅ Resource Governor

✅ Analyze top queries

✅ Review execution plans

***

### Phase 3 (Strategic)

✅ Partition 2+ billion row tables

✅ Create summary/aggregation tables

✅ Reporting server via Log Shipping

✅ Hardware upgrade if required

***

# Strong Closing Statement

"I do not believe hardware alone will solve the issue. The primary improvements will come from correcting indexing, query design, parallelism settings, TempDB configuration, and separating reporting workloads from transactional and CDC workloads. Once these optimizations are implemented, we can accurately determine whether additional CPU or memory is still required."

This answer will resonate well with architects, infrastructure teams, DBAs, and management alike.
