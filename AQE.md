What AQE controls
* Join strategy (in some cases)
* Shuffle partition sizes
* Skew handling (limited)
What AQE does NOT control
❌ Your data model ❌ Your join keys ❌ Your transformation order ❌ Your memory usage pattern ❌ Your UDF logic ❌ Your caching decisions


2️⃣ Then WHY do OOM errors still happen?
OOM (Out Of Memory) happens when Spark’s execution reality exceeds memory constraints.
Let’s go layer by layer.

3️⃣ OOM #1 – Executor memory overload (MOST COMMON)
Scenario
* One task processes too much data
* AQE cannot split it further
* That task runs out of memory
Why AQE can’t fix it
* AQE only reacts after shuffle
* If skew is extreme (1 partition = 50 GB)
* Or happens before shuffle
AQE is too late.


Real causes
* Bad partition key
* Highly skewed column (like country=INDIA)
* Large groupBy
* Collecting too much data in one task


4️⃣ OOM #2 – Broadcast gone wrong
AQE behavior
* AQE may decide: “This table is small → broadcast it” 
But “small” might be:
* 300 MB compressed
* 2–3 GB expanded in memory
💥 Executor memory explosion.

5️⃣ OOM #3 – Cache abuse (very common)

df.cache()
Looks harmless.
But:
* Data bigger than memory
* Cached multiple times
* Executors evict data aggressively
Result:
* GC overhead
* OOM
* Slowness
AQE does nothing here.


6️⃣ OOM #4 – UDF / Pandas UDF memory usage
AQE cannot see inside your code.
Example:
* Pandas UDF loads entire partition into memory
* Uses Python objects (very heavy)
Result:
* Python worker OOM
* Executor killed
AQE is blind here.




7️⃣ OOM #5 – Driver OOM (silent killer)

df.collect()
df.toPandas()
AQE has zero role.
Driver tries to:
* Pull all data
* Dies instantly



8️⃣ Why failures happen at large scale (core truth)
Spark is distributed, not magical
Failures increase with scale because:
* Data distribution is never uniform
* One bad partition = one failed task
* One failed task can kill a stage
* One stage retry can kill the job
AQE reduces probability, not guarantees safety.



9️⃣ What control data engineers ACTUALLY have (important)
You control:
✔ Partition keys ✔ Number of shuffle partitions ✔ Join order ✔ Broadcast hints ✔ Cache placement ✔ UDF usage ✔ Data volume per task
