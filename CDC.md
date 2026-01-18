# Change Data Capture (CDC) on Delta Lake — Batch Design Guide

Delta Lake is widely used in Databricks-based industrial platforms.
Most CDC failures happen **not due to tools**, but due to **design decisions not made upfront**.

This document is a **design checklist with reasoning**, not a feature list.
Focus: **correctness, recoverability, and scalability**.

---

## 🧠 What “Batch CDC on Delta” Really Means

You are designing a system where:

* A **source Delta table changes over time**
* Changes are **periodically detected (batch)**
* Changes are **applied to a target Delta table**
* With guarantees on:

  * Correctness
  * Idempotency
  * Recovery
  * Scalability

> **CDC ≠ MERGE**
> **CDC = correctness of data over time**

---

## 1️⃣ Source System Factors (Most Critical)

### 1. Do You Have CDC Information?

Ask this **first**.

| Scenario                       | Meaning      |
| ------------------------------ | ------------ |
| INSERT / UPDATE / DELETE flags | Best case    |
| Operation timestamp present    | Good         |
| Only snapshots available       | Hard         |
| No primary key                 | 🚨 High risk |

**Golden Rule**

> CDC without a **stable business key** is unreliable.

---

### 2. Change Identification Strategy

Choose **exactly one** strategy.

| Strategy          | When to Use        |
| ----------------- | ------------------ |
| `op_type` (I/U/D) | Native CDC support |
| `last_updated_ts` | Soft CDC           |
| Hash comparison   | Snapshot-based CDC |
| Version column    | Ideal              |

⚠️ **Hash-based CDC is expensive at scale**

---

### 3. Primary Key Stability

Verify:

* Can PK change?
* Is it composite?
* Is it nullable?

**If PK changes:**

* Treat as **DELETE + INSERT**
* **Never UPDATE primary key**

---

## 2️⃣ Time & Ordering Considerations

### 4. Event Time vs Ingestion Time

Pick one **explicitly**.

| Time Type         | Use Case               |
| ----------------- | ---------------------- |
| Source event time | Business correctness   |
| Ingestion time    | Operational simplicity |

> Never mix silently.

---

### 5. Late Arriving Updates (Critical)

Example:

* Order updated yesterday
* Arrives today

Decide upfront:

* Are late updates allowed?
* How many days late?
* Do you reprocess history?

Impacts:

* Partitioning
* Watermarks
* Reprocessing logic

---

## 3️⃣ Target Table Design (CDC-Friendly)

### 6. SCD Type Selection

Choose **before coding**.

| Type  | Meaning         |
| ----- | --------------- |
| SCD 1 | Overwrite       |
| SCD 2 | Full history    |
| SCD 3 | Limited history |

> Most industrial systems require **SCD 2**

---

### 7. Delta Table Layout

Design upfront:

* Partition columns (date / region)
* ZORDER keys
* VACUUM strategy
* OPTIMIZE cadence

> Bad layout = slow MERGE forever.

---

### 8. Idempotency (Non-Negotiable)

Your batch **must be rerunnable**.

Guarantees:

* Same input → same target state
* No duplicates
* No double updates

Achieved via:

* Deterministic MERGE conditions
* Version / timestamp checks

---

## 4️⃣ CDC Processing Logic (Core Engine)

### 9. Incremental Boundary Strategy

How do you decide **what to read**?

| Approach          | Risk         |
| ----------------- | ------------ |
| `max(updated_ts)` | Late updates |
| Watermark lag     | Safer        |
| Version-based     | Best         |

**Example**

```sql
updated_ts >= last_processed_ts - INTERVAL 2 DAYS
```

---

### 🔥 10. MERGE Strategy Design

This is where most failures happen.

Define explicitly:

* MATCH condition
* WHEN MATCHED UPDATE
* WHEN NOT MATCHED INSERT
* WHEN NOT MATCHED BY SOURCE DELETE (optional)

> Never perform blind MERGE.

---

### 11. Delete Handling

Options:

* Soft delete flag
* Physical delete
* History-preserving delete

**Industry preference**:

* Soft delete + SCD history

---

## 5️⃣ Performance & Scalability

### 12. Data Volume Growth

Think in:

* Rows/day
* Years of retention
* State growth

Plan for:

* Archival
* Compaction
* Partition pruning

---

### 13. MERGE Optimization Techniques

* Filter source before MERGE
* Repartition on join key
* ZORDER target
* Avoid wide conditions

---

### 14. Small Files Problem

CDC batches often create small files.

Mitigation:

* Auto Optimize
* Optimize writes
* Scheduled OPTIMIZE

---

## 6️⃣ Failure, Recovery & Auditability

### 15. Batch Checkpointing (Yes, Even for Batch)

Maintain:

* Last processed timestamp
* Batch ID
* Row counts

Store in:

* Delta control table

---

### 16. Restart & Replay Design

Must support:

* Job rerun
* Partial failure recovery
* Backfills

> This separates toy pipelines from industrial systems.

---

### 17. Auditing & Reconciliation

Track:

* Source counts
* Insert / Update / Delete counts
* Target counts
* Optional hash totals

Auditors **will** ask.

---

## 7️⃣ Schema Evolution & Data Quality

### 18. Schema Drift Handling

Decide:

* Auto-merge schema?
* Manual approval?
* Versioned schema?

> CDC breaks silently with schema drift.

---

### 19. Null & Default Handling

Common CDC bugs:

* Null PKs
* Null timestamps
* Missing columns

Strategy:

* Fail fast **or**
* Route to quarantine

---

## 8️⃣ Security & Governance (Often Missed)

### 20. Data Lineage

Ensure:

* Table-level lineage
* Column-level lineage (regulated data)

Delta + Unity Catalog help here.

---

### 21. PII & Masking

CDC pipelines often replicate sensitive fields.

> Design masking and encryption **early**.

---

## 9️⃣ Proven Industry Design Patterns

### 22. Bronze → Silver → Gold CDC

* **Bronze**: Raw changes
* **Silver**: Clean CDC applied
* **Gold**: Business aggregates

> Never apply CDC directly into Gold.

---

### 23. Control Tables Pattern

Maintain:

* `last_run_ts`
* `status`
* `retry_count`
* `metrics`

---

### 24. Backfill Strategy

Have an explicit mode:

* Full reload
* Partial reload
* Partition reload

---

## 🔑 Architect’s Golden Rules

* CDC is about **correctness**, not speed
* Always assume **late & duplicate data**
* MERGE logic must be **deterministic**
* Every CDC batch must be **rerunnable**
* Design **recovery before code**

---

*End of document*
