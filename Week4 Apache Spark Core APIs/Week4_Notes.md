# Week 4: Apache Spark Core APIs

## Topics Covered
- Python Basics
- Spark Usecase 1 - Orders Data
- Spark Core APIs - RDD
- More on Spark Parallelize
- More Spark Transformations
- Spark DAG Visualization | reduce Vs reduceByKey
- reduceByKey Vs groupByKey
- Spark Join
- Broadcast Joins
- Repartition Vs Coalesce
- Cache

---

## The Great Recap: Cluster, HDFS, MapReduce vs Spark

### 1. Cluster Hardware (Foundation)

A **cluster** = group of machines (nodes) arranged in racks inside a data center.

**Each machine/node has:**
- **CPU** → compute power
- **RAM** → fast, volatile memory
- **Disk (HDD/SSD)** → storage
- **Network** → connects nodes

**Providers:**
- **On-prem:** Dell, HP, IBM, Cisco, Lenovo
- **Cloud:** AWS (EC2), Azure (VMs), GCP (Compute Engine)

👉 Hardware is "dumb iron" until we install software layers on it.

### 2. HDFS = Distributed File System (Software Layer)

**HDFS** = distributed file system software.

- It splits files into blocks (default 128 MB) and stores them across nodes (DataNodes)
- Keeps 3 replicas (fault tolerance)
- Metadata (file → block → DataNode mapping) is handled by the NameNode

**HDFS mainly uses:**
- **Disk** (to persist file blocks)
- **Some RAM** (caching metadata)

⚠️ **Important:** HDFS does not own RAM. It only organizes how disk blocks are stored and accessed across the cluster.

👉 "Read from HDFS" = really "read blocks from DataNodes' disks, guided by NameNode metadata."

### 3. MapReduce = First Compute Engine

**Idea:** "Move compute to where data lives" (data locality).

**Flow:**
1. Mapper code runs locally where the block resides (data locality)
2. Intermediate results are written to local disk (not HDFS, just the node's disk)
3. Shuffled to reducer nodes → reducers aggregate → write final result back to HDFS

**Why slow?**
- Every stage → writes to disk, reads again
- For multi-stage jobs (chain of 5 MR), results hit disk multiple times → heavy I/O
- RAM in MapReduce → barely used, mostly just as small buffers

⚠️ **Bottleneck:** Too much disk I/O → slow.

### 4. Spark = Modern Compute Engine

Spark also runs on the same cluster but is designed to use RAM aggressively.

**How Spark Works:**
- Spark executors load data from HDFS (or S3/ADLS/local) into memory partitions
- Each partition (typically aligned with 128 MB block, but tunable) = a slice of the RDD/DataFrame
- Transformations (map, filter, reduceByKey, etc.) are applied in memory across partitions
- Intermediate results stay in executor memory (RAM), spilling to disk only if RAM is insufficient

👉 **Executor Memory** = Node's RAM allocated to Spark's executor JVM process.

**Only two mandatory disk I/Os:**
1. Read once from HDFS into partitions
2. Write final results back to HDFS

👉 Spark ≠ storage → it's a compute engine. It uses cluster RAM & CPUs.

### 5. Disk I/O Clarification

**MapReduce:**
- Mapper output → local disk
- Reducer reads from disk → aggregates → writes again
- Multi-stage = repeats

**Spark:**
- Mapper-like tasks (map/flatMap) stay in RAM
- Shuffle may spill to disk if data > memory, but Spark tries to keep it in RAM
- Final action → writes result to HDFS

### 6. Partitions & RDDs

- Each HDFS block (128 MB) → 1 partition in Spark
- A group of partitions = **RDD (Resilient Distributed Dataset)**

**RDDs are:**
- **Immutable:** new RDD created after each transformation
- **Lazy Execution:** Spark builds a DAG (Directed Acyclic Graph) of transformations. Execution only happens when an action (e.g., collect, count, saveAsTextFile) is called
- **Resilience:** If a partition is lost, Spark recomputes it using the lineage (the chain of transformations)

### 7. Memory vs Disk: Why RAM is Pricier

- **Disk** = cheap, slow, durable
- **RAM** = expensive, fast, volatile

**RAM is costlier because:**
- Built on different technology (DRAM vs magnetic/flash)
- Provides nanosecond access, but loses data on power-off
- Limited density compared to disk

That's why clusters traditionally used disk-heavy approaches.

### 8. Analogy for Intuition

- **Disks (HDFS blocks)** = fridges (cold, big, cheap)
- **RAM (Spark partitions)** = kitchen counter (fast, small, costly)
- **HDFS software** = labeling system for where food is kept across fridges
- **MapReduce** = after every chop, put ingredients back in fridge, re-take again later
- **Spark** = keep items on the counter until dish is ready, then put once in fridge

### 🔑 Final Takeaways

- HDFS = software layer managing disks across cluster machines
- Files are stored as blocks across DataNodes' disks (with replication)
- MR writes intermediate results to local disks → high I/O
- Spark keeps intermediate results in RAM → fewer I/Os
- Executors (RAM/CPU) always existed in clusters — Spark just leveraged them better
- Partitions in Spark ≈ blocks in HDFS, but now memory-resident RDDs
- Spark actions trigger DAG execution; transformations are just plan until then
- Spark's speed comes from in-memory compute + fewer mandatory disk I/Os

⚡ **So your mental flow:**
```
Cluster hardware → HDFS (storage) → MapReduce vs Spark (compute engines on top)
```

---

## 1 - Python Basics

Before going deep into Spark Core, let's recall Python functions & higher-order functions (because Spark transformations like map, reduceByKey are directly inspired by these).

### Normal Functions

A function is simply a reusable block of code.

**Example:**
```python
def my_sum(x, y):
    return x + y
    
print(my_sum(2, 2))   # 4
```

**Another example:** doubling every element in a list:
```python
my_list = [2, 3, 4, 5, 6]

def doubler(x):
    return 2 * x
    
output = list(map(doubler, my_list))
print(output)   # [4, 6, 8, 10, 12]
```

Here:
- `map()` applies a function to each element of the list
- The same way in Spark → `map()` transformation applies logic to each record of the dataset (1 row → 1 row)

### Lambda Functions

A lambda is just a shortcut for small, throwaway functions.

Instead of writing a `doubler()` function separately:
```python
output = list(map(lambda x: 2 * x, my_list))
print(output)   # [4, 6, 8, 10, 12]
```

💡 Think of it as: `lambda input: output`

### Reduce

Now let's say we want the sum of all elements in a list.

**Normal way:**
```python
def sum_list(x):
    total = 0
    for i in x:
        total += i
    return total

print(sum_list(my_list))  # 20
```

**Lambda + reduce:**
```python
from functools import reduce

result = reduce(lambda x, y: x + y, my_list)
print(result)   # 20
```

**How it works:**
- Reduce keeps combining elements pair by pair
- First call: 2 + 3 = 5
- Next: 5 + 4 = 9
- Next: 9 + 5 = 14
- Next: 14 + 6 = 20
- So the entire list is reduced to one final value

### Higher Order Functions

Both `map` and `reduce` are **higher-order functions:**
- They take another function as input
- They apply that function to data

This is exactly why Spark's APIs look familiar — Spark borrowed this functional style to process distributed datasets.

---

## 2 - Spark Usecase 1: Orders Data

### Problem Set

We'll compute:
1. Orders per status (count of each order_status)
2. Top-10 customers with the most orders (premium customers)
3. Distinct customer count (customers who placed ≥1 order)
4. Who has the most CLOSED orders

**Schema:**
```
order_id, order_datetime, customer_id, order_status
```

**Source (HDFS):**
```
/public/trendytech/retail_db/orders/*
```

### Core Ideas

#### 1. Actions: take() vs collect()

Both are actions (trigger execution of the DAG).

- Data lives in partitions in executor memory (spills to disk only if memory is full)
- **take(n):** asks executors for just enough partitions to fetch n records → small sample
- **collect():** brings all results to the driver → dangerous if the result is huge (can crash driver)

**Best practice:** use `collect()` only for tiny datasets; otherwise save to HDFS.

#### 2. Pair RDDs & "byKey" Ops

**Pair RDD** = (key, value) format, e.g. `("CLOSED", 1)` or `("12345", 1)`.

Required for ops like:
- `reduceByKey`, `groupByKey`
- `sortByKey`, `join`

Think of this as the MapReduce mental model:
- RecordReader → parse each line, produce key/values
- Map → map / flatMap in Spark
- Shuffle → happens internally when using reduceByKey, groupByKey, etc.
- Reduce → reduceByKey (per key) or reduce (global)

#### 3. Optimization Hints

- Always push filters early → shrink data before shuffle
- Use `reduceByKey` instead of `groupByKey` (does local combining)
- For "top-N" → `takeOrdered(N, key=…)` is better than a full sort
- Distinct counts use `.distinct().count()` → triggers shuffle but efficient enough

### Quick Reference

```python
orders = spark.sparkContext.textFile("/public/trendytech/retail_db/orders/*")

# 1) Orders per status
status_counts = (
    orders.map(lambda l: l.split(",")[3])
          .map(lambda s: (s, 1))
          .reduceByKey(lambda a, b: a + b)
          .sortBy(lambda kv: kv[1], ascending=False)
)

# 2) Top-10 customers
top10_customers = (
    orders.map(lambda l: l.split(",")[2])
          .map(lambda c: (c, 1))
          .reduceByKey(lambda a, b: a + b)
          .takeOrdered(10, key=lambda kv: -kv[1])
)

# 3) Distinct customers
distinct_customer_count = (
    orders.map(lambda l: l.split(",")[2]).distinct().count()
)

# 4) Most CLOSED orders
max_closed_customer = (
    orders.filter(lambda l: l.split(",")[3] == "CLOSED")
          .map(lambda l: (l.split(",")[2], 1))
          .reduceByKey(lambda a, b: a + b)
          .takeOrdered(1, key=lambda kv: -kv[1])
)
```

### Fast Recall Bullets

- Pair RDD → needed for all "byKey" ops
- `reduceByKey` is better than `groupByKey` (less shuffle)
- Push filters early (reduce network cost)
- `takeOrdered(N, key=…)` = efficient top-N
- Use `collect()` cautiously → only for small results

---

## 3 - Spark Core APIs: RDD

### What we're doing

The real file is huge (10 TB) → you don't prototype on it.

You build logic on a small local sample (a Python list), run fast, validate results.

Once logic is correct, you switch the source to real storage (HDFS/S3/ADLS) and run at scale.

### RDDs at a glance (context)

We're in Spark Core (RDD) API land (not DataFrames yet).

- **Transformations (lazy):** map, flatMap, filter, reduceByKey, distinct, sortBy, …
- **Actions (trigger):** collect, count, take, first, saveAsTextFile, …

### Two ways to get an RDD (important distinction)

#### From storage (real jobs):

```python
sc.textFile("/path/on/hdfs")
```
- Creates an RDD with partitions aligned to HDFS blocks
- Use this for actual big data

#### From local Python data (for prototyping only):

```python
sc.parallelize(local_list, numSlices=optional)
```
- Spreads a small in-memory list into N partitions
- Use this for unit-testing your logic on tiny samples

⚠️ `parallelize()` is not for 10 TB. It can only parallelize what's already in the driver's memory (your Python list). For 10 TB, you must read from distributed storage (`textFile`).

### Word Frequency logic (concept)

**Goal:** frequency of each word.

1. Normalize (lowercase, optionally strip punctuation)
2. Emit (word, 1) per token
3. Aggregate with `reduceByKey(_+_)`
4. (Optional) sort by frequency desc, or preview with take

**Why reduceByKey (not groupByKey):** does local combine pre-shuffle → less network.

```python
words = ["big", "Data", "Is", "Super", "big", "data", "is", "fun"]

word_counts = (
    spark.sparkContext.parallelize(words)
        .map(lambda x: x.lower())          # normalize
        .map(lambda x: (x, 1))             # pair RDD
        .reduceByKey(lambda a, b: a + b)   # aggregate counts
)

word_counts.collect()
```

### Fast recall bullets

- Use `parallelize` only for small, local prototypes
- For big data use `sc.textFile` (reads from HDFS/S3/ADLS)
- Transformations are lazy; actions run the DAG
- Prefer `reduceByKey` over `groupByKey` for counts/sums
- Tokenization matters: lowercase + strip punctuation → stable keys
- Don't `collect()` large outputs; save to storage instead

---

## 4 - More on Spark Parallelize

### 1) How many partitions does my RDD have?

```python
rdd.getNumPartitions()
```

**Why it matters:** Partitions = parallel tasks per stage.
- Too few → poor parallelism
- Too many → overhead / bigger shuffles

### 2) Where do partitions come from?

#### A) When you use `parallelize(local_list, numSlices=None)`

- If you don't set `numSlices`, Spark uses `spark.sparkContext.defaultParallelism` as the number of slices
- If you do set `numSlices`, Spark uses that (e.g., `numSlices=4`)
- `defaultParallelism` ≈ "how many tasks in parallel Spark tries to run by default"
- Practically: derived from your app's available cores (or from `spark.default.parallelism` if set). Admins can tune it.

🧠 **Key idea:** `parallelize` ships the local list from the driver to executors and splits it into `numSlices` partitions. It's for prototyping, not for big data.

#### B) When you use `sc.textFile("hdfs:///path")`

- Partitions are based on input splits (often ≈ HDFS block size, e.g., 128 MB), file format, and compression
- A 1 GB splittable text file with 128 MB blocks → ~8 partitions (but not guaranteed; formats and small-file layout can change this)
- You can hint a minimum with `minPartitions`:
  ```python
  rdd = sc.textFile("/path/on/hdfs", minPartitions=24)
  ```
- It's a hint, not a hard rule; the input format decides the exact splits
- `defaultMinPartitions` is a Spark setting used as the default floor for some file-reading APIs when you don't pass `minPartitions`. It does not force small files to split; think of it as a suggestion, not a guarantee.

### 3) "Is it distributed only if partitions > 1?"

Not exactly.

- With `parallelize`, the data starts on the driver and is then distributed to executors according to `numSlices`. Even 1 partition can still run on an executor, but it's just one task (no parallel speedup).
- With `textFile`, data is already distributed on HDFS across DataNodes; partitions map naturally to input splits.

**Rule of thumb:** Parallel speedup needs multiple partitions and available cores. One partition = one task = no intra-stage parallelism.

### 4) reduceByKey vs countByValue

#### reduceByKey (scalable)

- **Type:** Transformation → returns an RDD (stays distributed)
- **Why it scales:** Does map-side combine before shuffle → less network
- **Use when:** Counting/grouping in big data

```python
results = (orders_rdd
  .map(lambda l: (l.split(",")[3], 1))      # (status, 1)
  .reduceByKey(lambda a, b: a + b))          # RDD[(status, count)]
```

#### countByValue (driver-side)

- **Type:** Action → returns a Python dict on the driver
- **Risk:** If the result has many distinct values (high cardinality), you can OOM the driver
- **Use when:** Small domain (e.g., a handful of statuses) and small output only

```python
results = (orders_rdd
  .map(lambda l: l.split(",")[3])           # status
  .countByValue())                          # dict on driver
```

**Memory-safety rule:** If the result can be large, don't use `countByValue()`—stick to `reduceByKey()` and write the RDD out.

### 5) Code

#### Parallelize sample (prototype):

```python
words = ["big","Data","Is","Super","big","data","is","fun"]

word_counts = (
  sc.parallelize(words, numSlices=4)
    .map(lambda x: x.lower())
    .map(lambda w: (w, 1))
    .reduceByKey(lambda a, b: a + b)
)

word_counts.collect()
```

#### From HDFS (real data):

```python
orders_rdd = sc.textFile("hdfs:///public/trendytech/retail_db/orders/*")

status_counts = (
  orders_rdd
    .map(lambda l: l.split(",")[3])
    .map(lambda s: (s, 1))
    .reduceByKey(lambda a, b: a + b)
)
```

### Fast recall

- `getNumPartitions()` to inspect
- `parallelize(..., numSlices=K)` for prototypes; `textFile(..., minPartitions=K)` for real reads
- `defaultParallelism` drives parallelize when numSlices isn't given
- Use `reduceByKey` for scalable counts; `countByValue` only when the result is tiny

---

## 5 - More Spark Transformations

### What we learned so far:

- map, filter, flatMap
- reduceByKey, groupByKey

### Two Types of Transformations

All Spark transformations fall into two categories:

#### 1. Narrow Transformation

- No data movement across nodes
- Each partition is processed independently on the same node

**Example:**
```python
rdd2 = rdd1.map(lambda x: x * 2)
```
→ Each executor works on its own partition  
→ No shuffle, no network transfer

#### 2. Wide Transformation

- Shuffling involved — data moves across nodes over the network
- Spark must repartition or group data by keys to perform global operations

**Example:**
```python
rdd2 = rdd1.reduceByKey(lambda x, y: x + y)
```
→ Values with the same key must meet on one node  
→ Causes shuffle and disk I/O

### Spark Job Internals

When you trigger an action (like `collect()` or `count()`), Spark starts a job.

Each job → made of stages, and each stage → made of tasks.

| Level | Meaning |
|-------|---------|
| **Job** | Triggered by an Action |
| **Stage** | Logical phase between shuffles |
| **Task** | Work done per partition on each executor |

#### Key Rules to Remember

**No. of Jobs = No. of Actions**
- Example: two `.collect()` calls = two jobs

**No. of Stages = No. of Wide Transformations + 1**
- Each wide transformation (like reduceByKey) introduces a shuffle → new stage

**No. of Tasks = No. of Partitions**
- If RDD has 8 partitions, you'll see 8 tasks per stage

### Why Stages Matter

After every wide transformation, the intermediate results are written to the local disk (on executors), then read back for the next stage.

That's why minimizing wide transformations improves speed — less disk I/O and shuffle cost.

### Summary Table

| Concept | Description |
|---------|-------------|
| **Narrow Transformation** | Data stays within same partition/node |
| **Wide Transformation** | Causes shuffle across executors |
| **Job** | Created per action |
| **Stage** | Created per wide transformation |
| **Task** | Runs on each partition |
| **I/O Point** | At shuffle boundaries |

---

## 6 - Spark DAG Visualization | reduce Vs reduceByKey

### Summary of Key Events

| Step | What Happens | Where |
|------|--------------|-------|
| **File read** | Data loaded from HDFS/S3 into partitions | Executors' memory |
| **Narrow Transformations** | Performed in memory (no shuffle) | Same executor |
| **Wide Transformations** | Triggers shuffle, intermediate data written to local disk | Across executors |
| **New Stage** | Created after every shuffle boundary | DAG Scheduler |
| **Action** | Starts job execution | Driver program |
| **Tasks** | One per partition, executed on worker nodes | Executors |
| **Output** | Written back to distributed storage | HDFS / S3 / ADLS |

### Memory-Level Truths

- Each executor = one JVM process on a worker node
- Executor memory = that machine's RAM (reserved for Spark)
- Spark uses that RAM for all transformations until:
  - shuffle or
  - memory pressure (spill to local disk)
- The "local disk" = the worker node's physical disk, not HDFS
- Only final write (`saveAsTextFile`, etc.) goes back to HDFS/S3/ADLS

### reduceByKey vs reduce

**reduceByKey** → Transformation (wide) — works on pair RDDs (key, value)  
**reduce** → Action — works on normal RDDs, aggregates all elements into one value

#### reduceByKey

**Used for:** Per-key aggregation on pair RDDs  
**Type:** Wide transformation (shuffle happens)

**Example:**
```python
rdd = [("closed",1), ("pending",1), ("closed",1)]
rdd.reduceByKey(lambda x, y: x + y)
# Output: [("closed",2), ("pending",1)]
```

**How it works internally:**
1. **Map-side combine:** Each partition locally aggregates values for the same key (less shuffle data)
2. **Shuffle:** Sends combined results of each key to the reducer partition
3. **Reduce-side combine:** Merges partial results per key to produce final counts

✅ Keeps results distributed as an RDD — so you can continue transformations  
⚙️ Efficient because of local aggregation before shuffling

#### reduce

**Used for:** Aggregation across the entire RDD  
**Type:** Action (final result → single value)

**Example:**
```python
rdd = [3, 5, 7, 2]
rdd.reduce(lambda a, b: a + b)
# Output: 17
```

**How it works internally:**
1. Each partition computes a partial result
2. Then Spark merges all partials to a single output and sends it to the driver
3. Returns only one value, not an RDD
4. Not used for key-based aggregation — it just merges all records

### Why one is Transformation and one is Action

- `reduceByKey` keeps output as RDD for further processing → transformation
- `reduce` produces final result → triggers DAG → action

### When to Use What

| Goal | Function | Type | Works On | Returns | Shuffle |
|------|----------|------|----------|---------|---------|
| Aggregate per key | reduceByKey | Transformation | Pair RDD | RDD[(K,V)] | ✅ Yes |
| Aggregate all values | reduce | Action | Any RDD | Single value | ⚙️ No shuffle (only merge) |

### Summary Rules

- Use `reduceByKey` when you want per-key results and continue transformations
- Use `reduce` when you want a single, global result
- Avoid `groupByKey` when aggregating → it moves all values unnecessarily
- `reduceByKey` = map-side combine + shuffle + reduce-side combine

### Example comparison:

```python
# Per key
rdd.map(lambda x: (x, 1)).reduceByKey(lambda a, b: a + b)  # gives (word, count)

# Global
rdd.map(lambda x: x.amount).reduce(lambda a, b: a + b)  # gives total
```

### Quick Recall:

- **Transformation** → returns RDD (reduceByKey, map, filter, etc.)
- **Action** → returns non-RDD (final output) (reduce, count, collect)

---

## 7 - reduceByKey vs groupByKey

### The Scenario

We have a 3.5 GB file stored on HDFS.

Given default block size = 128 MB, the file is split into:
```
3.5 GB ÷ 128 MB ≈ 28 blocks → 28 partitions (RDD partitions)
```

Each partition resides on some worker node in the cluster.

### Step 1 — Map Phase (Narrow Transformation)

When we apply a narrow transformation like `map`, each partition is processed independently on its own worker node (data locality principle).

So → 28 tasks (1 per partition) → 28 results

e.g. each partition produces key-value pairs like:
```
("CLOSED", 1)
("OPEN", 1)
("CLOSED", 1)
...
```

No data movement yet → no shuffling → this is narrow.

### Step 2 — reduceByKey (Wide Transformation but Optimized)

When we call:
```python
rdd.reduceByKey(lambda x, y: x + y)
```

Spark performs a two-phase aggregation:

#### Local combining (map-side combine):

Each worker aggregates within its own partitions first.

Example (per node):
```
("CLOSED", 500)
("OPEN", 950)
("COMPLETE", 700)
```

Only unique keys × their partial counts remain.

#### Global reduce (shuffle phase):

These small aggregated outputs (one per key per node) are then shuffled to the reducers responsible for each key.

All "CLOSED" values → one reducer  
All "OPEN" values → another reducer, etc.

Because each node already aggregated locally, only summarized data travels over the network → minimal shuffle, balanced load, faster execution.

### Step 3 — groupByKey (Wide Transformation without Optimization)

Now, if you use:
```python
rdd.groupByKey()
```

there's no local aggregation before shuffle.

Each mapper sends all raw values for a key directly over the network.

**Example:**
- Cluster = 1000 nodes
- 1 TB dataset → ≈ 8000 partitions
- 9 distinct order statuses

**What happens:**
1. Every machine emits all records for its keys → all "CLOSED" records (maybe millions) are sent to one node, all "OPEN" records to another, etc.
2. Final stage: data for 9 keys spread across only 9 nodes

**Result:**
- 991 nodes are idle (can't help → parallelism drops → underutilization)
- Enormous network shuffle (1 TB data moved)
- High risk of Out-Of-Memory on those 9 reducers

That's why `groupByKey` is discouraged except when you truly need all values per key.

### Why Some Tasks Are Fast and Others Slow ("Stragglers")

Different keys → different data sizes.

Example: "CLOSED" might have 80% of data, while "COMPLETE" has 5%.

- The reducers handling heavier keys take longer
- Other executors finish early → sit idle → cluster under-utilized
- This uneven distribution is called **data skew**

`reduceByKey` reduces skew impact because it performs local aggregation first → shuffles less data, keeps tasks balanced.

### Key Takeaways

| Operation | Transformation Type | Local Aggregation | Shuffling Volume | Typical Use | Risk |
|-----------|-------------------|------------------|-----------------|-------------|------|
| **reduceByKey** | Wide (optimized) | ✅ Yes | 🔹Low (to final reducers only) | Counting / Summing / Aggregating | Efficient, recommended |
| **groupByKey** | Wide (unoptimized) | ❌ No | 🔸Huge (all values shuffled) | When you need all values per key | Causes skew, OOM, slow jobs |

### Recall Formula

```
# of Tasks = # of Partitions
# of Stages = # of Wide Transformations + 1
```

Each Wide Transformation → Data Written to Local Disk then Shuffled

---

## 8 - Spark Join

### Context

Spark join is a wide transformation, meaning it triggers data shuffling.

Even though joins are a fundamental operation, we study them only briefly here because:

In real-world Spark projects, joins are mostly handled using DataFrames or Spark SQL (we'll go deep into those later).

Still, understanding joins at the RDD level helps us visualize how Spark moves data during distributed operations.

### Dataset Setup

We have two datasets stored on HDFS:

| Dataset | Size | Blocks (128 MB) | Typical Partitions | Description |
|---------|------|----------------|-------------------|-------------|
| **Orders** | 1.1 GB | 9 blocks | 9 partitions | order_id, order_date, customer_id, order_status |
| **Customers** | 1 MB | 1 block | 2 partitions (min default) | customer_id, fname, lname, …, pincode |

**Cluster = 4 nodes**
```
N1       N2       N3        N4
p1,p2    p3,p4    p5,p6     p7,p8,p9     ← orders (9 partitions)
p1,p2                                     ← customers (2 partitions)
```

### The Goal

Join both datasets on `customer_id` to combine customer details with order details.

### Why Shuffle Happens

Assume:
- Customer ID 1012 appears in orders partition 9 (on node 4)
- The same 1012 is in customers partition 1 (on node 1)

To perform the join, both records with key 1012 must reach the same node.

Hence, Spark must shuffle data across the cluster so matching keys co-locate.

This shuffling stage is the expensive part — it involves network data transfer and disk writes of intermediate files.

### Mapper Logic (Before the Join)

For Spark to understand what to match on, both RDDs must emit data in (key, value) form with the same key type:

```
Orders RDD → (customer_id, order_status)
Customers RDD → (customer_id, customer_details)
```

### The Join Operation

```python
joined_rdd = customers_mapped.join(orders_mapped)
joined_rdd.saveAsTextFile("/user/itv021666/data/joined_output")
```

**What happens internally:**
1. Spark identifies all unique keys across both RDDs
2. It shuffles partitions so all records with the same key meet on one node
3. Matching pairs are joined, producing:
   ```
   (customer_id, (customer_record, order_record))
   ```

**Output Example:**
```
(1012, ("Mary Jones, TX, 92069", "CLOSED"))
(8827, ("Robert Hudson, PR, 00725", "COMPLETE"))
```

### Join = Wide Transformation

Because it requires data to move across nodes (network shuffle), join always creates a new stage in the Spark DAG.

In this example:
- Orders → 9 partitions
- Customers → 2 partitions
- Total = 11 tasks
- 1 wide transformation → 2 stages

Each stage writes intermediate data to the local disks of executors before shuffling.

### Key Takeaways

| Aspect | reduceByKey / map / filter | join |
|--------|---------------------------|------|
| **Transformation Type** | Narrow / Local | Wide (requires shuffle) |
| **Data Movement** | None | Yes (network shuffle) |
| **No. of Stages** | +0 | +1 |
| **Typical Cause of Shuffle** | Grouping / aggregation | Matching keys between datasets |
| **When to Use** | Small data, pre-aggregations | Combining related datasets |

### Recap

- Every join means shuffling — data for the same key must physically meet
- More shuffle → more time and more I/O
- We'll later optimize joins using Broadcast Joins and DataFrame APIs, which dramatically reduce shuffle for small lookup tables

---

## 9 - Broadcast Joins

### Problem with Regular Join

A standard join causes shuffling of data between nodes so that matching keys (like `customer_id`) end up on the same machine.

This is expensive — it involves disk writes, network transfers, and additional stages.

**Example cluster layout:**
```
N1       N2       N3        N4
p1,p2    p3,p4    p5,p6     p7,p8,p9     ← orders (9 partitions)
p1,p2                                     ← customers (2 partitions)
```

When joining, records of both datasets with the same key must meet in one node.

That's why Spark shuffles both datasets — especially the large one (orders) — leading to extra stages and I/O.

### The Idea Behind Broadcast Joins

We notice that in many real cases:
- One dataset (e.g., orders) is huge
- The other (e.g., customers) is very small (few MB)

Instead of shuffling both datasets, we can **broadcast** the small dataset to all nodes.

So each node already has a full copy of the smaller dataset.

Then, each node can perform the join locally with the partitions of the large dataset.

**Cluster after broadcasting:**
```
N1       N2       N3        N4
p1,p2    p3,p4    p5,p6     p7,p8,p9     ← orders (9 partitions)
All_cust All_cust All_cust All_cust       ← full copy of customers on all nodes
```

### Why It's Efficient

- **No shuffling** → no network transfer
- Every executor has a full copy of the lookup dataset
- Each node performs the join independently on its own partition
- No additional stage is created

### Code Example

```python
# Read both datasets
orders_rdd = sc.textFile("hdfs:///orders/*")
customers_rdd = sc.textFile("hdfs:///customers/*")

# Map to key-value pairs
orders_mapped = orders_rdd.map(lambda x: (x.split(",")[2], x.split(",")[3]))
customers_mapped = customers_rdd.map(lambda x: (x.split(",")[0], x))

# Collect customers to driver and broadcast
customers_dict = dict(customers_mapped.collect())
broadcast_customers = sc.broadcast(customers_dict)

# Join using broadcast variable
def join_with_broadcast(order_record):
    customer_id, order_status = order_record
    customer_info = broadcast_customers.value.get(customer_id, "Unknown")
    return (customer_id, (customer_info, order_status))

joined_rdd = orders_mapped.map(join_with_broadcast)
joined_rdd.saveAsTextFile("/user/itv021666/data/broadcast_joined_output")
```

### Key Points to Remember

- Use broadcast joins when one dataset is small enough to fit in executor memory
- Broadcasting avoids shuffle — the most expensive step in distributed joins
- Broadcast joins are much faster but memory-sensitive
- Usually, Spark automatically applies broadcast joins in DataFrames (based on size threshold)
- You can manually broadcast using `spark.sparkContext.broadcast()` when using RDDs

---

## 10 - Repartition Vs Coalesce

### When to Change Number of Partitions

You might increase or decrease the number of partitions in an RDD depending on how much data you're processing and how many executors are available.

### Increasing Partitions — For More Parallelism

**Suppose:**
- Cluster = 100 machines
- File = 5 GB → 40 blocks → 40 partitions
- Only 40 machines (or fewer) are being used; the rest are idle

If you increase partitions, Spark can use more executors → higher parallelism.

```python
orders_base.getNumPartitions()
repartitioned_orders = orders_base.repartition(100)
```

`repartition()` can increase or decrease partitions.

### Decreasing Partitions — For Efficiency

**Suppose:**
- File = 1 TB → 8,000 blocks → 8,000 partitions
- Cluster = 100 machines → each node ≈ 80 partitions
- After heavy filtering:
  - 128 MB → 1 MB per partition
  - → You now have 8,000 tiny partitions, each with ~1 MB data → unnecessary overhead

Better to combine them into fewer, larger partitions (say 80 × 100 MB each):

```python
optimized_orders = orders_base.repartition(80)
```

**Goal:** each partition should hold a meaningful amount of data.

Too many small partitions increase scheduling overhead.

### What About Coalesce?

`coalesce()` can only decrease partitions — not increase.

It's optimized for collapsing partitions without full shuffle.

**Example — 4-node cluster:**

**Before:**
```
N1: p1, p2
N2: p3, p4
N3: p5, p6
N4: p7, p8
```

If you do:
```python
new_rdd = base_rdd.coalesce(4)
```

Spark merges partitions within the same node as much as possible:

**After:**
```
N1: np1 (p1+p2)
N2: np2 (p3+p4)
N3: np3 (p5+p6)
N4: np4 (p7+p8)
```

No full reshuffle—just merges where data already resides.

However, resulting partitions can be unequal in size.

### Repartition vs Coalesce — Key Difference

| Aspect | repartition() | coalesce() |
|--------|--------------|------------|
| **Can Increase Partitions** | ✅ Yes | ❌ No |
| **Can Decrease Partitions** | ✅ Yes | ✅ Yes |
| **Shuffle** | Full Shuffle | Partial / No Shuffle |
| **Partition Size** | Nearly Uniform | Uneven Possible |
| **Performance** | Slower (due to reshuffle) | Faster (when reducing only) |

### Why Repartition Does Full Shuffle

Repartition ensures balanced partition sizes across the cluster.

Even distribution avoids "straggler" executors where one partition is huge and slows down job completion.

Uniform partition sizes help achieve stable, predictable performance during large aggregations or joins.

### When to Use

- **Increase partitions** → use `repartition()` for better parallelism
- **Decrease partitions** → prefer `coalesce()` (less shuffle)
- **Balanced workloads** → `repartition()` ensures even partition sizes

### Summary:

- Use `repartition()` when you need both control and balance (even sizes)
- Use `coalesce()` when just collapsing partitions to reduce overhead after filtering

---

## 11 - Cache

### What is Cache in Spark?

`cache()` is used to store an RDD in memory after it's computed once, so that subsequent actions on the same RDD can reuse the results without recomputation.

### Why Do We Cache?

By default, Spark uses lazy evaluation — transformations are only executed when an action (like `collect()` or `count()`) is called.

If you run multiple actions on the same RDD, Spark will recompute all the transformations leading to that RDD every time.

Caching avoids this — it keeps the RDD data in executor memory (RAM) after the first computation.

### Example

```python
orders_base = spark.sparkContext.textFile("/public/trendytech/retail_db/orders")

orders_filtered = orders_base.filter(lambda x: x.split(",")[3] != "PENDING_PAYMENT")

orders_mapped = orders_filtered.map(lambda x: (x.split(",")[2], 1))

orders_reduced = orders_mapped.reduceByKey(lambda x, y: x + y)

orders_reduced.cache()  # Marks this RDD to be cached
```

Now when you do:
```python
orders_reduced.collect()     # First time → triggers computation & caching
orders_reduced.filter(...).collect()  # Second time → reuses cached data
```

On the first action, Spark executes the full DAG and stores the RDD's partitions in memory.

On subsequent actions, it reads directly from cache, not recomputing transformations.

### Relation to Stages

When you execute an action like `collect()`, Spark builds a DAG and divides it into stages.

- After each wide transformation, results are written to local disk
- Cached RDDs, however, are kept in executor memory (RAM) for reuse across actions

**Example:**
```
collect() first time  → runs all stages → caches results
collect() second time → skips recomputation → reads cached RDD from memory
```

### What If There's Not Enough Memory?

If Spark's executor memory is insufficient to hold the cached data, it will automatically evict old cached partitions or spill them to local disk (not HDFS).

### cache() vs persist()

| Method | Default Storage | Options | Use Case |
|--------|----------------|---------|----------|
| **cache()** | MEMORY_ONLY | No options | Fast reuse for small to medium RDDs |
| **persist()** | Customizable | MEMORY_AND_DISK, DISK_ONLY, etc. | For large datasets or limited memory |

**Example:**
```python
orders_reduced.persist(StorageLevel.MEMORY_AND_DISK)
```

### Key Points to Remember

- `cache()` is lazy — data is cached only when an action triggers execution
- Cached data lives in executor memory, not HDFS
- Caching avoids recomputation, improving performance for repeated actions
- Use `persist()` when you need more control over where and how data is stored
- Unpersist when no longer needed to free memory:
  ```python
  orders_reduced.unpersist()
  ```

### Summary:

Cache is Spark's in-memory optimization technique that speeds up repeated computations by keeping frequently used RDDs in executor memory instead of recomputing them from scratch.