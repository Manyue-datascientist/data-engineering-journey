# Week 3 Summary: Distributed Processing Fundamentals

## Core Problem Solved This Week
**Storage alone isn't enough** - you learned HDFS stores petabytes, but now you need to **process** that data across hundreds of machines without writing custom distributed code every time.

---

## MapReduce: The Foundation

### The Key Insight
In distributed systems, **moving code to data is cheaper than moving data to code** (data locality principle).

### Programming Model
Everything is key-value pairs: `(key, value)`

**Two phases:**
1. **Map** - Parallel transformation across nodes
2. **Reduce** - Aggregation of results

### Critical Pipeline
```
RecordReader → Mapper → [Combiner] → Partitioner → Shuffle → Sort → Reducer
```

---

## Design Principles That Matter

### 1. Push Work to Mappers
Mappers run in parallel across the entire cluster. If you push heavy computation to reducers, you create a bottleneck.

**Bad:** Mapper outputs raw data → Reducer does all computation  
**Good:** Mapper does heavy lifting → Reducer just aggregates

### 2. Combiners = Local Reducers
Run aggregation **before** shuffle to reduce network transfer.

**Safe for:** Max, Min, Sum, Count (associative + commutative)  
**Unsafe for:** Average (without redesign)

### 3. Control Reducer Count
- **Default = 1 reducer** → Bottleneck
- **0 reducers** → No aggregation needed (filtering jobs)
- **N reducers** → Parallelized aggregation via hash partitioning

**Formula:** `partition = hash(key) % num_reducers`

---

## Real-World Use Case: Google Search

### The Problem
Crawlers produce: `website.com → [keywords]`  
Search needs: `keyword → [websites]` (inverted index)

### Why MapReduce Was Revolutionary
- **Scalability:** Process billions of web pages across thousands of machines
- **Fault tolerance:** Failed tasks restart automatically
- **Simplicity:** Developers write map/reduce logic, framework handles distribution

**Mapper:** `(word, website)` pairs  
**Reducer:** Group all websites per word → searchable index

---

## Mapper & Reducer Counts

### Mappers
```
Number of Mappers = File Size ÷ Block Size
```
**Example:** 1 GB file, 128 MB blocks → 8 mappers

**Parallel execution** depends on cluster resources (nodes × slots)

### Reducers
User-controlled, affects:
- Output files (3 reducers = 3 output files)
- Parallelism in aggregation
- Network shuffle overhead

---

## MapReduce Limitations

### Performance Issues
- **Disk-heavy:** Every stage reads from and writes to disk
- **Chained jobs:** 5 MapReduce jobs = 10 disk I/Os
- Shuffle phase = expensive network transfer

### Developer Experience
- Rigid key-value paradigm
- Everything must fit map → shuffle → reduce
- Simple tasks require complex design

### Ecosystem Fragmentation
- Batch processing only (no streaming)
- Needed separate tools: Hive (SQL), Pig (scripting), Sqoop (data transfer)

---

## Apache Spark: The Evolution

### Core Innovation
**In-memory processing** - intermediate results stay in RAM, not written to disk

**MapReduce:**
```
Disk → CPU → Disk → CPU → Disk (every stage)
```

**Spark:**
```
Disk → RAM → CPU → RAM → CPU → Disk (final output only)
```

### Performance Impact
Chain of 5 transformations:
- **MapReduce:** 10 disk I/Os
- **Spark:** 2 disk I/Os (load + save final)

**Result:** 10-100x faster for iterative algorithms

---

## Spark Architecture

### Abstraction Layers

**1. RDD (Low-level)**
- Resilient Distributed Dataset
- Immutable, distributed collection of objects
- Foundation of everything in Spark
- Flexible but hard to optimize

**2. DataFrames & Spark SQL (High-level)**
- Like Pandas or SQL tables
- Optimized execution plans
- Most common in modern data engineering

**3. Specialized APIs**
- Structured Streaming (real-time)
- MLlib (machine learning)
- GraphX (graph processing)

### Key Concepts

**Lazy Evaluation**
- Transformations build a plan (DAG)
- Execution happens only when action is called
- Enables optimization and minimal data movement

**Lineage-based Fault Tolerance**
- Each RDD remembers how it was created
- Lost partitions can be recomputed
- No need for replication (unlike HDFS)

**Data Locality**
- Process data where it lives (same node)
- Minimizes network transfer

---

## Blocks vs Partitions

**Block (Physical)**
- Storage unit on disk (HDFS: 128 MB)
- Managed by storage layer

**Partition (Logical)**
- In-memory slice of data
- Unit of parallelism in Spark
- Loaded from blocks into RAM

**RDD = Collection of partitions across cluster memory**

---

## Spark Execution Flow

### Example: Word Count
```python
rdd1 = sc.textFile("file.txt")           # Load from HDFS
rdd2 = rdd1.flatMap(lambda x: x.split()) # Split into words
rdd3 = rdd2.map(lambda x: (x, 1))        # Create pairs
rdd4 = rdd3.reduceByKey(lambda a,b: a+b) # Aggregate
rdd4.saveAsTextFile("/output")           # Save to HDFS
```

**What happens:**
1. Spark builds DAG (execution plan)
2. Driver sends tasks to workers
3. Workers process partitions in parallel (in RAM)
4. Final result written to storage

**Critical:** Use `saveAsTextFile()` for large results, NOT `collect()` (which brings everything to driver)

---

## Spark vs Databricks

### Apache Spark
- Open-source compute engine
- Self-managed (install, configure, maintain)
- Runs on YARN, Kubernetes, standalone

### Databricks
- Commercial platform built on Spark
- **Optimized runtime** (faster than vanilla Spark)
- **Managed infrastructure** (auto-scaling, cluster mgmt)
- **Delta Lake** (ACID transactions on data lakes)
- **Collaborative notebooks** (like Jupyter++)
- **Unity Catalog** (data governance)

**Analogy:** Spark = engine, Databricks = Tesla with autopilot

---

## Hadoop Ecosystem Position

### The Stack
```
┌─────────────────────────┐
│   Spark / MapReduce     │ ← Processing Layer
├─────────────────────────┤
│         YARN            │ ← Resource Management
├─────────────────────────┤
│         HDFS            │ ← Storage Layer
└─────────────────────────┘
```

**Modern Alternative:**
```
┌─────────────────────────┐
│         Spark           │ ← Processing
├─────────────────────────┤
│    Kubernetes/YARN      │ ← Orchestration
├─────────────────────────┤
│    S3/ADLS/GCS          │ ← Cloud Storage
└─────────────────────────┘
```

Spark is **storage-agnostic** - works with HDFS, S3, ADLS, GCS, local files

---

## Key Takeaways for Interviews

### MapReduce Concepts
1. **Data locality** - code goes to data
2. **Combiner optimization** - reduce shuffle overhead
3. **Partitioning strategy** - controls reducer distribution
4. **Shuffle = bottleneck** - most expensive operation

### Spark Advantages
1. **In-memory = speed** - avoid disk I/O between stages
2. **Lazy evaluation** - optimize entire pipeline before execution
3. **Lineage = fault tolerance** - recompute lost partitions
4. **Unified API** - batch, streaming, ML, SQL in one framework

### Design Principles
1. Maximize mapper work, minimize reducer work
2. Use combiners when safe (associative + commutative)
3. Choose reducer count based on data size and cluster resources
4. Save large results to storage, collect() only small datasets

---

## Practical Applications

### Data Pipeline Pattern
1. **Ingest** - Load raw data from storage
2. **Transform** - Filter, join, aggregate, enrich
3. **Load** - Write to target (warehouse, lake, database)

### When to Use What

**MapReduce:**
- Legacy systems still in production
- Simple batch ETL with disk-based reliability
- Teaching fundamental distributed concepts

**Spark:**
- Modern data engineering (industry standard)
- Iterative algorithms (ML training)
- Real-time streaming + batch (unified)
- Complex transformations requiring SQL-like operations

---

## Common Pitfalls to Avoid

1. **Using collect() on large datasets** → Driver OOM error
2. **Too many small reducers** → Task scheduling overhead
3. **Not using combiners** → Unnecessary network traffic
4. **Forgetting lazy evaluation** → Thinking transformations execute immediately
5. **Reducer-heavy logic** → Defeats distributed parallelism

---

## The Big Picture

**Week 2:** You learned **where** to store data (HDFS, blocks, replication)  
**Week 3:** You learned **how** to process data at scale (MapReduce → Spark)

**Next:** You'll likely learn specific tools (Hive for SQL, streaming frameworks, orchestration) that build on these fundamentals.

Understanding MapReduce is crucial even if you only use Spark - it teaches you **how to think in distributed systems**. Spark abstracts complexity, but knowing what's underneath makes you a better data engineer.

---

## Quick Reference

### MapReduce Flow
```
Input → RecordReader → Map → Combine → Partition → Shuffle → Sort → Reduce → Output
```

### Spark Flow
```
Load → Transform (lazy) → Transform (lazy) → Action (execute DAG) → Save
```

### Key Formulas
```
Mappers = File Size ÷ Block Size
Partition = hash(key) % num_reducers
Parallelism = min(tasks, executor_slots)
```

This week gave you the **foundational mental model** for distributed data processing - everything from here builds on these concepts.