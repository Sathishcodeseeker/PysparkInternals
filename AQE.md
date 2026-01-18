# Adaptive Query Execution (AQE) — What It Controls, What It Doesn’t, and Why OOM Still Happens

Adaptive Query Execution (AQE) is often misunderstood as a “magic fix” for Spark performance and memory issues.
This document clarifies **what AQE actually controls**, **what it does not**, and **why Out Of Memory (OOM) errors still occur** even when AQE is enabled.

---

## 1️⃣ What AQE Controls

AQE dynamically adjusts the query plan **at runtime**, based on observed execution statistics.

### AQE can influence:

* **Join strategy (in some cases)**

  * Switch between Sort-Merge Join and Broadcast Join
* **Shuffle partition sizes**

  * Coalesce small shuffle partitions
* **Skew handling (limited)**

  * Split skewed shuffle partitions when possible

---

## 2️⃣ What AQE Does *NOT* Control

AQE does **not** change your core data or logic.

❌ Your data model
❌ Your join keys
❌ Your transformation order
❌ Your memory usage pattern
❌ Your UDF / Pandas UDF logic
❌ Your caching decisions

---

## 3️⃣ Then Why Do OOM Errors Still Happen?

OOM (Out Of Memory) occurs when **Spark’s execution reality exceeds available memory**.

Let’s break it down layer by layer.

---

## 4️⃣ OOM #1 — Executor Memory Overload *(Most Common)*

### Scenario

* A single task processes too much data
* AQE cannot split it further
* The task runs out of memory

### Why AQE Can’t Fix It

* AQE reacts **only after shuffle**
* Extreme skew (e.g., one partition = 50 GB)
* Or the problem happens **before shuffle**

👉 AQE is simply **too late**.

### Common Root Causes

* Poor partition key
* Highly skewed columns (e.g., `country = 'INDIA'`)
* Large `groupBy`
* Too much data collected into one task

---

## 5️⃣ OOM #2 — Broadcast Join Gone Wrong

### AQE Behavior

AQE may decide:

> “This table is small — let’s broadcast it”

But “small” can mean:

* **300 MB compressed**
* **2–3 GB expanded in memory**

💥 Result: Executor memory explosion.

---

## 6️⃣ OOM #3 — Cache Abuse *(Very Common)*

```python
df.cache()
```

Looks harmless, but:

* Data size > available memory
* Same DataFrame cached multiple times
* Executors constantly evict cached blocks

### Symptoms

* Heavy GC overhead
* OOM errors
* Severe performance degradation

👉 AQE does **nothing** here.

---

## 7️⃣ OOM #4 — UDF / Pandas UDF Memory Usage

AQE cannot see inside your code.

### Example

* Pandas UDF loads entire partition into memory
* Uses Python objects (very memory-heavy)

### Result

* Python worker OOM
* Executor killed

👉 AQE is **blind** to this.

---

## 8️⃣ OOM #5 — Driver OOM *(Silent Killer)*

```python
df.collect()
df.toPandas()
```

* AQE has **zero role**
* Driver attempts to pull all data
* Driver crashes instantly

---

## 9️⃣ Why Failures Increase at Large Scale (Core Truth)

Spark is **distributed**, not magical.

Failures increase with scale because:

* Data distribution is never perfectly uniform
* One bad partition → one failed task
* One failed task → stage retry
* Repeated retries → job failure

👉 AQE **reduces probability**, it does **not guarantee safety**.

---

## 🔟 What Data Engineers Actually Control *(Important)*

You control:

✔ Partition keys
✔ Number of shuffle partitions
✔ Join order
✔ Broadcast hints
✔ Cache placement
✔ UDF usage
✔ Data volume per task

---

## ✅ Final Takeaway

AQE is a **runtime optimizer**, not a memory safety net.

It helps **when the plan can adapt**,
but it cannot fix **bad data distribution, poor design choices, or unsafe code patterns**.

Understanding this boundary is key to building **stable, scalable Spark jobs**.
