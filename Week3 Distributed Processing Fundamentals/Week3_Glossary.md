# Week 3 Glossary: Distributed Processing Terms

> This glossary contains only terms covered in Week 3 materials. Reading through this should help you recall the entire week's learning journey.

---

## MapReduce Fundamentals

### MapReduce
A programming paradigm for processing large datasets across distributed clusters. Everything is represented as (key, value) pairs. Two main phases: Map (parallel transformation) and Reduce (aggregation).

### Data Locality Principle
Code goes to data, not data to code. Moving computation to where data lives is cheaper than moving petabytes of data across the network.

### Key-Value Pairs
The fundamental data structure in MapReduce. All inputs and outputs are in (key, value) format.

**Example:** `(1, "Hello world")`, `("Hello", 1)`

---

## MapReduce Pipeline Components

### RecordReader
Converts raw input data into key-value pairs that mappers can process.

**Example:** Text line `"Hello world"` becomes `(0, "Hello world")` where 0 is the line offset.

### Mapper
User-defined function that processes each input key-value pair and outputs intermediate key-value pairs. Runs in parallel on each block of data.

**Example - Word Count:**
- Input: `(0, "Hello Hadoop")`
- Output: `("Hello", 1)`, `("Hadoop", 1)`

### Combiner
A local reducer that runs on mapper output before shuffle. Reduces the amount of data transferred across the network.

**Safe for:** Max, Min, Sum, Count (associative + commutative operations)  
**Unsafe for:** Average (without redesign - emit sum and count separately)

**Example - Temperature Data:** Instead of sending all temperature readings, send only the local max per date from each mapper.

### Partitioner
Decides which reducer gets which key-value pairs from mappers.

**Default:** Hash Partitioner
```
partition = hash(key) % num_reducers
```

Ensures the same key always goes to the same reducer for consistency.

### Shuffle
The process of moving mapper/combiner output across the network to appropriate reducers. Often the most expensive operation in MapReduce.

### Sort
Framework automatically groups all values for the same key before passing to reducer. Happens on reducer side after shuffle.

**Output format:** `key → {v1, v2, v3, ...}`

**Example:** `("Hello", {1, 1, 1})`

### Reducer
User-defined function that aggregates all values for a key to produce final output.

**Example - Word Count:**
- Input: `("Hello", {1, 1, 1})`
- Output: `("Hello", 3)`

---

## MapReduce Configuration

### Number of Mappers
Automatically determined by the framework based on input data size.

**Formula:**
```
Number of Mappers = File Size ÷ Block Size
```

**Example:** 1 GB file, 128 MB blocks → 8 mappers

**Parallelism:** Depends on cluster resources (nodes × mapper slots per node)

### Number of Reducers
User-controlled parameter that affects output and parallelism.

**Default:** 1 reducer

**Options:**
- **0 reducers:** No aggregation, mapper output is final (filtering jobs)
- **1 reducer:** Single output file, potential bottleneck
- **N reducers:** N output files, parallel aggregation

### Custom Partitioner
User-defined logic to override hash-based partitioning.

**Example:** Words < 4 characters → Reducer 1, Words ≥ 4 characters → Reducer 2

### Mapper Slots
Maximum number of mapper tasks that can run concurrently on a node.

**Example:** 4 nodes × 4 slots per node = 16 mappers can run in parallel

---

## Real-World Use Cases

### LinkedIn Profile Views Example
**Goal:** Count how many times each profile was viewed

**Input:** `[viewer_id, profile_viewed, timestamp]`

**Mapper Output:** `(profile_viewed, 1)` for each view

**Reducer Output:** `(profile_name, total_count)`

### Sensor Data Example
**Goal:** Find maximum temperature per day

**Without Combiner:** Every temperature reading sent across network

**With Combiner:** Only local max per date sent, reducer finds global max

### Google Inverted Index
**Problem:** Crawlers produce `website → keywords`, but search needs `keyword → websites`

**Mapper:** Outputs `(word, website)` pairs

**Reducer:** Groups all websites for each word

**Result:** Searchable index powering Google Search

---

## Apache Spark

### Apache Spark
Open-source distributed compute engine designed for fast, in-memory data processing. 10-100x faster than MapReduce for iterative algorithms.

**Created:** UC Berkeley, donated to Apache Software Foundation

**Current version:** Spark 3.x

### In-Memory Processing
Spark's core advantage - keeps intermediate results in RAM instead of writing to disk after each step.

**MapReduce:** Disk → CPU → Disk → CPU → Disk (every stage)

**Spark:** Disk → RAM → CPU → RAM → CPU → Disk (final output only)

**Chain of 5 jobs:**
- MapReduce: 10 disk I/Os
- Spark: 2 disk I/Os (load + save)

### Why Spark Over MapReduce

**Performance Issues in MapReduce:**
- Too many disk I/Os between stages
- Shuffle phase is expensive

**Developer Experience Issues:**
- Rigid key-value paradigm
- Everything must fit map → shuffle → reduce pattern
- Complex logic for simple problems

**Spark Solutions:**
- In-memory processing
- Flexible APIs (Python, Java, Scala, R)
- SQL-like operations
- Unified engine (batch, streaming, ML, graph)

---

## Spark Core Concepts

### RDD (Resilient Distributed Dataset)
The fundamental data structure in Spark. An immutable, distributed collection of objects partitioned across the cluster.

**Properties:**
- **Resilient:** Recovers from failures using lineage
- **Distributed:** Spread across cluster nodes
- **Immutable:** Never changes; transformations create new RDDs

### DataFrame
Higher-level abstraction built on RDDs. Represents data as a table with named columns (like Pandas or SQL tables).

**Advantage:** Easier to use, optimized execution plans

### Spark SQL
Module for working with structured data using SQL queries on DataFrames.

### SparkSession
Entry point to Spark functionality (Spark 2.x+). Unifies SparkContext, SQLContext, and HiveContext.

**Example:**
```python
spark = SparkSession.builder.master('yarn').getOrCreate()
```

---

## Spark Execution Model

### Transformation
Operation that creates a new RDD/DataFrame from an existing one. Lazy - doesn't execute immediately.

**Examples:** `map()`, `filter()`, `flatMap()`, `groupBy()`, `reduceByKey()`

### Action
Operation that triggers actual computation and returns results or writes to storage.

**Examples:** `collect()`, `count()`, `saveAsTextFile()`, `first()`

### Lazy Evaluation
Spark builds an execution plan (DAG) but doesn't execute until an action is called.

**Benefits:**
- Optimize entire pipeline before execution
- Push filters early to minimize data processed
- Avoid unnecessary computation

**Example:**
```python
rdd1 = sc.textFile("bigfile")  # No execution
rdd2 = rdd1.map(...)            # No execution
rdd3 = rdd2.filter(...)         # No execution
rdd3.first()                    # NOW everything executes
```

### DAG (Directed Acyclic Graph)
Spark's internal representation of the execution plan showing dependencies between operations.

**Purpose:** Optimization and fault recovery

### Lineage
The chain of transformations that created an RDD. If data is lost, Spark recomputes it by replaying the lineage.

**Example:** If RDD3 (from RDD1.map().filter()) is lost, Spark reruns map and filter on RDD1

**Comparison:** HDFS uses replication for fault tolerance, Spark uses lineage

### Immutability
RDDs cannot be modified. Every transformation produces a new RDD.

**Benefit:** Enables safe parallel processing and lineage-based recovery

---

## Storage Concepts

### Block (Physical)
Unit of storage on disk (HDFS, S3, ADLS).

**HDFS Default:** 128 MB

**Purpose:** How data is physically distributed across DataNodes

### Partition (Logical)
Logical chunk of data in memory that Spark processes. When Spark reads from disk, blocks become partitions in RAM.

**Relationship:** Typically one partition per block (but configurable)

### RDD as Collection of Partitions
RDD = distributed collection of partitions spread across cluster memory.

**Flow:** File on disk → split into blocks → loaded as partitions in RAM → forms RDD

---

## Cluster Execution

### Driver
The process that runs the main program and creates SparkSession. Builds the DAG and coordinates task execution.

**Responsibilities:** Job scheduling, task distribution, result collection

### Worker / Executor
Process on worker nodes that runs tasks and stores data partitions.

**Responsibilities:** Execute tasks, cache data in memory, return results to driver

### Data Locality
Processing happens where data lives. Tasks are sent to nodes containing the required partitions.

---

## Spark Operations

### flatMap()
Transformation that maps one input to zero or more outputs, then flattens the result.

**Example - Word Count:**
```python
rdd1.flatMap(lambda line: line.split(" "))
# "Hello World" → ["Hello", "World"]
```

### map()
Transformation that applies a function to each element.

**Example:**
```python
rdd2.map(lambda word: (word, 1))
# "Hello" → ("Hello", 1)
```

### reduceByKey()
Aggregates values for each key. Automatically uses combiners for efficiency.

**Example:**
```python
rdd3.reduceByKey(lambda a, b: a + b)
# ("Hello", {1,1,1}) → ("Hello", 3)
```

### collect()
Action that brings all data to the driver node.

**Warning:** Use only for small results. Large datasets (e.g., 2TB) will cause out-of-memory errors.

### saveAsTextFile()
Action that writes RDD to storage as text files.

**Preferred:** Use this instead of `collect()` for large results.

**Example:**
```python
rdd4.saveAsTextFile("/user/username/output")
```

---

## Databricks

### Databricks
Commercial platform built by Spark creators. Provides managed Spark with additional features.

**Components:**
- Optimized Spark runtime (faster than open-source)
- Cloud-native (AWS, Azure, GCP)
- Auto-scaling cluster management
- Collaborative notebooks
- Delta Lake (ACID transactions on data lakes)

**Analogy:** Spark = free engine, Databricks = Tesla with autopilot

---

## Spark API Layers

### RDD API (Low-level)
Original Spark API. Very flexible but hard to optimize.

**Use:** Custom logic that higher-level APIs can't handle

**Characteristics:** Like coding in assembly - powerful but verbose

### DataFrame API (High-level)
Tabular data with named columns. Easier to use, optimized by Spark.

**Use:** Most common for modern data engineering

### Spark SQL
SQL queries on DataFrames.

**Example:** `spark.sql("SELECT * FROM table WHERE age > 25")`

### Specialized APIs
- **Structured Streaming:** Real-time data processing
- **MLlib:** Machine learning library
- **GraphX:** Graph processing

**Recommendation:** Start with DataFrame/SQL, drop to RDD only if needed

---

## Hadoop Ecosystem Position

### Three Pillars of Hadoop
1. **HDFS** → Storage
2. **MapReduce** → Processing
3. **YARN** → Resource Management

### Spark's Flexibility
- **Storage agnostic:** Works with HDFS, S3, ADLS, GCS, local files
- **Resource manager agnostic:** Works with YARN, Kubernetes, Mesos, standalone

**Modern Stack:**
```
Spark (Processing)
YARN (Resource Management)
HDFS or S3/ADLS (Storage)
```

---

## Word Count Complete Example

### Problem
Count word frequency in a text file

### Spark Solution (RDD API)
```python
# Step 1: Load file
rdd1 = spark.sparkContext.textFile("/path/to/file.txt")

# Step 2: Split into words
rdd2 = rdd1.flatMap(lambda line: line.split(" "))

# Step 3: Create (word, 1) pairs
rdd3 = rdd2.map(lambda word: (word, 1))

# Step 4: Aggregate by key
rdd4 = rdd3.reduceByKey(lambda x, y: x + y)

# Step 5: Save results (preferred for large data)
rdd4.saveAsTextFile("/path/to/output")

# Or collect (only for small results)
rdd4.collect()
```

### Flow
```
Line → Words → (Word, 1) → Group & Sum → Result
```

---

## Key Design Principles

### Maximize Mapper Work
Push heavy computation to mappers (parallel) rather than reducers (bottleneck).

**Bad:** Mappers output raw data → Reducer does all work  
**Good:** Mappers do heavy lifting → Reducer just aggregates

### Use Combiners When Safe
Reduce shuffle overhead by aggregating locally before network transfer.

**Safe:** Associative + commutative operations (sum, count, max, min)

### Choose Reducer Count Wisely
- Too few → bottleneck
- Too many → overhead from many small files
- Balance based on data size and cluster resources

### Avoid collect() on Large Data
Always save large results to storage, never bring to driver.

---

## Key Takeaways

### Why MapReduce Matters
- Teaches distributed thinking
- Foundation for understanding Spark
- Still in legacy systems (banking, telecom, government)

### Why Spark Won
- In-memory = 10-100x faster
- Developer friendly (SQL-like)
- Unified engine (batch + streaming + ML)
- Industry standard for modern data engineering

### Core Mental Model
**Week 2:** Where to store data (HDFS)  
**Week 3:** How to process data at scale (MapReduce → Spark)

Understanding both is essential for ML/DS engineering roles in the Canadian job market.