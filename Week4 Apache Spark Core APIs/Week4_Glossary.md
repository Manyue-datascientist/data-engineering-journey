# Week 4 Glossary: Apache Spark Core APIs

> Compact reference - all concepts covered in Week 4

---

## Core Concepts

**RDD (Resilient Distributed Dataset)** - Immutable, partitioned collection spread across cluster. Fault-tolerant via lineage.

**Partition** - Logical chunk of RDD data. One partition = one task. Check with `rdd.getNumPartitions()`

**Lineage** - Chain of transformations that created an RDD. Enables recomputation if partition is lost.

**Executor** - JVM process on worker node that runs tasks and stores partitions in memory.

**Driver** - Process that runs main program, builds DAG, coordinates executors.

**Executor Memory** - RAM allocated to executor for processing and caching data.

---

## Creating RDDs

**sc.textFile(path)** - Read from distributed storage (HDFS/S3/ADLS). For production big data.

**sc.parallelize(list, numSlices)** - Create RDD from Python list. For prototyping only (small data).

**defaultParallelism** - Default partition count for `parallelize()`. Based on available cores.

---

## Transformations (Lazy)

**Transformation** - Operation that creates new RDD. Doesn't execute until action called.

**Lazy Evaluation** - Builds execution plan (DAG) without executing. Enables optimization.

### Narrow Transformations (No Shuffle)
- **map(func)** - Transform each element: `rdd.map(lambda x: x * 2)`
- **filter(func)** - Keep matching elements: `rdd.filter(lambda x: x > 10)`
- **flatMap(func)** - Map and flatten: `rdd.flatMap(lambda x: x.split())`

### Wide Transformations (Shuffle Required)
- **reduceByKey(func)** - Aggregate by key with map-side combine: `rdd.reduceByKey(lambda a,b: a+b)`
- **groupByKey()** - Group values by key. Avoid for aggregations (no map-side combine).
- **join(other)** - Join two pair RDDs: `rdd1.join(rdd2)`
- **distinct()** - Remove duplicates. Requires shuffle.
- **sortBy(func)** - Sort elements by key function.

---

## Actions (Trigger Execution)

**Action** - Triggers DAG execution, returns result or writes to storage.

- **collect()** - Bring all data to driver. ⚠️ Only for small results.
- **count()** - Count elements.
- **take(n)** - Return first n elements.
- **first()** - Return first element.
- **saveAsTextFile(path)** - Write to storage. Preferred for large results.
- **takeOrdered(n, key)** - Get top-N efficiently.
- **countByValue()** - Count occurrences, returns dict to driver. ⚠️ Small cardinality only.

---

## Pair RDD

**Pair RDD** - RDD of (key, value) tuples. Required for "byKey" operations.

**Example:** `rdd.map(lambda x: (key, value))`

---

## Execution Model

**DAG (Directed Acyclic Graph)** - Execution plan showing transformation dependencies.

**Job** - Computation triggered by one action.

**Stage** - Set of transformations between shuffles. `Stages = Wide Transformations + 1`

**Task** - Smallest unit of work. Processes one partition. `Tasks = Partitions`

**Shuffle** - Redistributing data across network. Most expensive operation. Happens at wide transformations.

---

## Optimization

### Cache & Persist
**cache()** - Store RDD in memory for reuse. `rdd.cache()`

**persist(level)** - Store with custom level (MEMORY_AND_DISK, etc.)

**unpersist()** - Remove from cache to free memory.

**When to use:** RDD used multiple times, iterative algorithms.

### Broadcast
**Broadcast Variable** - Read-only variable sent to all executors once.

```python
broadcast_var = sc.broadcast(small_dict)
# Use: broadcast_var.value
```

**Broadcast Join** - Join where small dataset is broadcast instead of shuffled. No shuffle for large dataset.

### Partition Management
**repartition(n)** - Increase/decrease partitions with full shuffle. Balanced sizes.

**coalesce(n)** - Decrease partitions only, minimal shuffle. May be uneven.

**When to use:** Increase for parallelism, decrease after filtering.

---

## Performance Concepts

**Narrow vs Wide Transformations:**
- **Narrow:** No shuffle, fast (map, filter)
- **Wide:** Shuffle required, slow (reduceByKey, join)

**Data Skew** - Uneven data distribution causes some tasks to run much longer.

**Map-Side Combine** - Local aggregation before shuffle (reduceByKey does this).

**Data Locality** - Schedule tasks where data lives. Reduces network transfer.

**Local Disk** - Worker node's disk (not HDFS) for shuffle files and spill.

**Spill to Disk** - Writing to disk when data doesn't fit in memory.

---

## Python Functional Programming

**Lambda** - Anonymous function: `lambda x: x * 2`

**map()** - Apply function to each element: `list(map(func, list))`

**reduce()** - Aggregate to single value: `reduce(lambda a,b: a+b, list)`

**Higher-Order Function** - Function that takes/returns functions. Spark API based on this.

---

## Key Formulas

```
Partitions from HDFS = File Size ÷ Block Size
Jobs = Actions
Stages = Wide Transformations + 1
Tasks = Partitions
```

---

## Common Patterns

**Word Count:**
```python
rdd.flatMap(lambda x: x.split())
   .map(lambda x: (x, 1))
   .reduceByKey(lambda a,b: a+b)
```

**Top-N:**
```python
rdd.takeOrdered(10, key=lambda x: -x[1])
```

**Distinct Count:**
```python
rdd.distinct().count()
```

---

## Critical Comparisons

### reduceByKey vs groupByKey
| | reduceByKey | groupByKey |
|-|-------------|------------|
| Map-side combine | ✅ Yes | ❌ No |
| Shuffle volume | Low | High |
| Use for | Aggregations | Need all values |

### repartition vs coalesce
| | repartition | coalesce |
|-|-------------|----------|
| Increase partitions | ✅ Yes | ❌ No |
| Shuffle | Full | Minimal |
| Partition sizes | Even | May be uneven |

### Transformation vs Action
| | Transformation | Action |
|-|---------------|--------|
| Execution | Lazy | Triggers DAG |
| Returns | New RDD | Result/write |

---

## Best Practices

✅ **DO:**
- Use `reduceByKey` for aggregations (not groupByKey)
- Cache RDDs used multiple times
- Broadcast small lookup tables for joins
- Use `saveAsTextFile()` for large results
- Push filters early in pipeline
- Coalesce after heavy filtering

❌ **DON'T:**
- Use `collect()` on large data (OOM risk)
- Use `groupByKey()` for counting/summing
- Forget lazy evaluation
- Ignore partition count (too few or too many)

---

## Debugging

**Spark UI** - Web interface showing jobs, stages, tasks, DAG, shuffle metrics.

**getNumPartitions()** - Check partition count: `rdd.getNumPartitions()`

**Stage Boundaries** - Created at wide transformations (shuffle points).

---

**Quick Recall:** Spark keeps data in memory (RAM) between transformations, only writes to disk at shuffle boundaries and final output. This makes it 10-100x faster than MapReduce.