# Safe Automated Storage Cost Optimization for SQL Server on Azure VMs with Database Chex

## Executive Summary

SQL Server workloads running on Azure virtual machines are often intentionally overprovisioned. Infrastructure teams typically prefer excess storage capacity, IOPS, throughput, CPU, and memory over the risk of reducing resources and affecting a production database.

That approach provides operational safety, but it can also create significant and persistent Azure cost.

Database Chex can address this problem by identifying infrastructure that is genuinely overprovisioned and recommending a lower-cost configuration while maintaining an agreed performance safety margin.

The challenge is that storage rightsizing cannot safely be based on Azure disk utilization alone.

Low disk utilization does not automatically mean that a disk is oversized. SQL Server might be consuming less storage because of memory caching, query behavior, blocking, CPU saturation, Azure host caching, VM-level I/O limits, temporary burst capacity, or other infrastructure constraints. A workload may also experience peaks during backups, ETL jobs, month-end processing, index maintenance, failovers, or other events that are not visible in a short monitoring window.

The proposed Database Chex solution therefore treats storage optimization as a **controlled performance-engineering process**, rather than a simple infrastructure resizing operation.

The solution follows this lifecycle:

```text
Observe
   ↓
Diagnose
   ↓
Establish Workload Demand
   ↓
Rule Out Competing Bottlenecks
   ↓
Calculate Required Performance
   ↓
Apply Minimum 20% Headroom
   ↓
Generate Lower-Cost Candidates
   ↓
Simulate Candidate Configuration
   ↓
Assess Risk and Confidence
   ↓
Execute Under Policy
   ↓
Validate Application and SQL Performance
   ↓
Commit or Automatically Roll Back
```

The central design principle is:

> Database Chex should only reduce infrastructure when it can demonstrate that the proposed configuration still satisfies the SQL Server workload's required performance envelope with at least 20% headroom and when there is no material evidence that another bottleneck is masking the workload's true storage requirement.

This approach allows Database Chex to evolve from a monitoring and recommendation platform into a closed-loop database optimization platform while maintaining strong operational safety.

---

# 1. Business Problem

SQL Server on Azure virtual machines gives customers significant control over infrastructure, but that flexibility creates a difficult operational trade-off.

Infrastructure teams must select:

* Azure VM size
* managed disk type
* disk capacity
* provisioned IOPS
* provisioned throughput
* number of disks
* caching configuration
* Storage Spaces configuration
* data, log, and `tempdb` placement

Customers without dedicated SQL Server DBA expertise often respond to uncertainty by provisioning more infrastructure than the workload actually requires.

For example, an environment may have:

```text
Provisioned disk performance:     20,000 IOPS
Observed normal workload:          4,000 IOPS
Observed peak workload:            8,000 IOPS
```

The apparent opportunity is obvious.

However, reducing the disk directly from 20,000 IOPS to 10,000 IOPS would be unsafe without first answering several questions:

* Was the peak workload fully observed?
* Did the sample include backups and maintenance?
* Did it include month-end or quarter-end processing?
* Is memory caching hiding potential read demand?
* Is SQL Server currently constrained by CPU?
* Is blocking suppressing database throughput?
* Is a query plan regression generating unusual I/O?
* Is the VM itself limiting disk performance?
* Is the workload using disk bursting?
* Is Azure host caching serving a significant portion of reads?
* Would the proposed storage configuration still support the workload after a SQL Server restart when the buffer cache is cold?

Microsoft's current SQL Server on Azure VM guidance recommends collecting CPU, memory, IOPS, throughput, latency, and SQL Server wait information across representative peak workload periods before selecting infrastructure capacity. ([Microsoft Learn][1])

Database Chex should apply the same principle before reducing infrastructure.

---

# 2. Solution Objective

Database Chex should answer the following question:

> What is the lowest-cost Azure infrastructure configuration that can reliably satisfy the observed and forecast SQL Server workload while retaining at least 20% performance headroom and without introducing measurable workload degradation?

This is fundamentally different from asking:

> How much disk capacity is currently unused?

The optimization must consider the complete workload performance envelope.

The target configuration must satisfy, at minimum:

```text
Required IOPS
Required throughput
Required storage capacity
VM I/O limits
VM CPU requirements
VM memory requirements
Caching requirements
SQL Server workload characteristics
High availability requirements
Recovery requirements
Expected workload growth
```

The resulting recommendation must also provide measurable financial value.

---

# 3. Core Architectural Principle: Diagnose Before Rightsizing

Database Chex should separate two questions:

### Question 1: What is causing the current performance behavior?

This is the responsibility of the **Diagnosis Engine**.

### Question 2: What infrastructure does the workload actually require?

This is the responsibility of the **Rightsizing and Optimization Engine**.

These should be independent components.

A low disk utilization measurement must never automatically trigger a storage reduction.

Instead, Database Chex must first determine whether current storage demand reflects the true workload requirement.

---

# 4. Multi-Layer Performance Model

A SQL Server request passes through several technology layers.

```text
Application
     │
     ▼
SQL Queries and Execution Plans
     │
     ▼
SQL Server Database Engine
     │
     ▼
Windows Operating System
     │
     ▼
Azure Virtual Machine
     │
     ▼
Azure Managed Disk
```

A slowdown or low throughput can originate at any of these layers.

Database Chex should therefore correlate telemetry across all of them.

The major diagnostic domains are:

| Domain         | Example Problems                                   |
| -------------- | -------------------------------------------------- |
| Query          | Poor execution plan, scan, excessive logical reads |
| SQL Engine     | Blocking, waits, memory grants, configuration      |
| Memory         | Buffer pool pressure, paging, low available memory |
| Compute        | CPU saturation                                     |
| Storage        | Disk IOPS, throughput, latency                     |
| VM             | VM-level IOPS or bandwidth limit                   |
| Cache          | Buffer pool or Azure host cache effects            |
| Infrastructure | Bursting, disk topology, Storage Spaces            |

---

# 5. Required Telemetry

The accuracy of the recommendation depends directly on the telemetry available.

Database Chex should collect signals from four primary sources.

## 5.1 Azure Monitor Metrics

Important per-disk metrics include:

```text
Data Disk IOPS Consumed Percentage
Data Disk Bandwidth Consumed Percentage
Data Disk Latency
Data Disk Queue Depth
Data Disk Read Operations/Sec
Data Disk Write Operations/Sec
Data Disk Read Bytes/Sec
Data Disk Write Bytes/Sec
Data Disk Target IOPS
Data Disk Target Bandwidth
Disk burst-credit metrics
```

Azure exposes these metrics with LUN-level dimensions, allowing individual disks to be analyzed independently. ([Microsoft Learn][2])

Important VM-level metrics include:

```text
VM Cached IOPS Consumed Percentage
VM Cached Bandwidth Consumed Percentage
VM Uncached IOPS Consumed Percentage
VM Uncached Bandwidth Consumed Percentage
VM cached burst-credit metrics
VM uncached burst-credit metrics
CPU utilization
```

Azure explicitly separates disk-level I/O capping from VM-level I/O capping. ([Microsoft Learn][3])

That distinction is critical for correct diagnosis.

---

# 6. SQL Server Telemetry

Database Chex should capture SQL Server information including:

```text
SQL Server wait statistics
Query Store runtime statistics
Query Store execution plans
logical reads
physical reads
writes
batch requests/sec
transactions/sec
SQL Server memory counters
Page Life Expectancy
memory grants pending
buffer cache behavior
blocking
deadlocks
database file statistics
transaction-log latency
tempdb behavior
SQL Agent activity
backup operations
DBCC operations
index maintenance
```

SQL telemetry allows Database Chex to determine whether infrastructure demand is caused by normal workload activity or by another database problem.

---

# 7. Windows Telemetry

Relevant Windows counters include:

```text
Logical Disk Reads/sec
Logical Disk Writes/sec
Logical Disk Read Bytes/sec
Logical Disk Write Bytes/sec
Average Disk sec/Read
Average Disk sec/Write
Disk Queue Length
Available Memory
Processor Utilization
```

Microsoft specifically recommends collecting disk IOPS and throughput through Windows performance counters while establishing SQL Server workload requirements. ([Microsoft Learn][1])

---

# 8. Azure Resource Configuration

Database Chex must also understand the static infrastructure topology:

```text
VM SKU
VM family
CPU count
memory
VM maximum IOPS
VM maximum throughput
managed disk type
managed disk size
provisioned IOPS
provisioned throughput
disk LUN
host caching mode
Storage Spaces configuration
disk striping
drive letters
SQL Server file locations
Availability Set or Availability Zone
HA/DR topology
```

Rightsizing cannot be performed correctly without this topology.

---

# 9. Determine Whether Storage Is Actually the Bottleneck

A storage bottleneck should be diagnosed using multiple correlated signals.

For example:

```text
Disk IOPS consumed:      98%
Disk latency:             20 ms
Disk queue depth:         increasing
PAGEIOLATCH waits:        increasing
VM IOPS consumed:         60%
```

This strongly indicates an individual disk IOPS constraint.

However:

```text
Disk IOPS consumed:       45%
VM uncached IOPS:        100%
Disk latency:             high
PAGEIOLATCH:              high
```

indicates that the VM storage envelope, rather than the individual managed disk, is limiting throughput.

Azure documentation specifically describes using disk and VM consumed-percentage metrics to distinguish these situations. ([Microsoft Learn][3])

In the second scenario, reducing disk performance would be inappropriate.

---

# 10. SQL Server Wait Analysis

Storage-related SQL Server waits are important evidence but should never be interpreted in isolation.

Important examples include:

```text
PAGEIOLATCH_SH
PAGEIOLATCH_EX
PAGEIOLATCH_UP
WRITELOG
```

However, `PAGEIOLATCH` does not automatically mean that Azure storage is slow.

A poorly designed query that scans a large table can generate high physical I/O against perfectly healthy storage.

Database Chex should therefore correlate:

```text
SQL I/O waits
+
physical I/O
+
disk latency
+
disk consumed percentage
+
VM consumed percentage
+
query behavior
```

Only when the signals align should storage receive a high root-cause confidence score.

---

# 11. Competing Bottleneck Analysis

Before changing storage, Database Chex should evaluate alternative explanations.

Recommended categories include:

| Potential Cause   | Example Evidence                           |
| ----------------- | ------------------------------------------ |
| Storage           | disk latency, disk throttling, PAGEIOLATCH |
| Transaction log   | WRITELOG, log write latency                |
| VM-level I/O      | VM cached/uncached IOPS limits             |
| Query regression  | Query Store plan change                    |
| Poor query design | abnormal reads per execution               |
| CPU               | CPU saturation, scheduler waits            |
| Memory            | low available memory, PLE decline          |
| Blocking          | lock waits, blocked sessions               |
| tempdb            | latch contention, tempdb I/O               |
| Configuration     | MAXDOP, max memory, file layout            |
| Caching           | cache-hit behavior masking physical demand |

Database Chex can represent these as independent diagnostic confidence scores.

For example:

```text
Storage constraint confidence:        91%
Query regression confidence:          14%
Memory pressure confidence:           11%
Blocking confidence:                   3%
CPU constraint confidence:             2%
```

The values do not necessarily have to total 100 because multiple conditions can coexist.

Automation should only proceed when storage demand is well understood and no competing condition materially undermines the analysis.

---

# 12. Query Store as a Critical Input

Query Store is especially important because storage demand can change dramatically because of an execution-plan regression.

Consider:

```text
Monday physical I/O:     4,000 IOPS
Tuesday physical I/O:   14,000 IOPS
```

An infrastructure-only system may conclude that the workload now requires 14,000 IOPS.

However, Query Store could show:

```text
Query ID: 194

Previous plan:
20,000 logical reads / execution

New plan:
8,500,000 logical reads / execution
```

The correct recommendation would be to investigate the query regression.

Increasing storage would treat the symptom rather than the cause.

This is an important differentiator for Database Chex because generic Azure rightsizing products typically have limited understanding of SQL Server execution behavior.

---

# 13. Memory Must Be Evaluated Before Storage

SQL Server uses memory aggressively as a data cache.

A storage workload can therefore be heavily influenced by the buffer pool.

Consider:

```text
Database size:               2 TB
SQL Server memory:         128 GB
Available OS memory:         low
Page Life Expectancy:       poor
Physical reads:             high
```

The excessive physical I/O may primarily result from memory pressure.

Reducing storage performance in that situation could make the system significantly slower.

The reverse problem is also important.

A database may normally generate low physical reads because frequently accessed pages are cached.

Following:

```text
SQL restart
VM restart
failover
patching
host maintenance
cache eviction
```

physical I/O can increase sharply.

Database Chex should therefore distinguish between:

```text
Warm-cache workload
Cold-cache workload
```

The candidate storage configuration should tolerate both where operationally required.

---

# 14. Blocking Must Be Evaluated

Blocking can artificially reduce infrastructure utilization.

For example:

```text
CPU:               30%
Disk IOPS:         25%
Throughput:        22%
```

may appear to indicate substantial overprovisioning.

But if hundreds of sessions are blocked on a transaction, the infrastructure is not being fully exercised.

Database Chex should not interpret suppressed demand as evidence that the infrastructure can safely be reduced.

---

# 15. CPU Constraints Must Be Evaluated

A CPU-bound SQL Server workload may not generate its potential storage demand.

For example:

```text
CPU:              99%
Disk IOPS:        45%
```

Reducing disk performance based only on the current I/O measurement could create a second bottleneck after the CPU issue is resolved.

Therefore CPU saturation should block or significantly reduce the confidence of storage-rightsizing recommendations.

---

# 16. Understand the Azure VM I/O Envelope

Every Azure VM SKU has maximum cached and uncached disk performance limits.

A collection of disks might theoretically provide more IOPS than the VM can consume.

For example:

```text
Attached disks aggregate capability:   50,000 IOPS
VM uncached IOPS limit:                25,000 IOPS
```

The workload cannot consume 50,000 uncached IOPS through that VM.

Database Chex must therefore ensure:

```text
Candidate disk requirement
<=
Candidate VM storage limit
```

and independently:

```text
Required IOPS <= VM IOPS limit
Required throughput <= VM throughput limit
```

A proposed VM downsizing operation must repeat the same checks because smaller VM SKUs frequently have lower storage limits.

---

# 17. Understand SQL Server Storage Roles

Database Chex should map Azure disks through Windows volumes to SQL Server files.

The desired relationship is:

```text
Database
   ↓
SQL file
   ↓
Windows volume
   ↓
Storage Spaces / volume configuration
   ↓
Azure managed disk
   ↓
LUN
```

Each storage component should then be classified by role:

```text
DATA
TRANSACTION LOG
TEMPDB
BACKUP
OPERATING SYSTEM
OTHER
```

This classification matters because different SQL Server file types generate different I/O patterns.

Microsoft's SQL Server on Azure VM best-practice guidance recommends evaluating data, log, and `tempdb` requirements separately and includes different caching guidance for data and log disks. ([Microsoft Learn][4])

---

# 18. Storage Spaces and Disk Striping

Many SQL Server deployments stripe multiple managed disks through Windows Storage Spaces.

For example:

```text
LUN 0 ─┐
LUN 1 ─┼── Storage Pool ── F:\ SQLData
LUN 2 ─┼
LUN 3 ─┘
```

Database Chex cannot safely optimize one disk without understanding the complete pool.

The system should collect:

```text
Storage pool
physical disk members
LUN mappings
stripe configuration
interleave
column count
filesystem
allocation unit
volume size
SQL file placement
```

Rightsizing decisions for striped storage should be made at the storage-pool level rather than purely at the individual-LUN level.

---

# 19. Caching Must Be Explicitly Modeled

Caching can materially change apparent storage demand.

Azure Premium SSD supports host caching, while Premium SSD v2 does not support host caching. ([Microsoft Learn][5])

Database Chex should therefore distinguish:

```text
Cached reads
Uncached reads
Writes
```

For example:

```text
SQL total read demand:       18,000 IOPS
Reads served by host cache:  14,000 IOPS
Managed disk reads:           4,000 IOPS
```

The correct model is not simply:

```text
Disk requirement = 4,000 IOPS
```

The effective performance architecture depends on both caching behavior and VM cached-I/O limits.

A change in disk technology or VM type may alter those assumptions.

---

# 20. Bursting Must Not Be Treated as Sustainable Capacity

Azure disks and VMs can provide burst performance depending on SKU and configuration.

For example:

```text
Base sustainable IOPS:     5,000
Temporary burst IOPS:     10,000
Workload peak:             8,000
```

A superficial analysis might conclude that the system has sufficient capacity.

However, if the workload regularly depends on burst performance, it may slow down after burst capacity is exhausted.

Database Chex should therefore calculate requirements against sustainable capacity.

Bursting should normally be treated as:

> Additional resilience for temporary peaks, not as baseline workload capacity.

Relevant Azure Monitor metrics include both disk and VM burst-credit consumption. ([Microsoft Learn][3])

Regular dependency on burst credits should substantially reduce automation eligibility.

---

# 21. Separate Capacity, IOPS, and Throughput

Storage has at least three independent constraints:

```text
Capacity
IOPS
Throughput
```

Each must be calculated separately.

For example:

```text
Used storage:             800 GiB
Projected growth:         100 GiB/year
Peak IOPS:              8,500
Peak throughput:          420 MB/s
```

A candidate must satisfy all three requirements.

The least expensive disk is irrelevant if it violates any one of them.

---

# 22. Define the 20% Headroom Requirement Correctly

A common mistake would be:

```text
Peak × 1.20
```

That produces only 16.7% unused capacity when measured against the final capacity.

If Database Chex must ensure that the workload consumes no more than 80% of the provisioned resource, the correct formula is:

```text
Provisioned Capacity >= Required Demand / 0.80
```

Example:

```text
Observed required IOPS = 12,000

Minimum provisioned IOPS:

12,000 / 0.80
= 15,000 IOPS
```

At 12,000 IOPS of demand:

```text
12,000 / 15,000 = 80%
```

leaving:

```text
20%
```

unused capacity.

The same formula applies independently to throughput.

Example:

```text
Required throughput = 400 MB/s

400 / 0.80
= 500 MB/s
```

---

# 23. Twenty Percent Should Be a Minimum, Not Always the Target

Different workloads have different levels of volatility.

A stable OLTP system may safely operate with approximately 20–25% headroom.

A highly bursty workload may require considerably more.

Database Chex should therefore calculate:

```text
Effective Headroom =
MAX(
    Policy Minimum,
    Workload Volatility Requirement,
    Growth Requirement,
    Recovery / Cold Cache Requirement
)
```

Example:

```text
Minimum policy requirement:       20%
Workload volatility adjustment:    8%
Expected growth adjustment:        5%

Effective recommended headroom:   33%
```

The exact implementation does not have to add percentages mechanically; a probabilistic model may ultimately be preferable.

The important principle is that 20% is a floor.

---

# 24. Do Not Size from Average Utilization

Average IOPS hides peaks.

Maximum IOPS can be distorted by one unusual event.

Database Chex should evaluate a distribution.

Recommended percentiles include:

```text
P50
P90
P95
P99
P99.5
P99.9
Maximum
```

The system should also classify workload windows.

For example:

```text
Business hours
Nightly batch
ETL window
Backup window
Index-maintenance window
Weekend processing
Month-end
Quarter-end
```

One possible policy is:

```text
Required IOPS =
MAX(
    P99.5 normal workload,
    P99 batch workload,
    P99 maintenance workload,
    P99 business-critical peak
)
```

Then:

```text
Provisioned IOPS =
Required IOPS / 0.80
```

The exact percentile should be configurable and eventually informed by observed production behavior.

---

# 25. Observation Window and Workload Periodicity

A seven-day observation period may be enough for some systems and dangerously insufficient for others.

SQL workloads often include:

```text
weekly reporting
biweekly payroll
month-end close
quarter-end processing
nightly ETL
weekly maintenance
monthly reconciliation
```

Database Chex should therefore evaluate workload coverage, not merely observation duration.

A useful initial policy might be:

```text
Minimum observation:
7 days

Preferred observation:
30 days

High-confidence optimization:
30–90 days depending on workload periodicity
```

However, the more important rule is:

> Automated downsizing should not occur until important known workload cycles have been observed.

The recommendation should explicitly communicate missing workload coverage.

Example:

> 21 days of telemetry are available, but month-end processing has not yet been observed. The recommendation is therefore classified as Medium Confidence and is not eligible for automatic execution.

---

# 26. Include Workload Growth

A configuration that has 20% headroom today may not have 20% headroom six months from now.

Database Chex should track historical trends such as:

```text
database size growth
transaction growth
IOPS growth
throughput growth
CPU growth
memory requirement growth
```

The sizing model can incorporate future demand.

For example:

```text
Current P99 requirement:      8,000 IOPS
Expected 90-day growth:         10%

Forecast demand:
8,800 IOPS

Minimum candidate:
8,800 / 0.80
= 11,000 IOPS
```

The system should calculate headroom against forecast workload where enough data is available.

---

# 27. Build a Candidate Configuration Solver

Database Chex should not simply recommend:

> Reduce IOPS by 20%.

Instead, it should generate all technically valid lower-cost configurations.

Inputs might include:

```text
Current VM configuration
Current storage topology
Observed performance envelope
Forecast workload
Required headroom
Azure region
Supported disk types
Supported VM families
Availability requirements
Storage capacity
IOPS requirement
Throughput requirement
Host caching requirements
HA/DR requirements
Customer policy
```

Possible candidates could include:

```text
Reduce disk IOPS
Reduce disk throughput
Change Premium SSD performance tier
Adopt Premium SSD v2
Change disk topology
Change VM SKU
Change both VM and storage
```

The optimizer would then select the lowest-cost candidate that satisfies all constraints.

Conceptually:

```text
Minimize:

MonthlyAzureCost(candidate)
```

Subject to:

```text
CandidateIOPS >= RequiredIOPS / 0.80

CandidateThroughput >= RequiredThroughput / 0.80

CandidateCapacity >= RequiredCapacity

CandidateVMIOPS >= RequiredIOPS

CandidateVMThroughput >= RequiredThroughput

CandidateMemory >= RequiredMemory

CandidateCPU >= RequiredCPU

CandidateDiskCount <= VMMaxDiskCount

Availability >= ExistingAvailabilityRequirement

RecoveryObjectives satisfied

SQL storage requirements satisfied
```

---

# 28. Premium SSD v2 Is Particularly Suitable for Safe Performance Rightsizing

Premium SSD v2 allows capacity, IOPS, and throughput to be configured more independently than traditional Premium SSD tiers.

Microsoft currently documents that Premium SSD v2 performance can be adjusted programmatically, without requiring a disk-size change. ([Microsoft Learn][6])

That makes it a strong candidate for controlled optimization.

For example:

```text
Current configuration

Capacity:       1,024 GiB
IOPS:          20,000
Throughput:       800 MB/s
```

Database Chex might determine that the workload safely requires:

```text
IOPS:          12,000
Throughput:       550 MB/s
```

The capacity can remain unchanged while the paid performance envelope is reduced.

Premium SSD v2 pricing depends on configured capacity, IOPS, and throughput, with baseline performance included before additional provisioned performance creates additional charges. ([Microsoft Learn][7])

This allows Database Chex to directly connect observed workload demand to potential Azure savings.

Microsoft currently allows Premium SSD v2 performance to be adjusted up to four times within a 24-hour period, with disk creation counting as one adjustment. ([Microsoft Learn][6])

The remediation engine must account for that operational limit when designing incremental changes and rollback behavior.

---

# 29. Traditional Premium SSD Requires Different Logic

Traditional Premium SSD uses defined performance tiers more heavily than Premium SSD v2.

Some Premium SSD configurations allow changing the disk performance tier without changing disk capacity and without downtime. ([Microsoft Learn][8])

Database Chex therefore requires disk-type-specific remediation strategies.

The automation engine should never assume that the same storage operation is valid for every Azure managed-disk type.

---

# 30. Disk Capacity Downsizing Is Fundamentally Different

Reducing disk performance is relatively simple.

Reducing disk capacity is not.

Azure does not support directly shrinking the size of an existing managed disk. Microsoft currently instructs customers who require a smaller data disk to create a new smaller disk, copy the data, and then retire the original disk after successful migration. ([Microsoft Learn][9])

Therefore Database Chex should distinguish two separate optimization classes.

---

# 31. Optimization Class A: Performance Rightsizing

Example:

```text
Current:

20,000 IOPS
800 MB/s
```

Recommended:

```text
12,000 IOPS
550 MB/s
```

Characteristics:

```text
No filesystem change
No SQL file movement
No database migration
No drive-letter change
No storage-capacity reduction
```

This should be the preferred starting point for automated remediation.

---

# 32. Optimization Class B: Capacity Rightsizing

Example:

```text
Current disk:
2 TiB

Used data:
600 GiB

Target:
1 TiB
```

Because an Azure managed disk cannot be shrunk directly, this operation would require orchestration similar to:

```text
Create smaller managed disk
        ↓
Attach disk
        ↓
Initialize and format
        ↓
Create Storage Spaces layout if required
        ↓
Copy data
        ↓
Quiesce SQL Server workload
        ↓
Perform final synchronization
        ↓
Stop or detach SQL dependencies as required
        ↓
Cut over
        ↓
Validate SQL file access
        ↓
Start SQL workload
        ↓
Validate database recovery
        ↓
Validate performance
        ↓
Retain original disk for rollback
        ↓
Delete original disk only after retention policy expires
```

This operation has a much higher risk profile.

Database Chex should initially treat capacity reduction as a recommendation or guided change rather than a fully autonomous operation.

---

# 33. VM Rightsizing Is Also a Separate Optimization

Reducing VM size can affect:

```text
CPU
memory
cached IOPS
uncached IOPS
cached throughput
uncached throughput
network bandwidth
temporary-disk capacity
maximum attached disks
```

Therefore a VM that appears oversized based on CPU may still be required because of storage throughput.

Example:

```text
Current VM:
E32-series

Candidate:
E16-series
```

Database Chex should produce a validation matrix such as:

| Resource           | Candidate Result |
| ------------------ | ---------------- |
| CPU                | PASS             |
| Memory             | PASS             |
| Network            | PASS             |
| Disk count         | PASS             |
| Cached IOPS        | PASS             |
| Uncached IOPS      | FAIL             |
| Throughput         | PASS             |
| tempdb requirement | PASS             |

Result:

```text
Candidate rejected
```

Microsoft's current SQL Server on Azure VM guidance explicitly recommends sizing compute by measured CPU, memory, IOPS, throughput, and peak workload needs. ([Microsoft Learn][1])

---

# 34. Cost Validation

A recommendation should only proceed if it creates meaningful savings.

For each candidate Database Chex should calculate:

```text
Current monthly infrastructure cost
Candidate monthly infrastructure cost
Monthly savings
Annualized savings
Percentage savings
Change risk
```

For example:

```text
Current storage cost:       $620/month
Candidate storage cost:     $390/month

Monthly savings:            $230
Annualized savings:       $2,760
Savings percentage:          37%
```

Database Chex should retrieve current Azure pricing programmatically rather than storing static prices in product logic because prices can vary by:

```text
region
currency
redundancy
disk type
capacity
performance
commercial agreement
```

A minimum savings threshold should also be configurable.

For example:

```text
Savings < $20/month:
Suppress remediation

Savings between $20–$100:
Recommendation only

Savings > $100:
Eligible for policy evaluation
```

The exact values should be customer configurable.

---

# 35. Recommendation Confidence Model

Every optimization should carry a confidence score.

A useful conceptual model is:

```text
Confidence =
Telemetry Completeness
×
Workload Coverage
×
Diagnosis Confidence
×
Demand Prediction Confidence
×
Topology Confidence
×
Candidate Validation Confidence
```

Example:

```text
Observation period:          45 days
Month-end included:          Yes
Backup windows included:     Yes
Telemetry gaps:              None
Storage topology known:      Yes
Query regressions:           None
Memory pressure:             None
Storage-demand stability:    High
Storage diagnosis:           94%
```

Database Chex might report:

```text
Overall recommendation confidence:
96 / 100
```

A possible automation policy could be:

```text
95–100
Eligible for policy-controlled autonomous remediation

85–94
Human approval required

70–84
Recommendation only

Below 70
Insufficient evidence
```

These thresholds should be configurable and refined through production experience.

---

# 36. Introduce Shadow Validation

Before touching production infrastructure, Database Chex should run the proposed configuration virtually against subsequent telemetry.

Assume:

```text
Current:
20,000 IOPS

Recommended:
12,000 IOPS
```

With 20% required headroom:

```text
Maximum workload target:
12,000 × 0.80
= 9,600 IOPS
```

Database Chex can continue monitoring while pretending that 12,000 IOPS is already the limit.

If the next seven days show:

```text
Maximum observed:
8,400 IOPS
```

the recommendation remains viable.

If a later workload reaches:

```text
10,800 IOPS
```

the candidate automatically fails shadow validation.

No infrastructure change has yet occurred.

This capability acts as a lightweight digital twin for the proposed configuration and can substantially reduce optimization risk.

---

# 37. Incremental Performance Reduction

Where the Azure disk type permits it, Database Chex should prefer incremental changes rather than one large reduction.

Example:

```text
20,000 IOPS
    ↓
17,000 IOPS
    ↓
14,000 IOPS
    ↓
12,000 IOPS
```

At each stage:

```text
Apply
↓
Observe
↓
Validate
↓
Continue or Roll Back
```

The number and timing of adjustments must respect Azure platform limits for the applicable disk technology.

---

# 38. Pre-Change Safety Gates

Before executing an automated change, Database Chex should run a final pre-change assessment.

Recommended gates include:

```text
SQL Server instance healthy
Databases ONLINE
Availability Groups healthy
Replicas synchronized where required
No active database restore
No active critical backup
No critical SQL Agent job running
No index maintenance running
No abnormal blocking
No unusual CPU pressure
No abnormal memory pressure
No current storage saturation
No active Azure resource issue
No active failover
Rollback configuration captured
Change allowed by maintenance policy
```

If any critical gate fails:

```text
Change = NOT EXECUTED
```

The system should record why.

Example:

> Optimization deferred because a production database backup is currently active.

---

# 39. Establish an Immediate Pre-Change Baseline

Historical baselines are necessary, but Database Chex should also capture a near-change performance baseline.

For example, during the 30–60 minutes before the change:

```text
transactions/sec
batch requests/sec
P50 query duration
P95 query duration
P99 query duration
CPU
memory
Page Life Expectancy
IOPS
throughput
disk latency
queue depth
wait statistics
blocking
deadlocks
top Query Store queries
application errors
```

These metrics provide the most accurate before-and-after comparison.

---

# 40. Validate Performance, Not Merely Availability

A successful Azure API operation does not mean the optimization succeeded.

Likewise:

```text
SQL Server service = Running
```

is not enough.

Database Chex must validate workload performance.

Possible validation criteria include:

```text
P95 query duration degradation <= 10%

P99 query duration within policy

Disk latency within baseline range

PAGEIOLATCH waits not materially increased

WRITELOG waits not materially increased

Disk IOPS utilization within expected limit

Disk throughput utilization within expected limit

VM I/O utilization healthy

Blocking not increased

Application errors not increased

Transaction throughput not reduced
```

Thresholds should be policy driven.

---

# 41. Normalize Performance Against Workload

Before-and-after metrics must be compared against similar workload volume.

Consider:

```text
Before:
5,000 transactions/minute
P95 latency = 100 ms

After:
3,000 transactions/minute
P95 latency = 90 ms
```

Latency appears better.

However, the workload is 40% lower.

Database Chex should therefore normalize metrics where appropriate.

Examples include:

```text
CPU per transaction
IOPS per transaction
physical reads per query execution
logical reads per execution
writes per transaction
latency per Query Store query_id
resource consumption per batch request
```

This helps prevent a low-traffic period from being mistaken for a successful optimization.

---

# 42. Automatic Rollback Must Be a Core Capability

Every automated change should contain an explicit rollback definition before execution.

For example:

```text
Original state:
20,000 IOPS
800 MB/s

Target state:
12,000 IOPS
550 MB/s

Rollback state:
20,000 IOPS
800 MB/s
```

The change record should include:

```text
original configuration
target configuration
rollback configuration
rollback trigger
rollback validation
rollback deadline
```

For capacity migrations, the original managed disk should remain available during the validation and retention window.

It should not be deleted immediately after cutover.

---

# 43. Immediate Versus Sustained Rollback Conditions

Some failures require immediate rollback.

Examples:

```text
SQL Server unavailable
database unavailable
database enters suspect state
disk errors
Availability Group unhealthy
critical application errors
failed disk operation
```

Performance conditions should usually require sustained evidence.

For example:

```text
P95 query latency increases >25%
for 10 minutes
```

or:

```text
Disk IOPS utilization >90%
for 5 minutes
```

or:

```text
Disk latency materially above baseline
for 5 minutes
```

or:

```text
Critical SQL I/O waits increase >50%
for 10 minutes
```

This prevents unnecessary rollback because of a one-minute spike.

---

# 44. Post-Change Observation Window

A change should not be considered permanently successful immediately after execution.

Database Chex should maintain a post-change validation period.

For example:

```text
Immediate validation:
first 15 minutes

Short-term validation:
first 2 hours

Workload-cycle validation:
24 hours or longer

Final confirmation:
after next representative peak period
```

The required window should depend on workload periodicity.

---

# 45. Recommendation Example

Consider the following environment.

```text
VM:
Azure SQL Server VM

Disk:
Premium SSD v2

Current provisioned IOPS:
20,000

Current throughput:
750 MB/s
```

Thirty-day telemetry:

```text
IOPS

P50:       2,800
P95:       5,700
P99:       7,400
P99.9:     8,600


Throughput

P50:       120 MB/s
P95:       290 MB/s
P99:       360 MB/s
P99.9:     410 MB/s
```

The observation window includes:

```text
Nightly backups:       Yes
Weekly maintenance:    Yes
Month-end:             Yes
ETL workload:          Yes
```

Diagnostic analysis:

```text
Memory pressure:       None
CPU saturation:        None
Blocking:              Normal
Query regression:      None
VM I/O throttling:     None
Disk throttling:       None
Burst dependency:      None
```

Required IOPS:

```text
8,600 / 0.80
= 10,750
```

Required throughput:

```text
410 / 0.80
= 512.5 MB/s
```

Database Chex selects:

```text
Recommended IOPS:
12,000

Recommended throughput:
550 MB/s
```

Resulting IOPS headroom:

```text
1 - (8,600 / 12,000)

= 28.3%
```

Resulting throughput headroom:

```text
1 - (410 / 550)

= 25.5%
```

Both exceed the required 20%.

The candidate therefore passes performance-envelope validation.

---

# 46. Shadow Validation Example

Database Chex keeps the recommendation in shadow mode for another seven days.

Observed maximums:

```text
IOPS:
8,750

Throughput:
418 MB/s
```

Against the candidate:

```text
IOPS utilization:

8,750 / 12,000
= 72.9%


Throughput utilization:

418 / 550
= 76.0%
```

The candidate remains within the safety envelope.

---

# 47. Post-Change Validation Example

Database Chex applies:

```text
20,000 → 12,000 IOPS

750 → 550 MB/s
```

Post-change results:

```text
P95 query latency:       +1.8%
P99 query latency:       +2.9%
PAGEIOLATCH waits:       unchanged
WRITELOG waits:          unchanged
Disk latency:            unchanged
Application errors:      unchanged
Peak IOPS utilization:    73%
Peak throughput:          76%
```

The performance change is within policy.

Database Chex records:

```text
Optimization Status:
SUCCESSFUL
```

---

# 48. Example Where Optimization Must Be Rejected: Memory Pressure

Assume:

```text
Provisioned IOPS:
20,000

Observed IOPS:
7,000
```

At first glance this appears overprovisioned.

However:

```text
Available memory:        Low
Page Life Expectancy:    Frequently collapsing
Physical reads:          Elevated
Query Store:             Plan regression detected
PAGEIOLATCH:             High
```

Database Chex should not reduce storage.

The recommendation should state:

> Storage optimization is currently blocked because SQL Server is experiencing memory pressure and a query plan regression that materially affect physical I/O demand. These conditions should be corrected before the long-term storage requirement is calculated.

---

# 49. Example Where Optimization Must Be Rejected: VM I/O Limit

Consider:

```text
Disk IOPS consumed:            55%
VM uncached IOPS consumed:    100%
Disk latency:                  Elevated
PAGEIOLATCH:                   Elevated
```

The disk is not the primary bottleneck.

The VM's storage envelope is being reached.

Database Chex should state:

> The managed disks are not currently the limiting resource. The Azure VM's uncached I/O limit is being reached. Storage downsizing is therefore not recommended.

---

# 50. Example Where Optimization Must Be Rejected: Blocking

Consider:

```text
CPU:                    25%
Disk IOPS consumed:     30%
Throughput consumed:    20%

Blocked sessions:       High
Lock waits:             High
```

Current infrastructure utilization is artificially low because application requests are blocked.

The workload is not exercising the hardware normally.

Database Chex should defer infrastructure rightsizing until blocking is resolved.

---

# 51. Automation Eligibility

Not every environment should be eligible for autonomous optimization.

Initial exclusions should include environments such as:

```text
complex SQL Server Failover Cluster Instances
shared-disk clustering
third-party replication products
unknown Storage Spaces configuration
unsupported storage topology
multiple SQL instances sharing the same volumes
incomplete Azure telemetry
incomplete SQL telemetry
unknown workload cycles
regular dependency on disk bursting
active severe performance issues
database files stored in unsupported or unusual configurations
systems requiring manual outage coordination
```

Database Chex should explicitly state:

> Savings opportunity detected, but this environment is not eligible for automated remediation.

That behavior increases customer trust.

---

# 52. Recommended Product Automation Levels

Database Chex should support progressively stronger remediation modes.

## Level 1 — Recommendation

Database Chex provides:

```text
Root-cause analysis
Required capacity
Recommended configuration
Expected headroom
Expected monthly savings
Risk assessment
```

The customer performs the change.

---

## Level 2 — Guided Automation

Database Chex:

```text
creates remediation plan
runs safety checks
requests customer approval
executes change
validates workload
rolls back automatically if needed
```

This is the recommended first production automation model.

---

## Level 3 — Policy-Based Autonomous Optimization

Customers define a policy such as:

```text
Allow automatic storage optimization when:

Recommendation confidence >= 95%

Savings >= $100/month

Required headroom >= 20%

Effective predicted headroom >= 25%

No outage required

Rollback supported

No known database health issue

No competing bottleneck

Shadow validation successful
```

Database Chex can then optimize eligible systems automatically.

---

# 53. Proposed Logical Architecture

```text
                    DATABASE CHEX
                          │
                          ▼
                Telemetry Collection
                          │
       ┌──────────────────┼──────────────────┐
       │                  │                  │
       ▼                  ▼                  ▼
 Azure Monitor       SQL Server          Windows
    Metrics           Telemetry          Telemetry
       │                  │                  │
       └──────────────────┼──────────────────┘
                          │
                          ▼
                Workload Normalization
                          │
                          ▼
                 Topology Discovery
                          │
                          ▼
                Bottleneck Diagnosis
                          │
              ┌───────────┴───────────┐
              │                       │
              ▼                       ▼
      Storage Requirement       Alternative Cause
              │                       │
              ▼                       ▼
        Demand Modeling            Explain
              │
              ▼
         Growth Forecast
              │
              ▼
       Candidate Generator
              │
              ▼
        Constraint Solver
              │
              ▼
         Cost Calculator
              │
              ▼
        Confidence Engine
              │
              ▼
         Shadow Validator
              │
              ▼
          Policy Engine
              │
              ▼
        Change Orchestrator
              │
              ▼
        Validation Engine
             / \
            /   \
           ▼     ▼
        SUCCESS  FAILURE
           │       │
           ▼       ▼
         COMMIT  ROLLBACK
```

---

# 54. Recommended Service Separation

The major platform capabilities should remain independent.

## Diagnosis Engine

Example output:

```json
{
  "primaryCause": "storage_iops_constraint",
  "confidence": 0.93,
  "competingCauses": {
    "memoryPressure": 0.08,
    "queryRegression": 0.04,
    "blocking": 0.02
  }
}
```

---

## Rightsizing Engine

Example:

```json
{
  "requiredIops": 9400,
  "requiredThroughputMBps": 420,
  "requiredCapacityGiB": 780,
  "minimumHeadroom": 0.20
}
```

---

## Optimization Engine

Example:

```json
{
  "targetIops": 12000,
  "targetThroughputMBps": 550,
  "predictedIopsHeadroom": 0.217,
  "predictedThroughputHeadroom": 0.236,
  "estimatedMonthlySavings": 287
}
```

---

## Risk Engine

Example:

```json
{
  "automationEligible": true,
  "confidence": 0.96,
  "rollbackSupported": true,
  "riskClassification": "LOW"
}
```

---

## Change Orchestrator

Responsibilities include:

```text
pre-change validation
Azure API change
state tracking
post-change monitoring
rollback execution
audit logging
```

Separating these concerns makes the architecture easier to test and extend.

---

# 55. Policy Engine

The policy engine should allow organizations to define their own operational tolerance.

Example:

```text
Minimum headroom:               20%
Minimum recommendation score:   90
Automatic-change threshold:     95
Minimum monthly savings:       $100
Required telemetry window:      30 days
Shadow-validation period:        7 days
Rollback monitoring period:      2 hours
Maintenance window required:    Yes
```

Different customers can therefore use the same intelligence engine while applying different levels of operational control.

---

# 56. Change Evidence Package

Every recommendation and remediation should produce an immutable evidence record.

Recommended fields include:

```text
Change ID
Customer
Subscription
Resource Group
VM
SQL Server instance
Database/storage mapping
Disk resource ID
LUN
Original configuration
Recommended configuration

Observation start
Observation end
Observed workload cycles

P50 IOPS
P95 IOPS
P99 IOPS
P99.9 IOPS

P50 throughput
P95 throughput
P99 throughput
P99.9 throughput

Disk latency
Queue depth
Disk limits
VM limits
Burst behavior

SQL wait analysis
Query Store analysis
Memory analysis
CPU analysis
Blocking analysis

Root-cause conclusion
Root-cause confidence

Required headroom
Predicted headroom

Cost before
Cost after
Expected monthly savings
Expected annual savings

Policy evaluation
Approval information
Execution information

Before-change baseline
After-change baseline

Validation result
Rollback status
Final configuration
```

This evidence is important for:

```text
customer trust
auditing
support
change management
incident investigation
compliance
model improvement
```

---

# 57. Customer-Facing Recommendation Design

The customer should not need to interpret raw Azure counters.

A strong Database Chex recommendation might read:

> **Storage Performance Is Overprovisioned**
>
> Database Chex analyzed 45 days of SQL Server and Azure infrastructure telemetry, including normal business traffic, nightly backups, weekly index maintenance, and month-end processing.
>
> The current data disk is configured for 20,000 IOPS and 750 MB/s throughput.
>
> The highest representative workload observed required approximately 8,600 IOPS and 410 MB/s.
>
> Database Chex recommends 12,000 IOPS and 550 MB/s.
>
> At the observed peak, the recommended configuration provides approximately 28% IOPS headroom and 25% throughput headroom.
>
> Database Chex also evaluated CPU utilization, SQL Server memory pressure, blocking, Query Store regressions, SQL Server I/O waits, VM-level I/O throttling, disk latency, and burst-credit usage. No competing bottleneck was found that would materially suppress or distort current storage demand.
>
> The recommended change is expected to reduce Azure storage cost by approximately $X per month.
>
> Database Chex can apply the performance change, continuously validate SQL Server and workload performance, and automatically restore the previous configuration if predefined performance thresholds are exceeded.

This communicates the important point:

> Database Chex is not recommending a cheaper disk because utilization appears low. It is recommending a cheaper configuration because it has built evidence that the workload does not require the existing configuration.

---

# 58. Rollout Strategy

The capability should be introduced incrementally.

| Phase | Capability                                      | Relative Risk |
| ----- | ----------------------------------------------- | ------------- |
| 1     | Detect overprovisioned storage                  | Low           |
| 2     | Diagnose root cause                             | Low           |
| 3     | Calculate required performance                  | Low           |
| 4     | Estimate cost savings                           | Low           |
| 5     | Shadow-test recommended configuration           | Low           |
| 6     | Customer-approved performance adjustment        | Medium-Low    |
| 7     | Automatic rollback                              | Medium-Low    |
| 8     | Policy-based automatic performance optimization | Medium        |
| 9     | VM rightsizing                                  | Medium-High   |
| 10    | Disk-capacity migration                         | High          |
| 11    | Automated storage topology restructuring        | Very High     |

This progression allows Database Chex to collect operational evidence and customer trust before introducing higher-risk remediation.

---

# 59. Recommended Initial Scope

The safest first version should focus on cases where:

```text
Azure topology is fully discovered

SQL Server telemetry is complete

At least one representative workload cycle exists

No significant active SQL Server health problem exists

Storage demand is stable

The proposed change does not move SQL files

The proposed change does not resize the filesystem

The proposed change does not require VM replacement

The storage technology allows performance adjustment

Rollback is simple and deterministic
```

Premium SSD v2 performance rightsizing is therefore an especially attractive initial automation scenario because IOPS and throughput can be adjusted independently and programmatically. ([Microsoft Learn][6])

---

# 60. Core Automation Decision

At the highest level, Database Chex should only automatically execute when the following logical expression evaluates to true:

```text
AUTOMATE =

TelemetryComplete

AND

WorkloadCyclesCovered

AND

TopologyKnown

AND

NoSignificantQueryRegression

AND

NoSignificantMemoryPressure

AND

NoSignificantBlocking

AND

NoCPUConstraintMaskingDemand

AND

NoVMIOConstraintMaskingDemand

AND

NoMaterialBurstDependency

AND

CandidateMeetsIOPSRequirement

AND

CandidateMeetsThroughputRequirement

AND

CandidateMeetsCapacityRequirement

AND

CandidateMeetsVMResourceRequirements

AND

EffectiveHeadroom >= 20%

AND

SavingsAreMaterial

AND

RecommendationConfidence >= PolicyThreshold

AND

ShadowValidationPassed

AND

RollbackAvailable

AND

PreChangeSafetyChecksPassed
```

Only then:

```text
Execute Optimization
```

After execution:

```text
IF PerformanceWithinPolicy
    COMMIT
ELSE
    ROLLBACK
```

This should become the fundamental operational contract of the Database Chex optimization engine.

---

# 61. Product Differentiation

Many cloud-management tools can observe:

```text
Provisioned IOPS
Actual IOPS
Disk size
Azure price
```

and conclude:

> This resource appears underutilized.

That alone is not particularly difficult.

Database Chex has an opportunity to provide something considerably more valuable.

The platform can understand:

```text
why SQL Server is performing physical I/O
whether that I/O is healthy
whether memory pressure is influencing it
whether Query Store identifies a regression
whether blocking is suppressing demand
whether CPU saturation is masking demand
whether the disk is capped
whether the VM is capped
whether Azure cache changes the picture
whether burst performance is being relied on
whether important workload cycles have been observed
whether the proposed configuration preserves performance
whether the workload remained healthy after the change
```

The differentiated customer message becomes:

> Database Chex does not simply identify unused Azure resources. It determines the SQL Server workload's actual infrastructure requirement, explains the evidence behind that conclusion, calculates the safest lower-cost configuration, validates it against real workload behavior, and protects the customer with automated rollback.

That is a much stronger value proposition.

---

# 62. Strategic Architecture Direction

The long-term capability should not be described internally as merely:

> Storage downsizing.

A more accurate architecture is:

> **Evidence-Based SQL Server Infrastructure Optimization**

Storage becomes the first optimization domain.

The same framework can later be applied to:

```text
Disk IOPS
Disk throughput
Disk type
Storage topology
VM size
CPU
Memory
tempdb placement
SQL configuration
backup infrastructure
HA/DR infrastructure
```

The optimization lifecycle remains consistent:

```text
Observe
    ↓
Diagnose
    ↓
Understand Demand
    ↓
Predict Future Requirement
    ↓
Generate Candidate
    ↓
Calculate Cost
    ↓
Evaluate Risk
    ↓
Simulate
    ↓
Execute
    ↓
Validate
    ↓
Rollback or Commit
```

---

# 63. Conclusion

Safely automating storage optimization for SQL Server on Azure virtual machines requires substantially more than measuring unused IOPS.

SQL Server performance is influenced by the interaction of:

```text
queries
execution plans
memory
blocking
CPU
SQL Server configuration
Windows
Azure VM limits
managed disks
host caching
disk bursting
storage topology
maintenance activity
workload growth
```

Database Chex should therefore implement storage optimization as a controlled, evidence-based performance process.

The recommended architecture:

1. Collects SQL Server, Windows, Azure VM, and managed-disk telemetry.
2. Maps SQL Server files to the underlying Azure storage topology.
3. Determines whether storage is actually responsible for the observed behavior.
4. Rules out memory, CPU, blocking, query, configuration, cache, and VM-level constraints.
5. Builds a statistical workload performance envelope.
6. Accounts for workload cycles, growth, cold-cache behavior, and burst dependency.
7. Calculates required IOPS, throughput, and capacity independently.
8. Applies at least 20% performance headroom.
9. Generates technically valid Azure configurations.
10. Selects a lower-cost configuration only when all performance constraints are met.
11. Shadow-tests the recommendation against live telemetry.
12. Applies strong pre-change safety gates.
13. Executes only under customer-approved policy.
14. Validates SQL Server and workload performance after the change.
15. Automatically restores the previous configuration when validation fails.
16. Records the complete evidence and decision trail.

The result is not simply an Azure cost recommendation.

It is a defensible statement that:

> **Database Chex has sufficient evidence that the SQL Server workload can operate on a lower-cost Azure configuration, has preserved the required performance safety margin, has ruled out competing causes that could distort the result, and can automatically protect the workload if the prediction proves incorrect.**

That is the level of confidence required before database infrastructure optimization should become autonomous.

---

# Microsoft Technical References

The solution architecture and Azure-specific behavior described in this article are aligned with current Microsoft guidance covering SQL Server on Azure VM performance baselining, Azure VM and managed-disk metrics, Azure managed-disk storage configuration, Premium SSD v2 behavior, and managed-disk resizing. ([Microsoft Learn][1])

Microsoft currently documents Premium SSD v2 as supporting granular configuration of capacity, IOPS, and throughput, with performance changes available programmatically and limited to a defined number of adjustments per 24-hour period. ([Microsoft Learn][6])

Microsoft also documents that Azure managed disks cannot be directly reduced in size; decreasing data-disk capacity requires creating a smaller disk and migrating data to it. ([Microsoft Learn][9])

Premium SSD v2 billing is based on provisioned capacity together with configured IOPS and throughput above the included baseline levels, making performance rightsizing directly relevant to cost optimization. ([Microsoft Learn][7])

[1]: https://learn.microsoft.com/en-us/azure/azure-sql/virtual-machines/windows/performance-guidelines-best-practices-collect-baseline?view=azuresql "Collect Baseline: Performance Best Practices & Guidelines - SQL Server on Azure VMs | Microsoft Learn"
[2]: https://learn.microsoft.com/en-us/azure/azure-monitor/reference/supported-metrics/microsoft-compute-virtualmachines-metrics?utm_source=chatgpt.com "Supported metrics - Microsoft.Compute/virtualMachines - Azure Monitor | Microsoft Learn"
[3]: https://learn.microsoft.com/en-us/azure/virtual-machines/disks-metrics?utm_source=chatgpt.com "Disk metrics - Azure Virtual Machines | Microsoft Learn"
[4]: https://learn.microsoft.com/th-th/azure/azure-sql/virtual-machines/Windows/performance-guidelines-best-practices-checklist?view=azuresql "Checklist: Best Practices and Guidelines - SQL Server on Azure VMs | Microsoft Learn"
[5]: https://learn.microsoft.com/en-us/azure/virtual-machines/disks-types?utm_source=chatgpt.com "Select a disk type for Azure IaaS VMs - managed disks - Azure Virtual Machines | Microsoft Learn"
[6]: https://learn.microsoft.com/en-us/azure/virtual-machines/disks-deploy-premium-v2?utm_source=chatgpt.com "Deploy a Premium SSD v2 managed disk - Azure Virtual Machines | Microsoft Learn"
[7]: https://learn.microsoft.com/en-us/azure/virtual-machines/disks-understand-billing?utm_source=chatgpt.com "Understand Azure Disk Storage billing - Azure Virtual Machines | Microsoft Learn"
[8]: https://learn.microsoft.com/en-us/azure/virtual-machines/disks-performance-options?utm_source=chatgpt.com "Overview - Options to improve Azure managed disk performance - Azure Virtual Machines | Microsoft Learn"
[9]: https://learn.microsoft.com/en-us/troubleshoot/azure/virtual-machines/windows/downsize-data-disk-without-losing-data?utm_source=chatgpt.com "Decrease the size of an Azure data disk without losing data - Virtual Machines | Microsoft Learn"
