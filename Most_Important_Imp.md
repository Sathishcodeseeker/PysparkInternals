# 🔥 Spark OOM & Failure Prevention

### Senior Engineer Pre-Flight Checklist

> Think of this as **mandatory pre-flight checks** before running any Spark job.

---

## 1️⃣ Data Shape Check (Most Skipped — Most Critical)

**Before writing a single line of code, answer these questions:**

* What is the **largest table size**?
* What is the **join key cardinality**?
* Is there **data skew**?
* Are there **hot keys**?
* What is the **row width** (number and size of columns)?

> ⚠️ If you skip this step, **AQE will not save you**.

---

## 2️⃣ Partitioning Checklist (Where Most OOMs Begin)

### Rule of Thumb

* **Each task should process 100–300 MB max**

### Recommended

```python
df = df.repartition("join_key")
```

### 🚫 Avoid at All Costs

```python
df.repartition(1)  # 💀 Guaranteed pain
```

### 🚩 Red Flags

* Very few partitions
* Extremely wide rows
* `groupBy` on low-cardinality columns

---

## 3️⃣ Join Safety Checklist (OOM Factory)

### Always Ask

* Which side is **smaller**?
* Can the smaller side **grow unexpectedly**?

### Safe Default

```python
spark.conf.set("spark.sql.autoBroadcastJoinThreshold", "-1")
```

### Explicit & Safe

```python
df_big.join(broadcast(df_small), "id")
```

> 👉 **You decide the join strategy — not Spark.**

---

## 4️⃣ Aggregation Safety (Silent Killer)

### 🚫 Dangerous

```python
df.groupBy("country").count()  # INDIA dominates
```

### ✅ Safer Options

* Use **salting**
* Increase shuffle partitions

```python
spark.conf.set("spark.sql.shuffle.partitions", 400)
```

---

## 5️⃣ Cache Discipline (Avoids Many Failures Alone)

### Cache **Only If**

* ✔ Data is reused multiple times
* ✔ Data fits comfortably in memory

### 🚫 NEVER

* Cache raw large datasets
* Cache before `filter` / `select`

### ✅ Correct Pattern

```python
df_filtered = df.filter(...).select(...)
df_filtered.cache()
```

---

## 6️⃣ UDF & Pandas UDF Rules (High-Risk Zone)

### Avoid When Possible

* Pandas UDFs on large partitions
* Python object-heavy logic

### If You Must

* Reduce partition size
* Use Arrow **intentionally**, not blindly

---

## 7️⃣ Driver Safety (Very Common Mistake)

### 🚫 NEVER on Large Data

```python
df.collect()
df.toPandas()
```

### ✅ Safe Alternative

```python
df.limit(1000).toPandas()
```

---

## 8️⃣ Spark UI: How to Find the Real Root Cause

### When a Job Fails

1. Open **Spark UI**
2. Go to **Stages**
3. Identify the stage with:

   * One task much slower than others
   * Extremely large input size

### Inspect

* Shuffle Read / Write
* Spill (Memory → Disk)

> 👉 This tells you **exactly** what went wrong.

---

## 9️⃣ AQE: Use It Correctly

### Let AQE Handle

* ✔ Minor skew
* ✔ Partition coalescing
* ✔ Join switching

### YOU Must Handle

* ❌ Extreme skew
* ❌ Bad partitioning
* ❌ Cache misuse
* ❌ UDF memory abuse

---

## 🔑 Golden Rules (Burn These In)

* Shuffles create **stages**
* Stages create **tasks**
* Tasks consume **memory**
* **One bad task can kill the entire job**

---

## 🧠 Mindset Shift

**Junior Engineer thinks:**

> “Spark failed.”

**Senior Engineer thinks:**

> “Which task processed too much data?”

---

## 📌 Final Takeaway

> **AQE optimizes execution plans.
> Data engineers design safety.**
