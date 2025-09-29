# Week 3: Distributed Processing Fundamentals

## Topics Covered
- About Distributed Processing
- Getting Started with Distributed Processing
- Distributed Processing Continuation
- Changing the Number of Reducers
- Use Case 1 - Sensor Data Example
- Real-time Industry Use Case of Distributed Processing
- Distributed Computing Demo
- Apache Spark
- Getting to know Apache Spark
- Apache Spark Vs Databricks
- Spark Execution Plan
- Word count example in Apache Spark

---

## 1 - Getting Started with Distributed Processing

### 🔹 Why Distributed Processing?

Last week you learned about distributed storage (HDFS). That solves where to store huge data.

But storage alone isn't enough — we must process petabytes.

Traditional programs (Python, Java, etc.) assume all data sits on one machine → fails when data is distributed.

**Solution:** Distributed processing frameworks like MapReduce.

### 🔹 MapReduce Basics

- It's a programming paradigm (not a language) for distributed systems
- Everything is represented as **(key, value)** pairs
- Example: (rollno, name) = (1, Manyue)
- Mapper input/output and Reducer input/output are all key-value pairs

**Two phases:**
- **Map phase** → parallel work across nodes
- **Reduce phase** → aggregates results

👉 If your data isn't naturally key-value (like plain text), **RecordReader** converts it:

```
Input: "Hello world" (line of text)  
Output: (1, "Hello world")
```

### 🔹 Example: Word Count

Let's say we want to count word frequency in a 500 MB file split into 4 blocks across 4 DataNodes.

**File:**
```
Hello Hadoop
Hello Manyue
```

#### 1. Input Split & RecordReader
File is split into blocks. Each line is turned into a key–value pair like:
```
(0, "Hello Hadoop")
(1, "Hello Manyue")
```

#### 2. Mapper Phase
Mapper code splits text into words and emits:
```
("Hello", 1)
("Hadoop", 1)
("Hello", 1)
("Manyue", 1)
```

#### 3. Shuffle & Sort (Framework Magic ✨)
Keys from all mappers are collected.

Framework groups values by key:
```
("Hello", [1, 1])
("Hadoop", [1])
("Manyue", [1])
```

#### 4. Reducer Phase
Reducer sums values for each key:
```
("Hello", 2)
("Hadoop", 1)
("Manyue", 1)
```

### 🔹 Mapper vs Reducer Design Tip

**Do as much work in the Mapper as possible.**

**Why?** Because Mappers run in parallel across many nodes.

If you push heavy work to the Reducer, it all ends up on one machine after shuffle — defeating the purpose of "distributed" processing.

### 🔹 How Many Mappers / Reducers?

**Mappers** = directly tied to number of blocks.
- Example: 1 GB file → 8 blocks (128 MB each) → 8 Mappers

**Reducers** = chosen by programmer (configurable).
- If 1 reducer → single output file
- If 5 reducers → 5 output files
- Tradeoff: more reducers = more parallelism in aggregation, but too many small files = overhead

### 🔹 Key Takeaways

- **Code → goes to data, not data → code** (data locality principle)
- **RecordReader** bridges "raw input" to "key-value pairs"
- **Map phase** = parallel, **Reduce phase** = aggregation bottleneck
- **Shuffle & Sort** = most expensive step (network transfer)
- Good design = maximize mapper work, minimize reducer load

⚠️ **"All of the 4 mappers output go to any of one datanode and reducer's work will start"**

That's not fully accurate — reducers don't just sit on one node.

- The framework decides how many reducers run and on which nodes
- The shuffle phase sends grouped keys to the correct reducer (could be spread across cluster)
- So, reducers also run distributed, not only on "one node"

---

## 2 - Distributed Processing Continuation

### 🔹 MapReduce Recap with Continuation

- **Mapper phase** → parallel work, happens on each block of data at DataNodes
- **Reducer phase** → aggregation, happens after shuffle + sort (where data actually moves)
- **Golden rule** → Do heavy-lifting in mappers, keep reducers light

**Why?** If reducers do too much, all data ends up concentrated on a few nodes = bottleneck = monolithic again.

### 🔹 LinkedIn Profile Views Example

**Input file (simplified):**
```
[1,Manasa,Sumit]
[2,Deepa,Sumit]
[3,Sumit,Manasa]
[4,Sumit,Deepa]
[5,Sumit,Manasa]
[6,Manasa,Deepa]
```

#### Step 1 – Record Reader Output (line → key, value):
```
(0, [1,Manasa,Sumit])
(1, [2,Deepa,Sumit])
...
```

#### Step 2 – Mapper Output (emit key = profile viewed, value = 1):
```
Sumit,1
Sumit,1
Manasa,1
Deepa,1
Manasa,1
Manasa,1
```

#### Step 3 – Shuffle & Sort (framework groups by key):
```
Deepa, {1}
Manasa, {1,1,1}
Sumit, {1,1}
```

#### Step 4 – Reducer Aggregation:
```
Deepa,1
Manasa,3
Sumit,2
```

✅ This is the final result: how many times each profile was viewed.

### 🔹 Number of Mappers

**Formula:**
```
No. of Mappers = No. of Blocks = File size ÷ Block size
```

**Example:** 1 GB file, 128 MB block size → 8 mappers created.

⚠️ **Parallelism:** depends on cluster resources (nodes × slots per node).

- If each node has 4 mapper slots → 4 nodes × 4 slots = 16 mappers can run in parallel
- In a smaller config, maybe only 4 run at a time
- So: 8 mappers created, but maybe only 4 run simultaneously

### 🔹 Number of Reducers

Controlled by the developer (user).

**By default:** 1 reducer.

But you can set:
- **0 reducers** → no aggregation (just mapper output)
- **N reducers** → framework will distribute keys across reducers

### ⚡ Quick Analogy:

**Mappers** = students writing answers in parallel.

**Reducers** = teacher collecting and tallying scores.

If the teacher does all the work, it's slow.

If students do most of the work (mapper-heavy), the teacher just adds up → very efficiently.

---

## 3 - Changing the Number of Reducers

### Why Change the Default (1 Reducer)?

**Problem:** All mappers may finish in 2 minutes, but the single reducer takes 10 minutes.
- Total = 12 minutes (bottleneck at the reducer)

**Solution:** Increase the number of reducers to parallelize aggregation.

**Rule of Thumb:** Push maximum work to the mappers, keep reducers as light as possible.

### When to Set Reducers = 0

Some jobs don't require aggregation.

**Example:** Filtering records.

In such cases:
- Mapper output itself is the final output
- No shuffling or sorting
- Example: 1 GB file = 8 blocks = 8 mappers → final = 8 output files (mapper outputs)

### Partitioning, Hash & Reducers

**Partitioning** decides which reducer gets which key-value pairs.

- Happens on mapper nodes after map tasks finish
- Ensures keys are distributed across reducers

**Default: Hash Partitioning**
```
partition = hash(key) % num_reducers
```

- Same key → always goes to the same reducer (consistency)
- Prevents reducer output from being incorrect

### Flow with Example

Suppose we are doing Word Count on a 1 GB file split into 8 blocks.

#### Step 1: Mapper Output
```
Mapper 1 → (apple,1), (banana,1)
Mapper 2 → (apple,1), (carrot,1)
Mapper 3 → (banana,1), (apple,1)
```

#### Step 2: Partitioning (Hash-based)
Num reducers = 2

Apply hash(key) % 2:
- apple → partition 0
- banana → partition 1
- carrot → partition 1

Partitions look like:
```
Partition 0 → (apple,1), (apple,1), (apple,1)
Partition 1 → (banana,1), (banana,1), (carrot,1)
```

#### Step 3: Shuffle
- Partition 0 → Reducer 1 machine
- Partition 1 → Reducer 2 machine

#### Step 4: Sort (at Reducer side)
```
Reducer 1: (apple, {1,1,1})
Reducer 2: (banana,{1,1}), (carrot,{1})
```

#### Step 5: Reduce
```
Reducer 1: apple → 3
Reducer 2: banana → 2, carrot → 1
```

✅ **Final Output:**
```
apple,3
banana,2
carrot,1
```

### Custom Partitioning

When hash isn't enough, you can define your own partitioner.

**Example:**
- Words < 4 chars → Reducer 1
- Words ≥ 4 chars → Reducer 2

Custom logic replaces `hash(key) % num_reducers`.

### Key Points to Remember

- **Reducers = 1** (default): All keys go to one reducer
- **Reducers > 1**: Mapper output partitioned across reducers via hash
- **Reducers = 0**: No shuffle/sort; mapper output = final output
- Partitioning is mapper-side, sorting is reducer-side
- Custom partitioning can override default hash-based partitioning

### Analogy

Think of reducers as checkout counters in a supermarket:

- If there's only 1 counter → everyone queues at 1 place (bottleneck)
- If there are many counters → customers (keys) are distributed based on a rule (hash or custom)
- The same customer (key) always goes to the same counter to get the right bill

### Concrete Example – Employee Salaries

**Dataset:**
```
Dept,Employee,Salary
IT,Alice,5000
IT,Bob,6000
HR,Carol,4000
HR,Dan,4500
Finance,Eve,7000
Finance,Frank,8000
```

**Mapper Output (key = Dept, value = Salary):**
```
IT,5000
IT,6000
HR,4000
HR,4500
Finance,7000
Finance,8000
```

**Partitioning (3 reducers = 3 partitions):**
- Reducer 1 → IT
- Reducer 2 → HR
- Reducer 3 → Finance

**Shuffle & Sort (keys grouped):**
```
IT → {5000, 6000}
HR → {4000, 4500}
Finance → {7000, 8000}
```

**Reducer Aggregation:**
```
IT → 11,000
HR → 8,500
Finance → 15,000
```

✅ Parallelism is achieved because reducers worked independently, instead of one reducer handling everything.

### 👉 Summary:

- Reducer count matters for balancing workload
- 0 reducers = no aggregation
- More reducers = parallel aggregation, but not "free"—too many reducers can increase overhead
- Data flow = Map → Partition → Shuffle → Sort → Reduce

---

## 4 - Use Case 1 - Sensor Data Example

### Problem

Sensor collects temperature data:
```
Date,Time,Temp
12/12/2015,00:00,50
12/12/2015,01:00,52
12/12/2015,02:00,49
12/13/2015,00:00,48
12/13/2015,01:00,54
12/13/2015,02:00,55
```

**Goal:** Find the maximum temperature per day.

**Expected output:**
```
12/12/2015,52
12/13/2015,55
```

Data size = 300 MB → 3 blocks → 3 mappers (3 nodes).

### Flow without Combiner (Naive Way)

1. **RecordReader input to Mapper**
   - Default input = (offset, line)
   - Example: (0, "12/12/2015,00:00,50")

2. **Mapper parses line**
   - Output: (date, temp)
   - Example: (12/12/2015, 50)

3. **Partitioning (mapper-side)**
   - By default: hash(date) % numReducers
   - Ensures the same key always goes to the same reducer

4. **Shuffle (network)**
   - Mapper outputs (all pairs) are sent to reducer nodes

5. **Sort (reducer-side, framework)**
   - Group all values for the same key
   - Example:
     ```
     12/12/2015 → {50,52,49}
     12/13/2015 → {48,54,55}
     ```

6. **Reducer function**
   - For each key, find max(temp)
   - Output:
     ```
     12/12/2015,52
     12/13/2015,55
     ```

⚠️ **Problem:**
- Every reading is sent across the network
- Reducer does most of the work → bottleneck

### Flow with Combiner (Optimized)

1. **RecordReader → Mapper**
   - Same as before: (offset, line) → parse → (date, temp)

2. **Combiner (on mapper node)**
   - Acts as "local reducer" before shuffle
   - For each date seen by this mapper, keep only local max
   - Example mapper block had:
     ```
     (12/12/2015, 50), (12/12/2015, 52), (12/12/2015, 49)
     → Combiner outputs: (12/12/2015, 52)
     ```

3. **Partitioning**
   - Same hash logic to decide reducer destination

4. **Shuffle**
   - Now only local maxima per key goes over the network

5. **Sort (reducer-side)**
   - Example:
     ```
     12/12/2015 → {52}
     12/13/2015 → {55,54}
     ```

6. **Reducer**
   - Final max across combined values
   - Output:
     ```
     12/12/2015,52
     12/13/2015,55
     ```

✅ **Benefits:**
- Far less data shuffled
- Mappers contribute computer work
- Reducers are lighter → job faster

### When to Use Combiners

Safe for **associative + commutative** functions:
- ✅ Max, Min, Sum, Count
- ❌ Average (because avg(avg1, avg2) ≠ true avg)

**Fix for Avg:**
- Mapper+Combiner output = (sum, count)
- Reducer aggregates sums and counts → final avg

### Final Flow (with Combiner)

```
RecordReader → Mapper → Combiner → Shuffle → Sort → Reducer
```

### Key Takeaways:

- Mapper input = (offset, line) → you parse it
- Shuffle = moves mapper/combiner output
- Sort = framework groups same keys before reducer
- Combiner = local aggregation to reduce shuffle load

---

## 5 - Real-time Industry Use Case of Distributed Processing

### 1. The Context: Why Google Needed This

**Google's core product** = Web Search.

To power it, they run web crawlers that fetch content from billions of websites.

The crawlers produce data like:
```
flipkart.com   clothes handbag laptop
amazon.com     clothes mobile purse
myntra.com     purse clothes tv
```

This raw format is URL → list of keywords found.

**Problem:** Search needs the inverse structure:
```
clothes → amazon.com, flipkart.com, myntra.com
purse   → amazon.com, myntra.com
```

👉 This is called **Inverted Indexing**.

👉 Without it, Google can't answer queries like "show me all websites with clothes."

### 2. Why MapReduce Was Needed

Before MapReduce, you could try:

Writing a single program to read all 40B+ pages and invert mappings.

**Problem:** Too much data to fit on one machine.

Traditional systems → one machine = one failure point, not scalable.

**Google needed a way to:**
- Split the work across thousands of machines
- Handle failures gracefully
- Still produce one consistent global index

This is where MapReduce became a game-changer.

### 3. The MapReduce Flow for Inverted Index

Let's see how it works:

**Input: Raw crawler output**
```
flipkart.com clothes handbag laptop
amazon.com   clothes mobile purse
myntra.com   purse clothes tv
```

#### Step 1: Record Reader
Splits each line into (key, value) pairs.

Example:
```
(0, "flipkart.com clothes handbag laptop")
(1, "amazon.com clothes mobile purse")
(2, "myntra.com purse clothes tv")
```

#### Step 2: Mapper
Mapper logic: take each word in the value, output (word, website).

**Example output:**
```
(clothes, flipkart.com)
(handbag, flipkart.com)
(laptop, flipkart.com)
(clothes, amazon.com)
(mobile, amazon.com)
(purse, amazon.com)
(purse, myntra.com)
(clothes, myntra.com)
(tv, myntra.com)
```

#### Step 3: Shuffle
Framework groups the same keys together, across all mappers.

Data moves across the network to partition keys consistently (via hash or custom logic).

**Output after shuffle:**
```
clothes → [flipkart.com, amazon.com, myntra.com]
handbag → [flipkart.com]
laptop  → [flipkart.com]
mobile  → [amazon.com]
purse   → [amazon.com, myntra.com]
tv      → [myntra.com]
```

#### Step 4: Sort
Ensures all identical keys are lined up together before reducer runs.

Example: all (clothes, …) pairs grouped.

#### Step 5: Reducer
Reducer takes the grouped values and aggregates them.

Example logic: remove duplicates.
```
clothes → {flipkart.com, amazon.com, myntra.com}
purse   → {amazon.com, myntra.com}
```

### 4. Why Not Directly Produce "Search-Friendly" Key-Values?

Good doubt. Why not design a crawler to directly output (word, website)?

- Crawlers work in parallel, each crawling different sites. They don't know the global picture
- You need a global shuffle + reduce step to group "all clothes entries" from millions of crawlers
- Without MapReduce: every crawler would need to know every other crawler's work → impossible at Google scale

**MapReduce solved this by:**
- Letting each crawler just dump raw data
- The system itself takes care of distributing, shuffling, and reducing

### 5. Why MapReduce Made Google Search Work

- Crawlers produce petabytes of raw data
- MapReduce allowed Google to parallelize inverted indexing across 1000s of machines
- **Fault-tolerance:** if one node dies while processing amazon.com, the task just restarts on another node
- **Scalability:** as the web grew, Google just added more machines to the cluster

✅ **Key Takeaway:**

Google Search is powered by an inverted index, and building it at web scale required MapReduce. Without it, the problem of grouping billions of keywords into one searchable index was practically unsolvable.

---

## 6 - Distributed Processing Glossary

### Record Reader
Component that converts raw input (line, bytes, etc.) into key–value pairs.

**Example:** A line "12/12/2015,00:00,50" becomes (0, "12/12/2015,00:00,50") where 0 is the key (offset).

### Mapper
User-defined function that processes each key–value pair from the Record Reader.
- Outputs new key–value pairs (often transforming/filtering)
- Runs in parallel on each block of data (data locality principle)

### Combiner (Local Aggregator)
Mini-reducer that runs on mapper's output before shuffle.
- Reduces volume of data transferred across the network
- Safe for associative & commutative operations (sum, min, max)
- ⚠️ Not safe for avg unless you redesign logic (emit sum + count)

### Partitioner
Decides which reducer gets which key.
- **Default** = HashPartitioner (hash(key) % numReducers)
- Ensures same key always goes to the same reducer
- **Custom Partitioner** → lets you define your own rules (e.g., short words → Reducer1, long words → Reducer2)

### Shuffle
Movement of mapper outputs across the network to their respective reducers.
- Happens after partitioning

### Sort
At reducer side, groups values of the same key together.

Input to reducer becomes:
```
key → {v1, v2, v3 …}
```

### Reducer
User-defined function that aggregates/group-processes the values of each key.
- Produces the final output
- Example: (hello, {1,1,1}) → (hello, 3)

### Number of Reducers
Controlled by developer (can be 0, 1, or more).
- Reducers = Partitions
- **0 reducers** → mapper output is final (no shuffle/sort)
- **Too few reducers** → bottleneck
- **Too many reducers** → overhead, many small files

### 🔑 Big Picture Flow:
```
Record Reader → Mapper → (optional Combiner) → Partitioner → Shuffle → Sort → Reducer → Final Output
```

---

## Apache Spark

## 1 - Getting to know Apache Spark

### Apache Spark Core Concepts Explained

#### Recap: Hadoop's Three Pillars
- **HDFS** → Storage
- **MapReduce (MR)** → Processing
- **YARN** → Resource Management

### Why MapReduce Became a Bottleneck

1. **Performance** → Too many Disk I/Os
   - Each MR job reads from disk and writes back to disk after every step
   - Chain of 5 MR jobs → 10 Disk I/Os

2. **Developer Experience** → Not friendly
   - Everything must be expressed as (key, value) → map → shuffle → reduce
   - Even simple problems require bending logic into this rigid paradigm

3. **Batch Only** → No native streaming or interactive querying

4. **Ecosystem Sprawl** → Needed separate tools like Hive, Pig, Sqoop for different tasks

### Enter Apache Spark

Designed as a distributed compute engine that is:
- **Fast** → In-memory processing
- **Flexible** → Batch, streaming, ML, SQL, graph
- **Developer Friendly** → APIs in Python, Java, Scala, R
- Works with any storage: HDFS, S3, ADLS, GCS, or even local FS
- Works with any resource manager: YARN, Mesos, Kubernetes

👉 **In Hadoop ecosystem:**
```
HDFS | Spark | YARN
```

### Spark's Advantage: In-Memory

Spark keeps intermediate results in RAM (RDDs / DataFrames) instead of writing to disk.

**Only 2 disk I/Os:**
1. Load from storage
2. Write back the final output

Chain of 5 transformations in Spark → all happen in-memory → finally one write.

**Why not MR do the same?**

MR was designed in 2003–2004 (Google paper → Hadoop). RAM was expensive, clusters were disk-heavy.

Spark (2010+) was designed in the era of cheaper RAM, hence built around in-memory abstractions.

### Developer Experience

**MapReduce:** You must design how mappers & reducers work.

**Spark:** You just declare what you want:
```python
df.groupBy("category").count()
```
Spark handles distribution & execution plan internally.

👉 Spark abstracts away "distributed thinking" → more like SQL programming.

### Disk IO & In-Memory Reads

Both Hadoop MR and Spark run on clusters of machines.

**Each machine = a server (node) → it has:**
- CPU
- Disk (HDD/SSD)
- RAM (memory)
- Network card

So when we say "distributed processing," it just means: use many machines' CPUs, RAM, and disks together.

👉 Hadoop MR and Spark differ mainly in how they use disk vs RAM.

### MapReduce Execution

#### Step 1 – Read Input
- Input file (say 500 MB) split into blocks (128 MB)
- Blocks live on DataNodes' disks
- MR sends code → runs mapper on the same node (data locality)

#### Step 2 – Mapper Output
- Mapper processes block and produces key-value pairs
- But mapper's output is not final → reducers need it
- **Problem:** Reducer might be on another node
- So mapper output must be saved to disk (local temp file) before being sent

#### Step 3 – Shuffle & Reduce
- Reducer pulls mapper outputs across the network
- Before reducer can use them, it reads from disk again (mapper's local file)
- Then reducer writes final result → back to HDFS (disk)

⚠️ **If you chain 5 MR jobs:**
- Each stage = read from disk → process → write to disk
- Disk I/O balloons. That's why MR feels slow.

### Spark Execution

#### Step 1 – Read Input
Same as MR → Spark reads from HDFS/S3/ADLS.

#### Step 2 – Transformations (in Memory)
- Spark represents data as RDDs / DataFrames (in-memory objects)
- If you run:
  ```python
  df.filter(...).groupBy(...).agg(...)
  ```
- Spark keeps intermediate results in RAM

👉 **Instead of:**
```
Disk → CPU → Disk → CPU → Disk
```

**You get:**
```
Disk → RAM → CPU → RAM → CPU → … → Disk (final)
```

#### Step 3 – Only Write Final Output
- After all transformations, Spark writes back to HDFS/S3
- Only two disk touches:
  1. Load from storage
  2. Save result

### Where RAM Fits In

Think of RAM as scratchpad memory for each node:

- **Disk** = library shelves (slow to pick/return books)
- **RAM** = your desk (fast, but limited size)
- **CPU** = you, actually reading/writing

**MapReduce** → every time you finish a paragraph, you put the book back on the shelf → next time, fetch again.

**Spark** → you keep the notes on your desk until the whole chapter is done → then return final book.

### Narrowing Further: Example

Say we want to compute average temperature per day from 500 MB sensor logs.

#### MapReduce Flow
- Mapper → read block from HDFS → write intermediate (day,temp) pairs to disk
- Reducer → fetch intermediate files → compute avg → write final avg to HDFS
- If you want another calculation (say max temp), repeat entire process with new disk reads/writes

#### Spark Flow
- Read file → keep as DataFrame in RAM
- Run `.groupBy("day").avg("temp")` → executed in memory
- Run `.groupBy("day").max("temp")` → same data in RAM, no need to reload from HDFS
- Write both results to HDFS

👉 Notice how Spark reuses the in-memory dataset. MR cannot.

### Key Takeaway

Both run on clusters with CPU + RAM + Disk.

**The difference is what resource becomes the main workhorse:**
- **MR** = disk-heavy → always spill intermediate results to local disk
- **Spark** = RAM-heavy → cache data in memory between steps

That's why Spark feels like "in-memory distributed processing."

---

## Apache Spark Vs Databricks

### Apache Spark

- Open-source distributed processing engine
- Created at UC Berkeley → donated to Apache Software Foundation
- Can run on:
  - On-prem clusters
  - Hadoop YARN
  - Kubernetes
  - Standalone

⚡ **Spark = only the engine.** You install + manage everything yourself.

### Databricks

A company founded by Spark creators.

Their product **Databricks Platform** = Spark + extra features:
- Cloud-native (AWS, Azure, GCP)
- Optimized Spark runtime → faster than open-source Spark
- Cluster management (easy spin-up/down, auto-scaling)
- Delta Lake (ACID on data lakes → core DE topic later)
- Collaborative notebooks (like Jupyter but built-in)
- Security + governance features
- Many more (Unity Catalog, MLflow, AI integration)

👉 **Think of it like:**
- **Spark** = free car engine
- **Databricks** = Tesla → same engine inside, but with battery, software, dashboard, autopilot, ready-to-use

### Spark APIs

Apache Spark has layers of abstraction:

#### 1. Core API → RDDs
- **RDD** = Resilient Distributed Dataset
- Low-level API (original Spark way)
- Very flexible: you can solve any problem here
- Hard to write & optimize → like coding in assembly
- Rarely used directly in modern DE unless custom logic is needed

#### 2. Higher-level APIs
Built later to make developer's life easier:
- **DataFrames** → tabular data (like Pandas, SQL tables)
- **Spark SQL** → SQL queries directly on DataFrames
- **Structured Streaming** → near real-time pipelines
- **MLlib** → ML library
- **GraphX** → graph processing

💡 **Recommended:**
- Always start with Spark SQL or DataFrames
- If not solvable → drop to RDD

⚖️ **Flexible = "Can I solve any type of problem here?"**
- **SQL** → Limited to relational-style problems
- **DataFrames** → Broader, still structured
- **RDD** → Anything, but harder

### 3 Core Steps in Spark Workflows

1. **Load** → Read from HDFS, S3, ADLS, GCS, etc.
2. **Transform** → Filter, join, groupBy, aggregate
3. **Write** → Save to target (data lake, DB, warehouse)

### RDD = The Basic Unit of Spark

**"RDD is the fundamental data structure in Spark."**

**Meaning:**
- Spark doesn't operate on raw files directly
- It always represents data as RDDs internally
- Even DataFrames/Spark SQL are abstractions built on top of RDDs

**Example flow:**
- You `read.csv("file.csv")` → Spark actually creates an RDD of rows
- When you run `df.groupBy("col")`, Spark translates it into RDD transformations under the hood
- So → everything in Spark eventually boils down to RDD operations

### ✅ Final Mental Picture

- **Spark** = distributed compute engine
- **RDD** = core foundation (like bricks in a building)
- **DataFrame & SQL** = walls/rooms built on bricks (easy for humans)
- **Databricks** = the full furnished house, ready to live in

---

## Spark Execution Plan

### 1. Blocks vs Partitions

**Block** → Physical data stored on disk (HDFS, S3, ADLS, etc.).
- Example: A 512 MB file → 4 blocks of 128 MB each

**Partition** → Logical slice of data in memory (RAM).
- When Spark reads from disk, it loads blocks into RAM as partitions

**RDD (Resilient Distributed Dataset)** → Collection of partitions spread across cluster memory.

**In short:**
- Disk → Block
- RAM → Partition
- Distributed dataset in memory → RDD

### 2. Execution Flow

**Example:**
```python
rdd1 = sc.textFile("file1")        # Load
rdd2 = rdd1.map(...)               # Transformation
rdd3 = rdd2.filter(...)            # Transformation
result = rdd3.collect()            # Action
```

- **Transformations** (map, filter) are **lazy** → Spark does not execute them immediately. Instead, it builds a plan
- **Action** (collect, count, saveAsTextFile) triggers execution → Spark now runs all transformations in sequence

### 3. DAG (Directed Acyclic Graph)

Spark builds a DAG internally.

**Example:** Load → Map → Filter → Collect becomes a DAG of stages.

DAG ensures that Spark can:
- Optimize execution (e.g., push filter before map to minimize records processed)
- Recover from failures using lineage

### 4. Why RDDs are Resilient

- Every RDD remembers its parent RDD and the transformation that created it
- If rdd3 is lost (say a node crashes), Spark can recompute it from rdd2 (lineage)
- RDDs are **immutable** → they never change in place. Every transformation produces a new RDD
- This immutability + lineage makes Spark resilient
- (Compare with HDFS: resilience comes from replication factor; in Spark it comes from lineage)

### 5. Why Lazy Evaluation?

Prevents unnecessary data movement.

**Example:** File with 1B rows:
```python
rdd1 = sc.textFile("bigfile")
rdd2 = rdd1.map(...)
rdd3 = rdd2.filter(lambda x: condition)
rdd3.first()
```

- If eager, Spark would load all 1B rows, do map, then filter
- With lazy evaluation, Spark knows only `first()` is needed, so it:
  - Loads minimal data
  - Pushes filter above map to reduce work
  - **Result:** much faster and efficient

### 6. Execution in Cluster (Driver + Workers)

- **Driver**: The brain. Builds the DAG and sends tasks to executors
- **Workers (executors)**: Where partitions of RDD are stored and processed
- Processing happens close to the data (data locality)

### Visual Recap:

1. File in HDFS → split into blocks on disk
2. Blocks → loaded as partitions in RAM → form an RDD
3. Each transformation creates a new RDD, but execution is delayed
4. When an action is called → Spark executes the DAG, stage by stage

---

## Word count example in Apache Spark

### 1. Spark Session Setup

**SparkSession** is the entry point to any Spark cluster.

Introduced in Spark 2.x, it unifies:
- SparkContext
- SQLContext
- HiveContext

**Current version:** Spark 3.x

```python
from pyspark.sql import SparkSession
import getpass
username = getpass.getuser()

spark = SparkSession. \
    builder. \
    config('spark.ui.port', '0'). \
    config("spark.sql.warehouse.dir", f"/user/{username}/warehouse"). \
    enableHiveSupport(). \
    master('yarn'). \
    getOrCreate()
```

This creates a Spark session running on YARN with Hive support enabled.

### 2. Steps in Spark

In Spark, typical workflow = **3 steps:**

1. Load data from storage (HDFS, S3, ADLS, local, etc.)
2. Apply transformations (map, filter, groupBy, etc.)
3. Save or collect results

### 3. Word Count Example (RDD API)

```python
# Step 1: Load file (each line → 1 element)
rdd1 = spark.sparkContext.textFile("/user/itv021666/data/inputfile.txt")

# Step 2: Split lines into words
rdd2 = rdd1.flatMap(lambda line: line.split(" "))

# Step 3: Map each word → (word, 1)
rdd3 = rdd2.map(lambda word: (word, 1))

# Step 4: Reduce by key (sum counts)
rdd4 = rdd3.reduceByKey(lambda x, y: x+y)

# Step 5: Collect small results / Save large results
rdd4.collect()
rdd4.saveAsTextFile("/user/itv021666/data/newoutput")
```

**Flow:**
```
Line → Words → (Word,1) → Group & Sum → Result
```

- `flatMap` → splits & flattens
- `map` → pairs words with 1
- `reduceByKey` → aggregates counts per word
- `collect` → bring result to driver (use only for small data)
- `saveAsTextFile` → write results to HDFS (preferred for large data)

⚠️ **Warning:**

`collect()` brings all data to the gateway node/driver.

If output is huge (e.g., 2TB), this can cause out-of-memory error.

Instead, save directly to HDFS:
```python
rdd4.saveAsTextFile("/user/itv021666/data/newoutput")
```

### 4. Complete Word Count Code

```python
rdd1 = spark.sparkContext.textFile("/user/itv021666/data/inputfile.txt")
rdd2 = rdd1.flatMap(lambda line: line.split(" "))
rdd3 = rdd2.map(lambda word: (word, 1))
rdd4 = rdd3.reduceByKey(lambda x, y: x+y)

# Collect small results
rdd4.collect()

# Save large results to HDFS
rdd4.saveAsTextFile("/user/itv021666/data/newoutput")
```

### 5. Key Concepts Recap

- **SparkSession** = entry point to cluster
- **RDD** = basic distributed collection in Spark
- **Transformations** (lazy):
  - flatMap, map, reduceByKey
- **Actions** (trigger execution):
  - collect, saveAsTextFile
- **Lazy Execution**:
  - Spark builds a DAG (execution plan) first
  - Actual execution happens only when an action is called
- **Why not always collect()?**
  - Use for small results only
  - For large results → save to HDFS