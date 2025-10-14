# Week 4 Summary: Apache Spark Core APIs

## 🎯 Core Learning: Deep Dive into Spark RDD Programming

This week focused on **Spark Core (RDD API)** - understanding how Spark processes data in memory, how to write efficient transformations, and optimization techniques for distributed computing.

---

## 🧠 The Foundation: Cluster Architecture Recap

### Hardware → Software Layers

```
Cluster Hardware (CPU, RAM, Disk, Network)
         ↓
    HDFS (Storage Layer - manages disk blocks)
         ↓
MapReduce vs Spark (Compute Engines - use RAM & CPU)
```

### Key Insight: Disk vs Memory

**MapReduce:**
- Reads from disk → Processes → Writes to disk (every stage)
- Chain of 5 jobs = 10 disk I/Os
- **Bottleneck:** Too much disk I/O

**Spark:**
- Reads from disk → Keeps in RAM → Processes → Writes final result
- Chain of 5 transformations = 2 disk I/Os (read + final write)
- **Advantage:** In-memory processing = 10-100x faster

### Analogy
- **Disk (HDFS)** = Fridge (slow, cheap, durable)
- **RAM (Spark)** = Kitchen counter (fast, expensive, volatile)
- **MapReduce** = Put ingredients back in fridge after every step
- **Spark** = Keep everything on counter until dish is ready

---

## 📊 RDD Fundamentals

### What is an RDD?

**RDD = Resilient Distributed Dataset**

- **Resilient:** Fault-tolerant via lineage (can recompute lost partitions)
- **Distributed:** Spread across cluster nodes as partitions
- **Dataset:** Collection of data elements

### Key Properties

1. **Immutable** - Every transformation creates a new RDD
2. **Lazy Evaluation** - Builds DAG; executes only when action is called
3. **Partitioned** - Split into chunks for parallel processing

### Creating RDDs

**For Prototyping (small data):**
```python
rdd = sc.parallelize([1, 2, 3, 4, 5], numSlices=4)
```

**For Production (big data):**
```python
rdd = sc.textFile("hdfs:///path/to/data")
```

---

## 🔄 Transformations vs Actions

### Transformations (Lazy)
Build execution plan, don't execute immediately.

**Examples:**
- `map()` - Transform each element
- `filter()` - Keep matching elements
- `flatMap()` - Map + flatten
- `reduceByKey()` - Aggregate by key (with shuffle)
- `groupByKey()` - Group values by key (expensive)

### Actions (Trigger Execution)
Execute the DAG and return results.

**Examples:**
- `collect()` - Bring all data to driver
- `count()` - Count elements
- `take(n)` - Get first n elements
- `saveAsTextFile()` - Write to storage

---

## ⚡ Narrow vs Wide Transformations

### Narrow Transformations
- No shuffle, no network transfer
- Each partition processed independently
- **Examples:** `map()`, `filter()`

### Wide Transformations
- Requires shuffle across network
- Creates new stage in DAG
- Writes intermediate data to disk
- **Examples:** `reduceByKey()`, `groupByKey()`, `join()`

### Impact on Performance

```
# of Stages = # of Wide Transformations + 1
# of Tasks = # of Partitions
```

**More wide transformations = More shuffles = Slower**

---

## 🔑 Critical Optimizations

### 1. reduceByKey vs groupByKey

| Aspect | reduceByKey | groupByKey |
|--------|-------------|------------|
| **Local Aggregation** | ✅ Yes (map-side combine) | ❌ No |
| **Shuffle Volume** | Low (only aggregated results) | High (all raw values) |
| **Performance** | Fast, efficient | Slow, can cause OOM |
| **Use Case** | Counting, summing | When you need all values |

**Rule:** Always prefer `reduceByKey` for aggregations.

### 2. Broadcast Joins

**Problem:** Regular joins shuffle both datasets

**Solution:** Broadcast small dataset to all nodes

```python
# Small dataset
broadcast_var = sc.broadcast(small_dict)

# Join locally on each node (no shuffle)
result = large_rdd.map(lambda x: (x, broadcast_var.value.get(x)))
```

**When to Use:**
- One dataset is small (few MB to few hundred MB)
- Avoids expensive shuffle
- Dramatically faster for lookup joins

### 3. Partitioning Strategy

**Too Few Partitions:**
- Poor parallelism
- Underutilized cluster

**Too Many Partitions:**
- Task scheduling overhead
- Many tiny files

**Optimal:**
```
Partitions ≈ 2-3x number of cores in cluster
```

### 4. Cache for Reuse

```python
rdd.cache()  # Store in memory after first computation

rdd.collect()  # First action - computes and caches
rdd.count()    # Second action - reuses cache (no recomputation)
```

**When to Cache:**
- RDD used multiple times
- Iterative algorithms
- Interactive exploration

**Storage Options:**
- `cache()` = MEMORY_ONLY (default)
- `persist(MEMORY_AND_DISK)` = Spill to disk if needed

---

## 🔧 Partition Management

### repartition() vs coalesce()

| Operation | Increase? | Decrease? | Shuffle? | Balance? |
|-----------|-----------|-----------|----------|----------|
| **repartition()** | ✅ Yes | ✅ Yes | Full shuffle | Even sizes |
| **coalesce()** | ❌ No | ✅ Yes | Minimal/none | May be uneven |

**Use Cases:**

**Increase partitions (more parallelism):**
```python
rdd = rdd.repartition(200)  # Spread across more executors
```

**Decrease partitions (reduce overhead):**
```python
rdd = rdd.coalesce(10)  # After heavy filtering
```

---

## 📈 Job Execution Model

### DAG (Directed Acyclic Graph)

Spark builds an execution plan showing dependencies:

```
textFile → map → filter → reduceByKey → collect
   └─────────┬─────────┘       └────┬────┘
          Stage 1            Stage 2
```

### Stages and Tasks

**Stage:** Set of transformations between shuffles

**Task:** Work on one partition

**Example:**
- File: 1 GB, 128 MB blocks → 8 partitions
- Transformations: map, filter, reduceByKey
- Result: 2 stages, 8 tasks per stage

---

## 💡 Real-World Use Case: Orders Data

### Problem Set

1. Count orders per status
2. Find top-10 customers by order count
3. Count distinct customers
4. Find customer with most CLOSED orders

### Solution Pattern

```python
orders = sc.textFile("/data/orders/*")

# 1. Orders per status
status_counts = (orders
    .map(lambda x: (x.split(",")[3], 1))
    .reduceByKey(lambda a, b: a + b)
    .sortBy(lambda x: x[1], ascending=False)
)

# 2. Top-10 customers
top_customers = (orders
    .map(lambda x: (x.split(",")[2], 1))
    .reduceByKey(lambda a, b: a + b)
    .takeOrdered(10, key=lambda x: -x[1])
)

# 3. Distinct customers
distinct_count = orders.map(lambda x: x.split(",")[2]).distinct().count()
```

---

## ⚙️ Python Functional Programming Refresher

### Higher-Order Functions

**map()** - Transform each element
```python
list(map(lambda x: x * 2, [1, 2, 3]))  # [2, 4, 6]
```

**reduce()** - Aggregate to single value
```python
from functools import reduce
reduce(lambda a, b: a + b, [1, 2, 3, 4])  # 10
```

**Why This Matters:**
Spark RDD API is directly inspired by functional programming patterns.

---

## 🚨 Common Pitfalls

### 1. Using collect() on Large Data
**Problem:** Brings all data to driver → OOM crash

**Solution:** Use `saveAsTextFile()` or `take(n)` for sampling

### 2. Using groupByKey() for Aggregation
**Problem:** Shuffles all raw values → network bottleneck

**Solution:** Use `reduceByKey()` instead

### 3. Not Caching Reused RDDs
**Problem:** Recomputes from scratch every time

**Solution:** Cache RDDs used multiple times

### 4. Too Many Small Partitions
**Problem:** After filtering, 8000 partitions × 1 MB each

**Solution:** Coalesce to fewer, larger partitions

### 5. Forgetting Lazy Evaluation
**Problem:** Thinking transformations execute immediately

**Solution:** Remember only actions trigger computation

---

## 📊 Performance Decision Tree

```
Need to aggregate by key?
├─ Yes → Use reduceByKey (not groupByKey)
└─ No → Continue

Need to join datasets?
├─ One small dataset? → Use broadcast join
└─ Both large? → Regular join (but optimize partitioning)

Using RDD multiple times?
├─ Yes → Cache it
└─ No → Don't cache

After heavy filtering?
├─ Many tiny partitions? → Coalesce
└─ Want more parallelism? → Repartition
```

---

## 🔍 Debugging & Monitoring

### Check Partition Count
```python
rdd.getNumPartitions()
```

### View Execution Plan
Spark UI → DAG Visualization shows:
- Stages (separated by shuffles)
- Tasks per stage
- Shuffle read/write volumes

### Memory Management
```python
rdd.cache()           # Mark for caching
rdd.unpersist()       # Free memory
```

---

## 🎓 Key Formulas & Patterns

### Partitioning
```
Mappers = File Size ÷ Block Size
Tasks = Number of Partitions
Stages = Wide Transformations + 1
```

### Pair RDD Pattern
```python
# Always for "byKey" operations
rdd.map(lambda x: (key, value))
   .reduceByKey(lambda a, b: combine(a, b))
```

### Word Count Pattern (Classic)
```python
text_rdd
  .flatMap(lambda line: line.split())
  .map(lambda word: (word, 1))
  .reduceByKey(lambda a, b: a + b)
```

---

## 🔗 Connection to Previous Weeks

**Week 2 (HDFS):**
- Files stored as 128 MB blocks on DataNodes
- Replication factor = 3 for fault tolerance

**Week 3 (MapReduce → Spark):**
- MapReduce = disk-heavy (slow)
- Spark = memory-heavy (fast)

**Week 4 (Spark RDD Deep Dive):**
- How to write efficient Spark code
- Optimization techniques (cache, broadcast, reduceByKey)
- Understanding DAG and stages

---

