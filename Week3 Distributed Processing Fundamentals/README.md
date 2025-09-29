# Week 3: Distributed Processing Fundamentals

> **Learning Path**: ML/AI Post-Graduate Program - Data Engineering Fundamentals

## Overview

This week builds directly on Week 2's distributed storage concepts. You learned **where** to store petabytes of data (HDFS). Now you'll learn **how** to process that data at scale using MapReduce and Apache Spark.

## Core Question Answered

**"Storage alone isn't enough - how do we process distributed data without writing custom distributed code every time?"**

**Answer:** Use distributed processing frameworks that handle parallelization, fault tolerance, and data movement automatically.

---

## Learning Objectives

By the end of this week, you will be able to:

- [ ] Explain the MapReduce programming paradigm and its two phases
- [ ] Design mapper and reducer functions for real-world problems
- [ ] Understand when and how to use combiners for optimization
- [ ] Configure the number of reducers and understand partitioning
- [ ] Explain why Spark replaced MapReduce in modern data engineering
- [ ] Write basic Spark programs using the RDD API
- [ ] Understand lazy evaluation, DAG, and lineage in Spark
- [ ] Compare open-source Spark with Databricks platform

---

## Course Materials

### Study Materials
- [`Week3_Notes.md`](./Week3_Notes.md) - Complete course notes
- [`Week3_Summary.md`](./Week3_Summary.md) - Condensed summary for quick review
- [`Week3_Glossary.md`](./Week3_Glossary.md) - Comprehensive terminology reference
- [word_count_ApacheSpark-RDD.ipynb](./word_count_ApacheSpark-RDD.ipynb) - Hands-on Spark RDD example

---

## Critical Concepts

### MapReduce Pipeline Flow
```
Input Data
    ↓
RecordReader (raw → key-value pairs)
    ↓
Mapper (parallel transformation)
    ↓
Combiner [optional] (local aggregation)
    ↓
Partitioner (decide which reducer)
    ↓
Shuffle (network transfer)
    ↓
Sort (group by key)
    ↓
Reducer (final aggregation)
    ↓
Output
```

### Spark Execution Flow
```
Load Data (from HDFS/S3/ADLS)
    ↓
Transformation 1 (lazy, builds plan)
    ↓
Transformation 2 (lazy, builds plan)
    ↓
Transformation 3 (lazy, builds plan)
    ↓
Action (triggers execution of entire DAG)
    ↓
Save Results
```

---

---

## Key Formulas

### Number of Mappers
```
Mappers = File Size ÷ Block Size
```
**Example:** 1 GB file, 128 MB blocks → 8 mappers

### Partition Assignment (Hash Partitioning)
```
partition = hash(key) % num_reducers
```
Ensures same key always goes to same reducer

### Parallel Mapper Execution
```
Concurrent Mappers = Nodes × Mapper Slots per Node
```
**Example:** 4 nodes × 4 slots = 16 mappers can run simultaneously

---

## MapReduce vs Spark Comparison

| Aspect | MapReduce | Spark |
|--------|-----------|-------|
| **Processing** | Disk-based | In-memory |
| **Speed** | Slower (many disk I/Os) | 10-100x faster |
| **Developer Experience** | Rigid key-value paradigm | Flexible, SQL-like APIs |
| **Chained Jobs** | Each job writes to disk | Keeps data in RAM |
| **Use Cases** | Batch processing | Batch, streaming, ML, SQL |
| **Era** | 2004-2014 (dominant) | 2014-present (industry standard) |

---

## Spark API Layers

### RDD (Low-level)
- Most flexible, hardest to use
- Direct control over distributed data
- Use only when higher APIs can't solve the problem

### DataFrame (High-level)
- Tabular data with named columns
- Optimized execution plans
- Most common in industry

### Spark SQL
- SQL queries on DataFrames
- Familiar syntax for SQL users

**Recommendation:** Start with DataFrame/SQL, drop to RDD only if needed

---
