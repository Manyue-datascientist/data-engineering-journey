# Week 4: Apache Spark Core APIs

## Overview

Deep dive into **Spark Core (RDD API)** - understanding in-memory processing, writing efficient transformations, and mastering optimization techniques for distributed computing at scale.

## Core Question Answered

**"How does Spark achieve 10-100x faster performance than MapReduce, and how do we write efficient Spark code?"**

**Answer:** In-memory processing, lazy evaluation with DAG optimization, and strategic use of cache, broadcast, and partition management.

---

## Learning Objectives

By the end of this week, you will be able to:

- [ ] Explain Spark's in-memory processing advantage over MapReduce
- [ ] Create and manipulate RDDs using transformations and actions
- [ ] Distinguish between narrow and wide transformations
- [ ] Optimize with reduceByKey, cache, broadcast joins
- [ ] Manage partitions using repartition and coalesce
- [ ] Understand DAG execution model and stage boundaries
- [ ] Apply functional programming patterns in distributed computing

---

## Course Materials

### Study Materials
- [`Week4_Notes.md`](./Week4_Notes.md) - Complete course notes
- [`Week4_Summary.md`](./Week4_Summary.md) - Quick review guide
- [`Week4_Glossary.md`](./Week4_Glossary.md) - Terminology reference

---

## Critical Concepts

### Spark vs MapReduce Execution

```
MapReduce:
Read HDFS → Process → Write Disk → Read Disk → Process → Write HDFS
(5 jobs = 10 disk I/Os)

Spark:
Read HDFS → RAM → RAM → RAM → Write HDFS
(5 transformations = 2 disk I/Os)
```

### RDD Pipeline Flow

```
textFile → map → filter → reduceByKey → collect
   └─────────┬─────────┘      └────┬────┘    └─┬─┘
          Stage 1            Stage 2        Action
        (no shuffle)        (shuffle)     (triggers)
```

### Transformation Types

| Type | Operations | Shuffle | Creates Stage |
|------|-----------|---------|---------------|
| **Narrow** | map, filter, flatMap | ❌ No | ❌ No |
| **Wide** | reduceByKey, join, groupByKey | ✅ Yes | ✅ Yes |

---

## Key Formulas

### Partitions & Execution
```
Partitions from HDFS = File Size ÷ Block Size
Jobs = Number of Actions
Stages = Wide Transformations + 1
Tasks = Number of Partitions
```

### Example
```
1 GB file, 128 MB blocks → 8 partitions
filter → map → reduceByKey → collect
            → 2 stages, 8 tasks per stage
```

---

## Core Optimization Patterns

### 1. reduceByKey vs groupByKey
```python
# ✅ Efficient - map-side combine
rdd.reduceByKey(lambda a, b: a + b)

# ❌ Inefficient - shuffles all values
rdd.groupByKey().mapValues(sum)
```

### 2. Cache Reused RDDs
```python
rdd.cache()  # Store in memory after first computation
```

### 3. Broadcast Small Datasets
```python
broadcast_var = sc.broadcast(small_dict)
# Avoid shuffling small lookup tables
```

### 4. Manage Partitions
```python
rdd.repartition(n)  # Increase/decrease with shuffle
rdd.coalesce(n)     # Decrease without full shuffle
```

---

## Comparison Tables

### Creating RDDs

| Method | Use Case | Data Source |
|--------|----------|-------------|
| `sc.textFile(path)` | Production | HDFS/S3/ADLS |
| `sc.parallelize(list)` | Prototyping | Driver memory |

### Transformations vs Actions

| | Transformation | Action |
|-|---------------|--------|
| **Execution** | Lazy | Eager |
| **Returns** | New RDD | Result |
| **Examples** | map, reduceByKey | collect, count |

### Partition Management

| | repartition() | coalesce() |
|-|--------------|------------|
| **Increase** | ✅ Yes | ❌ No |
| **Shuffle** | Full | Minimal |
| **Balance** | Even | May be uneven |

---

## Quick Reference

### Common Patterns
```python
# Word Count
rdd.flatMap(lambda x: x.split())
   .map(lambda x: (x, 1))
   .reduceByKey(lambda a, b: a + b)

# Top-N
rdd.takeOrdered(10, key=lambda x: -x[1])

# Distinct Count
rdd.distinct().count()
```

### Performance Checklist
✅ Use `reduceByKey` not `groupByKey`  
✅ Cache frequently reused RDDs  
✅ Broadcast small lookup tables  
✅ Coalesce after filtering  
❌ Avoid `collect()` on large data  

---

**This week provides the foundation for writing efficient, production-scale Spark applications.**