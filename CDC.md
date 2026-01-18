CDC batch-on-Delta → Delta is very common in Databricks industrial platforms, and most failures happen because these factors are not thought through upfront.
I’ll explain this as a design checklist + reasoning, not just features. No fluff. This is how industry-ready CDC systems are designed.

🧠 First: What “CDC batch on Delta” really means
You are designing a system where:
* Source Delta table changes over time
* You periodically (batch) detect changes
* You apply those changes to a target Delta table
* With correctness, recoverability, scalability
CDC ≠ just MERGE CDC = data correctness over time

1️⃣ Source System Factors (Most Important)
1. Do you have CDC information or not?
Ask this first:
Scenario	Meaning
INSERT / UPDATE / DELETE flags present	Best case
Operation timestamp present	Good
Only snapshots available	Hard
No primary key	🚨 design risk
Golden rule
CDC without a stable business key is unreliable.

2. Change identification strategy
You must decide how you detect change:
Strategy	When to use
op_type (I/U/D)	Source supports CDC
last_updated_ts	Soft CDC
hash comparison	Snapshot-based CDC
version column	Ideal
⚠️ Hash-based CDC is expensive at scale.

3. Primary key stability
Ask:
* Can PK change?
* Is it composite?
* Is it nullable?
If PK changes:
* Treat as DELETE + INSERT
* Never UPDATE primary key directly

2️⃣ Time & Ordering Considerations
4. Event time vs ingestion time
You must pick one explicitly:
Time type	Use case
Source event time	Business correctness
Ingestion time	Operational simplicity
Never mix silently.

5. Late arriving updates (critical)
Example:
* Order updated yesterday
* Arrives today
You must decide:
* Accept late updates?
* Up to how many days?
* Reprocess historical partitions?
This affects:
* Partition design
* Reprocessing logic

3️⃣ Target Table Design (CDC Friendly)
6. SCD Type decision
Before writing code, choose:
Type	Meaning
SCD 1	Overwrite
SCD 2	History
SCD 3	Limited history
Most industrial systems need SCD 2.

7. Delta table layout
Design upfront:
* Partition column (date / region)
* ZORDER keys
* Vacuum strategy
* Optimize cadence
Bad layout = slow MERGE forever.

8. Idempotency (non-negotiable)
Your batch must be re-runnable.
Guarantee:
* Same input → same target state
* No duplicates
* No double updates
Achieved by:
* Deterministic MERGE condition
* Version or timestamp logic

4️⃣ CDC Processing Logic (Core Engine)
9. Incremental boundary strategy
How do you decide what to read?
Approach	Risk
max(updated_ts)	Late updates risk
watermark lag	Safer
version-based	Best
Example:
read from source where updated_ts >= last_processed_ts - 2 days

🔥 10. MERGE strategy design
This is where most people fail.
You must define:
* MATCH condition
* WHEN MATCHED UPDATE conditions
* WHEN NOT MATCHED INSERT
* WHEN NOT MATCHED BY SOURCE DELETE (optional)
Never do blind MERGE.

11. Delete handling
Options:
* Soft delete flag
* Physical delete
* History-preserving delete
Industry prefers:
* Soft delete + SCD history

5️⃣ Performance & Scalability
12. Data volume growth
Think in:
* Rows/day
* Years of retention
* State growth
Plan:
* Archival strategy
* Table compaction
* Partition pruning

13. MERGE optimization
Key techniques:
* Filter source before merge
* Repartition on join key
* ZORDER target
* Avoid wide MERGE conditions

14. Small files problem
CDC batches often create small files.
Mitigate using:
* Auto Optimize
* Optimize writes
* Scheduled OPTIMIZE

6️⃣ Failure, Recovery & Auditability
15. Batch checkpointing (yes, even batch)
Maintain:
* Last processed timestamp
* Last batch ID
* Row counts
Store in:
* Delta control table

16. Restart & replay design
You must support:
* Job rerun
* Partial failure recovery
* Data backfill
This separates toy pipelines from industrial pipelines.

17. Auditing & reconciliation
Track:
* Source count
* Insert/Update/Delete count
* Target count
* Hash totals (optional)
Auditors will ask.

7️⃣ Schema Evolution & Data Quality
18. Schema drift handling
Decide:
* Auto-merge schema?
* Manual approval?
* Versioned schema?
CDC breaks silently with schema drift.

19. Null & default handling
CDC bugs often come from:
* Null PK
* Null timestamps
* Missing columns
Fail fast or route to quarantine.

8️⃣ Security & Governance (Often Missed)
20. Data lineage
Ensure:
* Table-level lineage
* Column-level lineage (if regulated)
Delta + Unity Catalog help here.

21. PII & masking
CDC pipelines often replicate sensitive fields. Design masking/encryption early.

9️⃣ Design Patterns (Proven in Industry)
22. Bronze → Silver → Gold CDC
* Bronze: Raw changes
* Silver: Clean CDC applied
* Gold: Business aggregates
Never do CDC directly into Gold.

23. Control tables pattern
Maintain:
* last_run_ts
* status
* retry_count
* metrics

24. Backfill strategy
Have explicit backfill mode:
* Full reload
* Partial reload
* Partition reload

🔑 Architect’s Golden Rules (Memorize These)
1. CDC is about correctness, not speed
2. Always assume late & duplicate data
3. MERGE logic must be deterministic
4. Every CDC batch must be rerunnable
5. Design recovery before writing code
