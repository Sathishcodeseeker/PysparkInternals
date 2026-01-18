🔥 Spark OOM & Failure Prevention – Senior Engineer Checklist
Think of this as “pre-flight checks” before you run a Spark job.

1️⃣ Before writing code: DATA SHAPE CHECK (most skipped)
Ask these 5 questions:
1. What is the largest table size?
2. What is the join key cardinality?
3. Is data skewed?
4. Are there hot keys?
5. What is the row width (number & size of columns)?
👉 If you skip this, AQE won’t save you.

2️⃣ Partitioning checklist (most OOMs start here)
Rule of thumb:
Each task should process 100–300 MB max
What YOU must do:

df = df.repartition("join_key")
Avoid:

df.repartition(1)   # 💀

Red flags:
* Very few partitions
* Very wide rows
* groupBy on low-cardinality column

3️⃣ Join safety checklist (OOM factory)
Always ask:
* Which side is smaller?
* Can it grow unexpectedly?
Safe defaults:

spark.conf.set("spark.sql.autoBroadcastJoinThreshold", "-1")
Then explicitly:

df_big.join(broadcast(df_small), "id")
👉 You decide, not Spark.

4️⃣ Aggregation safety (silent killer)
Dangerous:

df.groupBy("country").count()  # country = INDIA dominates
Safer:
* Use salting
* Or increase shuffle partitions

spark.conf.set("spark.sql.shuffle.partitions", 400)

5️⃣ Cache discipline (this alone avoids many failures)
Only cache if:
✔ Data reused multiple times ✔ Data fits in memory
NEVER:
* Cache raw large datasets
* Cache before filter / select

df_filtered = df.filter(...).select(...)
df_filtered.cache()

6️⃣ UDF / Pandas UDF rules (OOM hotspot)
Avoid if possible:
* Pandas UDF with large partitions
* Python object-heavy logic
If needed:
* Reduce partition size
* Use Arrow wisely

7️⃣ Driver safety (very common mistake)
NEVER on large data:

df.collect()
df.toPandas()
Instead:

df.limit(1000).toPandas()

8️⃣ Spark UI: how to find root cause (VERY IMPORTANT)
When job fails:
1. Go to Stages
2. Find stage with:
    * One task much slower
    * Huge input size
3. Check:
    * Shuffle Read / Write
    * Spill (memory → disk)
👉 That tells you exactly what went wrong.

9️⃣ AQE: how to use it correctly
Let AQE handle:
✔ Minor skew ✔ Partition coalescing ✔ Join switching
YOU handle:
❌ Extreme skew ❌ Bad partitioning ❌ Cache misuse ❌ UDF memory

🔑 Golden rules (print these mentally)
• Shuffles cause stages • Stages cause tasks • Tasks consume memory • One bad task can kill a job

🧠 Senior engineer mindset shift (important)
Junior thinks:
“Spark failed”
Senior thinks:
“Which task processed too much data?”

📌 Final one-line summary (lock this in)
AQE optimizes plans, but data engineers design safety.
