Scenario/Problem:

Below if the problem that I would like your thoughts about.  

Please let me know if you have any questions. 

What Database Chex Is Database Chex is a database confidence platform for IT teams responsible for Azure SQL Database and SQL Server running on Azure virtual machines, especially when they do not have dedicated DBA expertise. 

It translates database and infrastructure telemetry into clear health assessments, prioritized risks, explanations, and recommended actions across availability, performance, security, and cost. 

Cost Optimization Opportunity SQL Server on Azure VMs is frequently overprovisioned because reducing disk capacity, IOPS, throughput, or VM size feels risky. 

Database Chex has an opportunity to identify when customers are paying for unused storage performance and recommend a lower-cost configuration without compromising the workload. 

The challenge: 

How can Database Chex safely automate changes to the storage configuration for SQL Server on an Azure VM while maintaining at least 20% headroom, validating that the change will reduce cost without affecting performance, and confirming that the original issue was caused by storage rather than queries, memory pressure, blocking, SQL Server configuration, caching, or VM-level limits?

==================================================================================================
Analysis:

The key design principle is that Database Chex should **not treat this as a simple rightsizing problem**. It should treat it as a controlled database performance experiment.

For SQL Server on Azure VMs, storage cost, SQL Server behavior, VM limits, caching, memory, and query patterns are tightly connected. A disk that appears underutilized today may still be protecting the workload from intermittent peaks, ETL windows, checkpoints, backups, index maintenance, failovers, or cache misses. Microsoft itself recommends measuring peak CPU, memory, IOPS, throughput, latency, and SQL Server wait statistics before choosing or changing the infrastructure configuration. ([Microsoft Learn][1])

My recommended architecture would look like this.

## 1. Start with the question Database Chex actually needs to answer

Database Chex should not initially ask:

> "Can I reduce this disk from 20,000 IOPS to 10,000 IOPS?"

It should ask:

> "What performance envelope does this SQL Server workload actually require, what resource is currently limiting it, and what is the cheapest Azure configuration that can satisfy that envelope with an acceptable safety margin?"

That distinction matters.

The system needs to independently determine four things:

1. **Observed workload demand**
2. **Actual bottleneck**
3. **Required future capacity including headroom**
4. **Azure configuration capable of supplying that capacity**

Only then should it calculate savings.

---

# 2. Separate "storage utilization" from "storage bottleneck"

One of the biggest architectural mistakes would be saying:

> Disk is provisioned for 20,000 IOPS, workload normally uses 5,000 IOPS, therefore reduce it to 7,500.

That is insufficient.

SQL Server might currently be using only 5,000 IOPS because:

* SQL Server is CPU constrained.
* queries are blocked.
* memory pressure is suppressing throughput.
* the VM itself is throttling I/O.
* the disk is already experiencing latency.
* SQL Server's buffer pool is hiding physical reads.
* Azure host cache is servicing a large percentage of reads.
* a bad execution plan has suddenly increased physical I/O.
* an ETL workload only appears once per week.
* backups or maintenance run monthly.
* disk bursting temporarily masks undersizing.

Therefore Database Chex needs a **causal diagnosis layer before a rightsizing layer**.

---

# 3. Think of the system as four diagnostic layers

I would model the telemetry like this:

```text
Application / Queries
        ↓
SQL Server Engine
        ↓
Windows / OS
        ↓
Azure VM
        ↓
Azure Managed Disk
```

A performance symptom can originate at any one of these layers.

Database Chex should correlate evidence across them rather than looking at Azure disk metrics by themselves.

---

# 4. Layer 1 — prove whether SQL Server is waiting on storage

This is probably the most important step.

SQL Server wait statistics give you evidence of what SQL Server threads are actually waiting for.

For storage diagnosis, important waits include:

* `PAGEIOLATCH_SH`
* `PAGEIOLATCH_EX`
* `PAGEIOLATCH_UP`
* `WRITELOG`

`PAGEIOLATCH_*` indicates SQL Server waiting for database pages to be read from storage, while `WRITELOG` commonly indicates waiting for log flushes. Microsoft specifically identifies these as important I/O-related waits. ([Microsoft Learn][2])

But Database Chex should never conclude:

> PAGEIOLATCH = slow disk.

Because a badly written query scanning 500 GB may generate enormous I/O against perfectly healthy storage.

Instead, ask:

```text
Are SQL requests waiting on I/O?
            +
Is physical disk latency elevated?
            +
Is disk/VM throttling occurring?
            +
Is the I/O volume reasonable for the workload?
```

When those signals line up, storage becomes a strong causal candidate.

---

# 5. Layer 2 — determine whether the disk itself is throttling

Azure already exposes excellent signals for this.

Database Chex should collect at minimum, per LUN:

* Data Disk IOPS Consumed Percentage
* Data Disk Bandwidth Consumed Percentage
* Data Disk Latency
* Data Disk Queue Depth
* Data Disk Read Operations/sec
* Data Disk Write Operations/sec
* Data Disk Read Bytes/sec
* Data Disk Write Bytes/sec
* Data Disk Target IOPS
* Data Disk Target Bandwidth
* burst credit metrics

Azure specifically documents `Data Disk IOPS Consumed Percentage` and `Data Disk Bandwidth Consumed Percentage` as disk-capping signals. At or around 100%, the disk is hitting its provisioned limit. ([Microsoft Learn][3])

Database Chex therefore wants patterns such as:

```text
Disk IOPS consumed        98%
Disk latency              18 ms
Queue depth               rising
PAGEIOLATCH waits         rising
```

That is strong evidence of a disk IOPS problem.

Similarly:

```text
Bandwidth consumed        99%
IOPS consumed             47%
Latency                    22 ms
PAGEIOLATCH               high
```

suggests a throughput constraint rather than IOPS.

This distinction is important because reducing IOPS might still be safe when throughput is the true requirement, or vice versa.

---

# 6. Layer 3 — determine whether the VM is actually the limiter

Every Azure VM SKU has its own storage limits.

Even if the attached disks could theoretically supply 50,000 IOPS, the VM may only allow 25,000 uncached IOPS.

Azure therefore exposes:

* VM Uncached IOPS Consumed Percentage
* VM Uncached Bandwidth Consumed Percentage
* VM Cached IOPS Consumed Percentage
* VM Cached Bandwidth Consumed Percentage

These are separate from disk-level limits. Azure explicitly distinguishes disk capping from VM-level I/O capping. ([Microsoft Learn][4])

This lets Database Chex distinguish:

### Situation A

```text
Disk IOPS consumed:       45%
VM uncached IOPS:         100%
Latency:                  high
```

Conclusion:

> Do not modify the disk. The VM storage envelope is the bottleneck.

### Situation B

```text
Disk IOPS consumed:       100%
VM uncached IOPS:         55%
Latency:                  high
```

Conclusion:

> Individual disk/storage configuration is limiting the workload.

That difference is extremely valuable for Database Chex because it prevents a bad recommendation.

---

# 7. Layer 4 — prove that another SQL Server problem isn't creating the symptom

Before recommending a storage change, Database Chex should run competing diagnoses.

I would build these as "alternative hypothesis scores."

For example:

| Candidate cause | Evidence                                                     |
| --------------- | ------------------------------------------------------------ |
| Storage         | disk latency, PAGEIOLATCH, WRITELOG, disk throttling         |
| Memory          | PLE collapse, memory grants pending, low available memory    |
| Query           | large logical/physical reads, Query Store regression         |
| Blocking        | blocked request duration, lock waits                         |
| CPU             | CPU saturation, SOS_SCHEDULER_YIELD                          |
| TempDB          | PAGELATCH waits, tempdb file contention                      |
| VM I/O limit    | VM cached/uncached IOPS or bandwidth throttling              |
| Cache behavior  | host cache hit/miss behavior                                 |
| SQL config      | max server memory, MAXDOP, file layout, growth configuration |

Then produce something like:

```text
Root Cause Confidence

Storage bottleneck          84%
Query inefficiency          18%
Memory pressure             12%
Blocking                     4%
CPU pressure                 3%
```

These do not necessarily need to sum to 100 because several conditions can coexist.

I would avoid fully automated storage changes unless the storage confidence exceeds some threshold, perhaps:

```text
Storage root cause confidence >= 80%
AND
No competing cause > 30%
```

The exact thresholds should eventually be learned from production experience.

---

# 8. Query Store should be a major input

Database Chex should correlate infrastructure changes with Query Store.

Imagine this:

```text
Monday
Disk reads: 4,000 IOPS

Tuesday
Disk reads: 14,000 IOPS
```

An infrastructure-only system might conclude:

> We now need 14,000 IOPS.

But Query Store might reveal that one query changed plans Tuesday morning and started performing a table scan.

Database Chex should detect:

```text
Query 194

Before plan change:
logical reads: 20,000

After plan change:
logical reads: 8,700,000
```

Then the recommendation becomes:

> Fix the query or plan regression.

Not:

> Buy more storage.

This is where Database Chex can differentiate itself from generic Azure rightsizing products.

---

# 9. Memory must be considered before storage downsizing

SQL Server uses memory aggressively for caching database pages.

Suppose:

```text
Database size:      2 TB
SQL max memory:     128 GB
Physical reads:     high
```

A lack of buffer pool memory could create high physical I/O.

Increasing memory might actually decrease storage demand dramatically.

Conversely, historical IOPS might look artificially low because most frequently accessed pages are cached.

Then after:

* restart
* failover
* patching
* VM maintenance
* cache eviction

the database suddenly generates far more storage reads.

Therefore Database Chex should evaluate two modes:

### Warm-cache workload

Normal steady-state behavior.

### Cold-cache workload

Behavior after restart/failover/cache eviction.

This should affect headroom.

---

# 10. Headroom should not simply mean average × 1.20

The user requirement says maintain at least 20% headroom.

I would define that mathematically as:

```text
Provisioned capacity >= Required workload / 0.80
```

So if observed required IOPS is:

```text
12,000 IOPS
```

minimum target capacity is:

```text
12,000 / 0.80
= 15,000 IOPS
```

That means utilization at the design peak is 80%, leaving 20% of provisioned capacity unused.

For throughput:

```text
Observed peak = 400 MB/s

Required target:
400 / 0.80
= 500 MB/s
```

Do this independently for:

* IOPS
* throughput
* storage capacity
* VM storage limits
* possibly CPU and memory when resizing the VM.

---

# 11. But "observed peak" needs a better definition

Do not use maximum.

One rogue one-minute event could ruin the model.

Do not use average either.

Average hides production peaks.

I would use several percentiles.

For example:

```text
P50
P95
P99
P99.9
max
```

and segment the workload by time period.

For example:

```text
Normal business hours
Nightly batch
Weekend ETL
Month-end
Backup window
Index maintenance
```

A possible safe sizing rule:

```text
Required IOPS =
max(
    P99.5 normal workload,
    P99 maintenance window,
    P99 batch workload
)
```

Then:

```text
Provisioned IOPS =
Required IOPS / 0.80
```

You can make this even safer by incorporating growth.

For example:

```text
Forecast IOPS =
current requirement × projected 90-day growth
```

Then calculate headroom against the forecast rather than today's workload.

---

# 12. Observation windows should depend on workload periodicity

Seven days may be enough for some customers.

It may be completely wrong for others.

A SQL Server workload could have:

* month-end close
* payroll every two weeks
* Monday morning batch
* quarterly financial processing
* nightly backups
* weekly index maintenance

Database Chex therefore needs workload-cycle awareness.

I would recommend something like:

```text
Minimum: 7 days

Preferred:
30 days for business systems

High confidence:
45–90 days where historical metrics exist
```

But the rule should be:

> Do not automate downsizing unless the observation period includes known peak workload cycles.

Database Chex could explicitly tell the user:

> "We have 11 days of telemetry, but month-end processing has not yet occurred. Recommendation confidence is currently Medium."

That is much better than pretending certainty.

---

# 13. Storage needs to be modeled by role

Never evaluate every attached disk identically.

Database Chex should first map every SQL file to physical storage.

Conceptually:

```text
Database → file → Windows volume → storage pool → physical disk → Azure LUN
```

Then categorize:

```text
DATA
LOG
TEMPDB
BACKUP
OS
```

This matters because their I/O patterns are different.

Microsoft recommends separating data, log, and `tempdb`, and generally recommends read-only host caching for SQL data disks and no host caching for log disks. ([Microsoft Learn][5])

Therefore Database Chex could say:

```text
LUN 2
Role: SQL Data
Read-heavy
Host caching: ReadOnly

LUN 3
Role: Transaction Log
Write-heavy
Host caching: None

D:
Role: tempdb
Local ephemeral storage
```

Without this mapping, automated storage changes are dangerous.

---

# 14. Account for Storage Spaces / disk striping

SQL Server Azure VMs frequently stripe disks.

Suppose:

```text
4 × Premium SSD
```

appear as:

```text
F:\ SQLData
```

Database Chex cannot optimize one LUN independently without understanding the Storage Spaces layout.

It should capture:

* pool membership
* stripe width
* interleave
* disk count
* capacity
* filesystem allocation unit
* drive-to-LUN relationship.

Microsoft recommends Storage Spaces when multiple Azure disks are used to increase bandwidth toward the VM's aggregate I/O limit. ([Microsoft Learn][5])

So optimization happens at the **storage pool level**, not just disk level.

---

# 15. Caching makes rightsizing harder

Suppose a Premium SSD can deliver:

```text
5,000 uncached IOPS
```

but SQL Server data disks have host read caching enabled.

SQL might effectively receive much more read performance because many reads come from Azure BlobCache.

Microsoft notes that cached reads operate against separate VM cached limits rather than the VM's uncached storage envelope. ([Microsoft Learn][5])

Therefore Database Chex should maintain separate models for:

```text
Cached reads
Uncached reads
Writes
```

Otherwise you can make subtle errors.

For example:

```text
SQL workload:
18,000 total read IOPS

Azure disk:
5,000 uncached IOPS

Cache:
14,000 cached read IOPS
```

The disk isn't undersized.

The cache is doing exactly what it is supposed to do.

---

# 16. Burst performance needs special treatment

Azure disks and VM SKUs can temporarily burst.

That is dangerous for rightsizing.

Imagine:

```text
Normal demand:        4,000 IOPS
Peak demand:          8,000 IOPS
Base disk:            5,000 IOPS
Burst capability:     10,000 IOPS
```

The workload looks healthy until burst credits are exhausted.

Therefore the baseline requirement should be calculated against **sustainable capacity**, not temporary burst capacity.

Bursting should be treated as safety margin, not planned baseline performance.

Database Chex should track:

* burst credits remaining/used
* duration of bursts
* peak frequency
* dependency on bursting.

If the application regularly depends on disk bursting, I would mark the system as:

```text
Not eligible for automatic downsizing.
```

---

# 17. Capacity is different from performance

Three independent storage constraints matter:

```text
Capacity
IOPS
Throughput
```

Database Chex should calculate each separately.

For example:

```text
Used space:         800 GB
Projected growth:   80 GB/year
IOPS needed:        6,000
Throughput needed:  350 MB/s
```

The cheapest configuration has to satisfy all three simultaneously.

---

# 18. Important Azure limitation: you cannot simply shrink a managed disk

This is one of the most important implementation details.

Azure does **not** support directly decreasing the size of a managed disk. Microsoft explicitly documents that managed disks cannot be downsized in place. ([Microsoft Learn][6])

The supported conceptual workflow for reducing capacity is:

```text
Create smaller disk
        ↓
Copy/migrate data
        ↓
Validate
        ↓
Cut over
        ↓
Detach old disk
        ↓
Delete old disk later
```

Microsoft documents this approach for downsizing Azure data disks. ([Microsoft Learn][7])

This means Database Chex should distinguish two optimization classes.

---

# 19. Class A — performance-only downsizing

This is the easiest and safest optimization.

Example:

```text
Premium SSD v2

Current:
20,000 IOPS
800 MB/s

Recommended:
12,000 IOPS
500 MB/s
```

Premium SSD v2 lets IOPS and throughput be adjusted independently, and those settings directly affect cost. Microsoft currently allows performance adjustments up to four times within a 24-hour period. ([Microsoft Learn][8])

This is ideal for automation.

You are not touching:

* filesystem
* database files
* drive letters
* volumes
* data.

You're just changing the Azure disk's performance envelope.

This should probably be **Phase 1 of Database Chex automation**.

---

# 20. Class B — disk capacity downsizing

Much riskier.

Because Azure can't shrink the existing managed disk, Database Chex has to orchestrate migration.

For example:

```text
Current:
P40 2 TB

Actual data:
600 GB

Recommended:
1 TB disk
```

Database Chex would need to:

```text
Provision replacement disk
→ initialize Windows disk
→ format
→ construct Storage Spaces if needed
→ copy database files
→ quiesce SQL Server
→ final sync
→ detach old storage
→ attach/swap drive mapping
→ restart SQL
→ verify DB recovery
→ verify SQL file locations
→ validate performance
→ retain old disk for rollback
```

That moves Database Chex from a monitoring platform into a production change orchestration platform.

The risk profile is completely different.

I would therefore delay automated capacity downsizing until the performance-tier automation has proven reliable.

---

# 21. VM downsizing is another separate operation

The same applies to VM sizing.

A smaller VM may reduce cost but also changes:

* CPU
* memory
* max attached disks
* cached IOPS
* uncached IOPS
* disk bandwidth
* temporary SSD size
* network bandwidth.

This creates coupling.

For example:

```text
Current VM:
E32ds_v5

Proposed:
E16ds_v5
```

CPU may be fine.

Memory may be fine.

But perhaps the E16's uncached storage limit is below the workload requirement.

Database Chex therefore needs a multi-resource feasibility test:

```text
CPU        PASS
Memory     PASS
Network    PASS
Disk count PASS
IOPS       FAIL
Throughput PASS
TempDB     PASS
```

Result:

```text
VM resize rejected.
```

Azure's SQL VM guidance specifically recommends sizing the VM against measured CPU, memory, IOPS, throughput, and peak workload requirements. ([Microsoft Learn][1])

---

# 22. Build a "candidate configuration solver"

Rather than hardcoding:

> Reduce by 20%.

I would create an optimization engine.

Inputs:

```text
Observed workload envelope
Growth forecast
Required 20% headroom
Current topology
Region
VM family
Disk availability
Disk type
Caching requirements
Storage Spaces topology
Availability requirements
Customer restrictions
```

Candidates could include:

```text
Keep same disk, lower IOPS
Keep same disk, lower throughput
Move Premium SSD → Premium SSD v2
Change number of disks
Change VM SKU
Change VM + disk configuration
```

The solver rejects any candidate violating a constraint.

Conceptually:

```text
Find configuration C

minimize:
monthly_cost(C)

subject to:

IOPS(C)       >= required_IOPS / 0.80
Throughput(C) >= required_MBps / 0.80
Capacity(C)   >= required_capacity / 0.80

Memory(C) >= required_memory
CPU(C) >= required_CPU

VM_IOPS(C) >= storage requirement
VM_MBps(C) >= storage requirement

disk_count(C) <= VM_max_disks

availability(C) >= current SLA

SQL storage best practices = satisfied
```

That turns the feature into an optimization problem instead of a collection of rules.

---

# 23. Calculate cost from actual Azure billing dimensions

For Premium SSD v2, cost is particularly suitable for this because Azure bills based on capacity plus configured IOPS and throughput. ([Microsoft Learn][9])

So Database Chex could show:

```text
Current disk:

Capacity:         1,024 GiB
Provisioned IOPS: 20,000
Throughput:       800 MB/s
Monthly cost:     $X


Recommended:

Capacity:         unchanged
Provisioned IOPS: 12,000
Throughput:       450 MB/s
Monthly cost:     $Y

Savings:
$Z/month
$Z×12/year
```

I would require minimum savings before automation.

For example:

```text
Savings < $20/month:
recommendation suppressed

Savings 2%:
recommendation informational only
```

The risk of production change needs to justify the savings.

---

# 24. Introduce a confidence score

Every recommendation should carry a confidence score.

Something like:

```text
Confidence =

Data completeness
× workload coverage
× diagnosis confidence
× prediction confidence
× configuration certainty
```

Example:

```text
Observation period        30/30
Month-end included        Yes
Telemetry gaps            None
Storage root cause        91%
Candidate headroom        31%
Growth stability          High
Query regressions         None

Recommendation confidence: 94 / 100
```

Then automation policy:

```text
95–100 → eligible for autonomous execution

85–94 → approval required

70–84 → recommendation only

<70 → insufficient evidence
```

That would make Database Chex much safer operationally.

---

# 25. The 20% headroom should be a floor, not always the target

For stable OLTP:

```text
20–25%
```

may be reasonable.

For bursty workloads:

```text
30–40%
```

might be appropriate.

For unpredictable systems:

```text
50%+
```

could be necessary.

So I would implement:

```text
Effective headroom =
max(
    20%,
    volatility buffer,
    growth buffer,
    workload-specific buffer
)
```

Example:

```text
Minimum policy:           20%
Workload volatility:      +8%
Growth forecast:          +5%

Effective target:          33%
```

That still satisfies the "at least 20%" requirement while avoiding false precision.

---

# 26. Use a shadow recommendation before touching anything

The safest automation architecture would have multiple maturity stages.

### Stage 0 — Observe

Database Chex records:

```text
Current provisioned IOPS = 20,000
Recommended = 12,000
```

but makes no change.

For the next 14 days, continually test:

> Would the recommended configuration have violated headroom?

You are effectively running a "digital twin" of the proposed storage configuration.

Example:

```text
Proposed limit: 12,000 IOPS
Safe workload ceiling at 20% headroom: 9,600 IOPS

Observed peak after recommendation:
8,400

Result:
candidate remains safe
```

But if three days later:

```text
Observed peak: 10,800
```

candidate is rejected automatically.

This shadow-mode concept would greatly reduce risk.

---

# 27. Then use a canary change where possible

For performance-adjustable disks such as Premium SSD v2, don't immediately drop from:

```text
20,000 → 10,000
```

You can walk down gradually.

Example:

```text
20,000
↓
17,000
↓
14,000
↓
12,000
```

At each step:

```text
observe
validate
continue or rollback
```

Because Premium SSD v2 allows granular IOPS and throughput adjustment without changing capacity, it is especially suitable for controlled rightsizing. ([Microsoft Learn][8])

However, the platform limits how often performance can be adjusted, so Database Chex needs to account for that when planning changes. ([Microsoft Learn][10])

---

# 28. Define explicit pre-change safety gates

Before Database Chex executes anything, I would require:

```text
SQL Agent jobs:
No critical job running

Backups:
No active backup or restore

HA:
Healthy

Database state:
ONLINE

Availability Groups:
Synchronized

CPU:
Below threshold

Memory:
Healthy

Blocking:
No abnormal blocking

Current I/O:
Within normal operating band

Azure status:
Healthy

Snapshot / rollback:
Ready

Maintenance policy:
Satisfied
```

Any failing gate cancels the change.

Not postpones blindly—just marks it as not currently safe.

---

# 29. Establish a baseline immediately before change

Historical baseline is not enough.

Create a near-change baseline:

```text
30–60 minutes before change
```

Capture:

* transactions/sec
* batch requests/sec
* request latency
* query duration
* CPU
* memory
* PLE
* IOPS
* throughput
* disk latency
* queue depth
* wait statistics
* top queries
* blocking
* deadlocks.

Then compare the same metrics after the change.

This creates a clean before/after experiment.

---

# 30. Post-change validation should evaluate performance, not just uptime

A dangerous automation system might say:

```text
SQL Server service is running.
Change successful.
```

That is nowhere near enough.

Database Chex should determine:

```text
Did user-facing workload performance materially degrade?
```

Example validation criteria:

```text
P95 query duration:
must not increase >10%

Disk latency:
must remain below policy

PAGEIOLATCH:
must not materially increase

WRITELOG:
must not materially increase

Disk IOPS utilization:
must remain <=80% design threshold

Disk throughput:
must remain <=80%

VM I/O:
must remain <=80%

Blocking:
no abnormal increase

Errors/timeouts:
no increase

Transaction throughput:
no meaningful decrease
```

That gives you an objective pass/fail test.

---

# 31. Performance equivalence requires workload normalization

This is another subtle point.

Suppose:

```text
Before:
5,000 transactions/min
average query latency 100 ms

After:
3,000 transactions/min
average latency 90 ms
```

It might look like performance improved.

But workload volume dropped.

Database Chex should normalize performance against workload intensity.

For example:

```text
IOPS / batch request
CPU / transaction
latency by Query Store query_id
reads / execution
writes / transaction
```

Compare similar workload periods.

Otherwise you will confuse low traffic with improved performance.

---

# 32. Automatic rollback needs to be first-class

Rollback cannot be an afterthought.

Every automated change should have:

```text
Before state
Target state
Rollback state
Rollback deadline
Rollback trigger
```

For performance-only adjustment:

```text
Old:
20,000 IOPS
800 MB/s

New:
12,000 IOPS
500 MB/s
```

Rollback is:

```text
restore:
20,000
800
```

That is relatively straightforward.

For capacity migration:

```text
old disk must remain intact and attached/detached-but-retained
```

until the validation window is complete.

Never delete it immediately.

---

# 33. Use both instantaneous and sustained rollback triggers

Don't rollback because of one random spike.

Example logic:

```text
Immediate rollback:

database unavailable
disk errors
AG unhealthy
SQL service failure
severe application errors
```

But performance rollback might require persistence:

```text
P95 latency +25%
for >10 minutes

OR

IOPS consumed >90%
for >5 minutes

OR

disk latency > threshold
for >5 minutes

OR

critical SQL waits increase >50%
for >10 minutes
```

The exact numbers should become customer-configurable policies.

---

# 34. Some cases should never be fully autonomous

I would initially exclude:

* FCI
* complex shared storage
* third-party replication
* unusual clustering
* encrypted storage where migration is complex
* unsupported disk layouts
* unknown Storage Spaces configuration
* multi-instance SQL Servers sharing disks
* SQL data on OS disk
* heavily burst-dependent workloads
* incomplete telemetry
* workloads with unknown maintenance cycles
* high-frequency autoscaling environments.

Database Chex should be comfortable saying:

> "Potential savings detected, but the configuration is not eligible for automated remediation."

That's a feature, not a failure.

---

# 35. I would split recommendations into three levels

### Level 1 — Recommend

Example:

> Database Chex detected approximately $310/month of potential storage savings.

Customer makes the change.

### Level 2 — Guided automation

Database Chex:

* builds plan
* runs safety checks
* customer approves
* Database Chex executes
* Database Chex validates
* automatically rolls back if necessary.

This should probably be the first production version.

### Level 3 — Autonomous optimization

Database Chex executes automatically under customer policy.

For example:

```text
Allow automatic changes when:

Confidence >=95
Savings >=$100/month
Headroom >=25%
No outage required
Automatic rollback supported
```

This is where the product can eventually go.

---

# 36. The architecture could look like this

```text
            DATABASE CHEX
                  │
                  ▼
        Telemetry Collection
                  │
     ┌────────────┼─────────────┐
     │            │             │
Azure Monitor   SQL Server    Windows
metrics         telemetry     telemetry
     │            │             │
     └────────────┼─────────────┘
                  ▼
        Workload Normalization
                  │
                  ▼
       Bottleneck Diagnosis
                  │
          ┌───────┴───────┐
          │               │
       Storage?          Other?
          │               │
          ▼               ▼
   Capacity Model      Explain cause
          │
          ▼
   Candidate Generator
          │
          ▼
   Cost Optimization
          │
          ▼
       Risk Engine
          │
          ▼
    Shadow Validation
          │
          ▼
     Approval Policy
          │
          ▼
    Change Orchestrator
          │
          ▼
    Validation Engine
       │          │
     PASS        FAIL
       │          │
       ▼          ▼
    Commit     Rollback
```

---

# 37. Separate diagnosis and remediation services

From an architectural perspective, I would strongly keep these independent.

### Diagnosis Engine

Produces:

```json
{
  "rootCause": "disk_iops_constraint",
  "confidence": 0.91
}
```

### Rightsizing Engine

Produces:

```json
{
  "requiredIops": 9400,
  "requiredThroughputMBps": 420,
  "requiredCapacityGiB": 780
}
```

### Optimization Engine

Produces candidate:

```json
{
  "targetIops": 12000,
  "targetThroughputMBps": 550,
  "headroom": 0.23,
  "estimatedMonthlySavings": 287
}
```

### Risk Engine

Produces:

```json
{
  "automationEligible": true,
  "riskScore": 8,
  "rollbackSupported": true
}
```

### Orchestrator

Executes Azure change.

This separation is important because later Database Chex might support AWS, PostgreSQL, Oracle, Azure SQL MI, etc.

The diagnostic engine shouldn't know how Azure ARM updates disks.

---

# 38. Keep an immutable "change evidence package"

For every recommendation and automated action, store evidence.

Something like:

```text
Change ID
Customer
VM
SQL instance
Disk/LUN
Original configuration
Proposed configuration

Observation period
Workload coverage

P50/P95/P99 IOPS
P50/P95/P99 throughput
Latency
VM limits
Disk limits

Top SQL waits
Top queries
Memory status
CPU status
Blocking status

Root-cause reasoning
Confidence score

Projected savings
Expected headroom

Approval
Execution time

Before metrics
After metrics

Validation result
Rollback status
```

This will be extremely important for enterprise trust, support, auditing, and explaining why Database Chex did something.

---

# 39. The customer-facing explanation matters just as much as the algorithm

Don't show:

> "P99 IOPS = 6,243. P30 threshold = 5,000."

Show:

> **Storage appears overprovisioned**
>
> Your SQL Server data volume is configured for 20,000 IOPS but required no more than 8,900 IOPS during the last 30 days, including nightly backups, weekly maintenance and month-end processing.
>
> Database Chex recommends reducing provisioned performance to 12,000 IOPS. At the observed peak, this leaves approximately 26% headroom.
>
> We also checked CPU, memory pressure, blocking, query regressions, SQL Server I/O waits and VM-level storage throttling. None indicate that another bottleneck is masking storage demand.
>
> Estimated savings: $240/month.
>
> Database Chex can apply the change and automatically restore the current storage configuration if performance degrades.

That is a confidence product.

Not just a monitoring product.

---

# 40. An example of the complete analysis

Imagine:

```text
VM: E32ds_v5
Disk: Premium SSD v2

Current:
IOPS = 20,000
Throughput = 750 MB/s
```

Thirty-day telemetry:

```text
IOPS
P50          2,800
P95          5,700
P99          7,400
P99.9        8,600

Throughput
P50          120 MB/s
P95          290 MB/s
P99          360 MB/s
P99.9        410 MB/s
```

Known peaks included:

```text
Nightly backups      yes
Weekly maintenance   yes
Month-end            yes
```

SQL analysis:

```text
PAGEIOLATCH           normal
WRITELOG              normal
Memory pressure       none
CPU saturation        none
Blocking              normal
Query regressions     none
VM I/O throttling     none
Disk throttling       none
```

Database Chex calculates:

```text
Required IOPS:
8,600 / .80
= 10,750

Required throughput:
410 / .80
= 512.5 MB/s
```

Round safely:

```text
Recommended:

12,000 IOPS
550 MB/s
```

Expected peak utilization:

```text
IOPS:
8,600 / 12,000 = 71.7%

Headroom = 28.3%

Throughput:
410 / 550 = 74.5%

Headroom = 25.5%
```

Pass.

Now Database Chex runs shadow mode for another week.

Maximum observed:

```text
8,750 IOPS
418 MB/s
```

Still safe.

Then it changes Premium SSD v2:

```text
20,000 → 12,000 IOPS
750 → 550 MB/s
```

After change:

```text
P95 query latency         +1.8%
P99 query latency         +2.9%
PAGEIOLATCH               unchanged
WRITELOG                  unchanged
disk latency              unchanged
application errors        unchanged
peak IOPS utilization     73%
peak throughput           76%
```

Database Chex concludes:

```text
Optimization successful.
```

That is a strong automated change.

---

# 41. Contrast that with a case Database Chex should reject

Suppose telemetry says:

```text
Provisioned IOPS        20,000
Observed IOPS            7,000
```

Looks overprovisioned.

But:

```text
SQL memory:
Available memory extremely low

Page Life Expectancy:
frequent collapse

Query Store:
major plan regression

Physical reads:
increased 300%

PAGEIOLATCH:
high
```

Database Chex should say:

> Storage cost optimization is currently blocked. SQL Server is experiencing memory pressure and a query plan regression that are materially affecting physical I/O demand. Resolve those conditions before determining the required storage performance.

That's much more valuable than blindly rightsizing infrastructure.

---

# 42. Another rejected scenario: VM bottleneck

Suppose:

```text
Disk IOPS utilization          55%
VM uncached IOPS utilization  100%
Latency                        high
PAGEIOLATCH                    high
```

Database Chex should say:

> The attached disks are not the primary bottleneck. The Azure VM's storage I/O ceiling is being reached.

This distinction is directly supported by Azure's separate disk-level and VM-level utilization metrics. ([Microsoft Learn][11])

Again, no disk downsizing.

---

# 43. A practical product rollout

If I were architecting this product, I would not start with fully autonomous storage migration.

I would release it in this sequence:

| Phase | Capability                                                 |        Risk |
| ----- | ---------------------------------------------------------- | ----------: |
| 1     | Detect storage overprovisioning                            |         Low |
| 2     | Explain root cause and savings                             |         Low |
| 3     | Simulate target configuration                              |         Low |
| 4     | Customer-approved Premium SSD v2 IOPS/throughput reduction |  Medium-low |
| 5     | Automatic rollback                                         |  Medium-low |
| 6     | Policy-based autonomous performance rightsizing            |      Medium |
| 7     | VM rightsizing                                             | Medium-high |
| 8     | Disk-capacity migration                                    |        High |
| 9     | Fully autonomous topology restructuring                    |   Very high |

That gives Database Chex a path to build customer trust gradually.

---

# 44. The feature's moat is not the cost calculation

Azure already knows:

```text
Configured IOPS
Current IOPS
Disk price
```

A lot of tools can say:

> "You're underusing this disk."

Database Chex's differentiated value is:

> "We understand SQL Server well enough to tell you **why** the I/O looks this way, whether the observed demand is trustworthy, what Azure component is limiting it, whether a query or memory problem is hiding the real demand, what configuration is actually safe, and whether the workload remained healthy after the change."

That is substantially harder.

And much more valuable.

---

# 45. The core safety formula I would use

Conceptually, the system should not execute unless all of this is true:

```text
AUTOMATE =

TelemetryComplete
AND WorkloadCycleCovered
AND StorageDemandStable
AND NoSignificantQueryRegression
AND NoMemoryPressure
AND NoBlockingProblem
AND NoCPUConstraint
AND NoVMIOConstraint
AND StorageTopologyKnown
AND CandidateMeetsIOPS
AND CandidateMeetsThroughput
AND CandidateMeetsCapacity
AND Headroom >= 20%
AND SavingsMaterial
AND RollbackAvailable
AND ValidationAvailable
```

Then:

```text
execute change
```

and afterwards:

```text
IF PerformanceRegression
    rollback
ELSE
    commit
```

That would be the central philosophy of the feature.

---

## One architectural change I would make to the original problem statement

I would slightly redefine the goal from:

> "Safely automate changes to storage configuration."

to:

> **"Safely automate evidence-based infrastructure optimization while preserving the workload's required performance envelope."**

Storage is just the first resource.

Once Database Chex builds this engine correctly, the exact same framework can later optimize:

```text
VM CPU
Memory
Disk IOPS
Disk throughput
Disk type
Storage topology
SQL configuration
```

The sequence remains:

```text
Observe
→ Diagnose
→ Model demand
→ Generate candidate
→ Simulate
→ Calculate savings
→ Assess risk
→ Execute
→ Validate
→ Rollback/commit
```

That architecture turns Database Chex from a database monitoring product into a **closed-loop SQL Server optimization platform**.

[1]: https://learn.microsoft.com/en-us/azure/azure-sql/virtual-machines/windows/performance-guidelines-best-practices-collect-baseline?view=azuresql "Collect Baseline: Performance Best Practices & Guidelines - SQL Server on Azure VMs | Microsoft Learn"
[2]: https://learn.microsoft.com/en-us/azure/azure-sql/database/monitoring-with-dmvs?view=azuresql "Monitor Performance Using DMVs - Azure SQL Database & SQL database in Microsoft Fabric | Microsoft Learn"
[3]: https://learn.microsoft.com/en-us/azure/azure-monitor/reference/supported-metrics/microsoft-compute-virtualmachines-metrics?utm_source=chatgpt.com "Supported metrics - Microsoft.Compute/virtualMachines - Azure Monitor | Microsoft Learn"
[4]: https://learn.microsoft.com/en-us/azure/virtual-machines/disks-performance?utm_source=chatgpt.com "Virtual machine and disk performance - Azure Virtual Machines | Microsoft Learn"
[5]: https://learn.microsoft.com/nb-no/azure/azure-sql/virtual-machines/windows/performance-guidelines-best-practices-storage?view=azuresql "Storage: Performance Best Practices & Guidelines - SQL Server on Azure VMs | Microsoft Learn"
[6]: https://learn.microsoft.com/en-us/troubleshoot/azure/virtual-machines/windows/troubleshoot-disk-resize?utm_source=chatgpt.com "Troubleshoot Azure disk resize failures - Virtual Machines | Microsoft Learn"
[7]: https://learn.microsoft.com/en-us/troubleshoot/azure/virtual-machines/windows/downsize-data-disk-without-losing-data?utm_source=chatgpt.com "Decrease the size of an Azure data disk without losing data - Virtual Machines | Microsoft Learn"
[8]: https://learn.microsoft.com/en-us/azure/virtual-machines/disks-deploy-premium-v2?utm_source=chatgpt.com "Deploy a Premium SSD v2 managed disk - Azure Virtual Machines | Microsoft Learn"
[9]: https://learn.microsoft.com/en-us/azure/virtual-machines/disks-understand-billing?utm_source=chatgpt.com "Understand Azure Disk Storage billing - Azure Virtual Machines | Microsoft Learn"
[10]: https://learn.microsoft.com/en-us/azure/virtual-machines/disks-types?utm_source=chatgpt.com "Select a disk type for Azure IaaS VMs - managed disks - Azure Virtual Machines | Microsoft Learn"
[11]: https://learn.microsoft.com/en-us/azure/virtual-machines/disks-metrics?utm_source=chatgpt.com "Disk metrics - Azure Virtual Machines | Microsoft Learn"
