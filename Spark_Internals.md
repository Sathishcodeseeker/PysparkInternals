# Spark Internals – A Practical Guide

## Big Picture: Why Plans Exist

When working with Apache Spark, execution does **not** happen immediately. Spark separates *intent* from *execution* to optimize performance.

Spark works through **three conceptual plans**:

| Plan Type          | Purpose                           |
| ------------------ | --------------------------------- |
| **Logical Plan**   | What needs to be done             |
| **Physical Plan**  | How it should be done             |
| **Execution Plan** | What actually runs on the cluster |

This separation is the foundation of Spark’s performance and scalability.

---

## 1️⃣ Why Does Spark Even Have Plans?

Consider the following code:

```python
df.filter(df.age > 30).groupBy("dept").count()
```

Spark does **not** execute this immediately.

Instead, Spark thinks in three steps:

1. Understand **WHAT** the user wants
2. Decide **HOW** to do it efficiently
3. Execute it on the cluster

This is why Spark separates planning into stages.

---

## 2️⃣ Logical Plan — *WHAT to Do*

Think of the **Logical Plan** as:

> A pure, declarative description of the query

Characteristics:

* No cluster awareness
* No executors
* No partitions
* No shuffles
* Just transformations on data

### Example

```python
df.filter(df.age > 30).select("name", "dept")
```

Logical Plan interpretation:

1. Read table
2. Filter rows where `age > 30`
3. Select `name` and `dept`

---

### Types of Logical Plans

Spark uses **two logical plans internally**:

#### 1️⃣ Unresolved Logical Plan

* Column names not verified
* Table names not resolved
* Created immediately after code is written

Example:

> “Filter `age > 30`” (Spark hasn’t checked if `age` exists)

#### 2️⃣ Resolved Logical Plan

* Schema validated
* Column data types confirmed
* Table metadata resolved

❗ Errors such as missing columns occur **here**, not at runtime.

---

### Logical-Level Optimizations (Catalyst Optimizer)

Spark applies **rule-based optimizations** at the logical level.

#### 🔹 Predicate Pushdown

```python
df.filter(df.age > 30).select("name")
```

Internally optimized to:

```
Read → Filter → Select
```

Filters are applied as early as possible.

---

#### 🔹 Column Pruning

If only `name` is selected, Spark does **not** read unnecessary columns from disk.

**Best practice:** call `select()` early.

---

### Key Takeaway (Logical Plan)

> Logical Plan answers: **“What transformations are required on the data?”**

---

## 3️⃣ Physical Plan — *HOW to Do It*

Once Spark knows *what* to do, it decides:

> How can this be executed efficiently on the cluster?

This is where real engineering decisions happen.

### Physical Plan Determines:

* Join strategy (broadcast vs shuffle)
* Aggregation strategy (hash vs sort)
* Number of stages
* Shuffle boundaries
* Partitioning strategy

---

### Example: Join Strategies

```python
orders.join(customers, "cust_id")
```

Possible physical plans:

* `BroadcastHashJoin`
* `SortMergeJoin`
* `ShuffledHashJoin`

Spark chooses based on:

* Table size
* Available statistics
* Configuration (e.g. `spark.sql.autoBroadcastJoinThreshold`)

---

### Why Physical Plan Matters

When a Spark job is slow:

* Logical Plan is usually correct
* **Physical Plan is usually the problem**

Common issues:

* Unexpected shuffles
* Suboptimal join strategy
* Data skew

---

### Key Takeaway (Physical Plan)

> Physical Plan answers: **“What execution strategy should Spark use?”**

---

## 4️⃣ Execution Plan — *ACTUAL RUN*

While not explicitly named in Spark docs, the **Execution Plan** is effectively:

> Physical Plan compiled into RDD tasks

This is where:

* DAG is built
* Jobs → Stages → Tasks are created
* Tasks are sent to executors
* Memory, CPU, and disk are used

---

### What Happens at Execution Time

* Shuffles write data to disk
* Executors process partitions
* Long tasks may appear
* OOM errors can occur
* Data skew becomes visible

---

### Simple Mapping

| Plan          | Spark Component                |
| ------------- | ------------------------------ |
| Logical Plan  | Catalyst Optimizer             |
| Physical Plan | Spark SQL Planner              |
| Execution     | DAG Scheduler + Task Scheduler |

---

## 5️⃣ How YOU Use This in Real Life (Very Important)

### 🔍 `explain()` Is Your Best Friend

```python
df.explain(True)
```

This shows:

* Parsed Logical Plan
* Analyzed Logical Plan
* Optimized Logical Plan
* Physical Plan

---

### Real-World Debugging Examples

#### 🚨 Slow Join

* Logical Plan: correct
* Physical Plan: `SortMergeJoin`
* Fix: broadcast the smaller table

#### 🚨 Too Many Shuffles

* Physical Plan shows many `Exchange`
* Fix: reduce wide transformations or repartition correctly

#### 🚨 Data Skew

* One task runs forever
* Physical Plan shows hash partitioning
* Fix: salting, AQE, or skew hints

---

## 6️⃣ What You MUST Remember (Exam + Interview + Job)

### 🔑 Core Truths

* Transformations are **lazy**
* Logical Plan is **cluster-independent**
* Physical Plan is where **performance lives**
* Execution is where **failures happen**

---

### One-Line Mental Model

> **Logical Plan = Intent**
> **Physical Plan = Strategy**
> **Execution = Reality**

---

