1 - Spark Internals

1. Big picture: why plans exist
2. Logical Plan – what Spark wants to do
3. Physical Plan – how Spark decides to do it
4. Execution Plan – what actually runs on the cluster
5. How these matter in real projects (joins, skew, slow jobs)
6. What you must remember as a data engineer

1️⃣ Why does Spark even have plans?
When you write this:
df.filter(df.age > 30).groupBy("dept").count()
👉 Spark does NOT execute immediately
Instead Spark thinks like this:
“First understand WHAT the user wants Then decide HOW to do it efficiently Then run it on the cluster”
That’s why Spark has three stages of thinking :
Stage	Purpose
Logical Plan	What needs to be done
Physical Plan	How to do it
Execution Plan	Actually doing it
This separation is the secret behind Spark’s power.

2️⃣ Logical Plan (WHAT to do)
Think of Logical Plan as:
“A pure mathematical description of the query”
No cluster. No executors. No partitions. No shuffle.
Just operations on data.

Example
df.filter(df.age > 30).select("name", "dept")
Logical Plan says:
* Read table
* Filter rows where age > 30
* Select name and dept columns
That’s it.

Types of Logical Plans
Spark internally has two logical plans:
1️⃣ Unresolved Logical Plan
* Column names not verified
* Table names not resolved
* Happens immediately after you write code
Example:
“Filter age > 30” (but Spark hasn’t checked if age exists)

2️⃣ Resolved Logical Plan
* Spark checks schema
* Confirms column types
* Resolves table metadata
If a column doesn’t exist → error happens here

Important Optimizations at Logical Level (Catalyst)
This is huge.
Spark applies rule-based optimizations, like:
🔹 Predicate Pushdown
df.filter(df.age > 30).select("name")
Spark changes order internally to:
Read → Filter → Select
Filter happens as early as possible.

🔹 Column Pruning
If you select only name, Spark won’t read other columns from disk.
This is why:
select() early is good practice

Key takeaway (Logical Plan)
Logical Plan answers: “What transformations are needed on the data?”

3️⃣ Physical Plan (HOW to do it)
Now Spark asks:
“I know WHAT to do. HOW should I do it efficiently on this cluster?”
This is where real engineering decisions happen.

Physical Plan decides:
* Broadcast join or shuffle join?
* Hash aggregation or sort aggregation?
* Number of stages
* Shuffle boundaries
* Partitioning strategy

Example: Join
orders.join(customers, "cust_id")
Physical Plan choices:
* BroadcastHashJoin
* SortMergeJoin
* ShuffledHashJoin
Spark chooses based on:
* Table size
* Statistics
* Configuration (spark.sql.autoBroadcastJoinThreshold)

You’ll see things like:
BroadcastHashJoin
Exchange hashpartitioning
SortMergeJoin
These are physical operators.

Why Physical Plan matters to YOU
When your job is slow:
* Logical plan is usually fine
* Physical plan is usually the problem
Examples:
* Unexpected shuffle
* Wrong join strategy
* Data skew not handled

Key takeaway (Physical Plan)
Physical Plan answers: “What execution strategy should Spark use?”

4️⃣ Execution Plan (ACTUAL RUN)
Execution Plan is not a separate plan in Spark docs, but practically:
Physical Plan compiled into RDD tasks
This is where:
* DAG is built
* Stages are created
* Tasks are sent to executors

What happens here:
* Jobs → Stages → Tasks
* Shuffles write to disk
* Executors process partitions
* Memory & CPU are used
This is where:
* OOM errors happen
* Long tasks occur
* Skew kills performance

Simple mapping
Plan	Spark Component
Logical Plan	Catalyst optimizer
Physical Plan	Spark SQL planner
Execution	DAG Scheduler + Task Scheduler
5️⃣ How YOU use this in real life (VERY IMPORTANT)
🔍 explain() is your best friend
df.explain(True)
You’ll see:
1. Parsed Logical Plan
2. Analyzed Logical Plan
3. Optimized Logical Plan
4. Physical Plan

Real-world debugging examples
🚨 Slow Join
* Logical Plan: correct
* Physical Plan: SortMergeJoin
* Fix: broadcast smaller table

🚨 Too many shuffles
* Physical Plan shows many Exchange
* Fix: repartition properly or reduce wide transformations

🚨 Skewed data
* One task runs forever
* Physical plan shows hash partitioning
* Fix: salting / AQE / skew hints

6️⃣ What you MUST remember (exam + interview + job)
🔑 Core truths
* Transformations are lazy
* Logical plan is independent of cluster
* Physical plan is where performance lives
* Execution is where failures happen

One-line mental model (very important)
Logical Plan = Intent Physical Plan = Strategy Execution = Reality

Next steps (tell me when to proceed)
If you say “OK”, next I will explain:
👉 How Logical → Physical plan conversion actually happens 👉 What is Catalyst Optimizer step-by-step 👉 How Adaptive Query Execution (AQE) changes physical plan at runtime
I’ll stop here and wait.
