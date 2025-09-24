# Week 2: Distributed Storage & Data Lake

## Topics Covered
- Datalake Storage & Getting Started with the Labs
- HDFS Overview
- About Practice Labs
- Linux Commands - cd & ls
- More Linux Commands
- HDFS Commands
- More About Practice Lab
- HDFS Vs Cloud Data Lake

---

## 1 - HDFS Overview

### Architecture: Master–Slave

**NameNode (Master)** → stores metadata (mapping of file → blocks → DataNodes)

**DataNodes (Slaves)** → store the actual blocks of data

### Example:
- **File** = file1.txt, size = 500 MB
- **Default block size** = 128 MB
- **File is split into:**
  - B1 = 128 MB
  - B2 = 128 MB
  - B3 = 128 MB
  - B4 = 116 MB

**Blocks are distributed across 4 DataNodes**

NameNode stores metadata like:

| File  | Block | Stored In |
|-------|-------|-----------|
| file1 | B1    | DN1       |
| file1 | B2    | DN2       |
| file1 | B3    | DN3       |
| file1 | B4    | DN4       |

When a client requests file1.txt, it contacts the NameNode, which provides block locations. The client then directly reads from DataNodes.

### 👉 Analogy:
A 1000-page book with an index page:
- **Index (NameNode)** = knows where each chapter starts
- **Pages (DataNodes)** = hold the actual content

### What is a Node / DataNode Really?

**Yes** — a DataNode = a server (machine) in the cluster.

A cluster is a group of such servers (nodes) working together.

**Example:** A 4000-node Hadoop cluster = 4000 physical/virtual machines, each running DataNode software.

All nodes are usually identical servers with similar CPU, memory, disk capacity.

### Block Size (128 MB Default)

**Why not smaller or larger?**

- **Smaller block size (64 MB)** → more blocks → more parallelism → but more metadata entries in NameNode (burden increases)
  - Example: 1 GB file = 16 blocks → NameNode must track 16 mappings

- **Larger block size (256 MB)** → fewer blocks → less parallelism → but lighter load on NameNode
  - Example: 1 GB file = 4 blocks

⚖️ **128 MB was chosen as a balance between parallelism and NameNode overhead**

### Scalability of NameNode

- Data nodes can be easily increased to store more data
- But NameNode also grows in metadata size → can become a bottleneck
- **Solution:** NameNode Federation → multiple independent NameNodes manage different namespaces (e.g., sales data → NN1, finance data → NN2)
- This allows horizontal scaling of metadata management

### Fault Tolerance

#### Replication Factor (Default = 3)
- Each block is stored on 3 different DataNodes
- Ensures data availability even if 1–2 nodes fail

#### Rack Awareness
- A rack = group of servers in one location
- Replicas are placed on different racks (e.g., Bangalore, SF, Canada)
- Ensures survival from rack-level failures (power outage, flood, etc.)

#### NameNode Failure
- **In Hadoop 1.x** → Single Point of Failure
- **Secondary NameNode** was introduced → but ⚠️ not a true backup. It only checkpoints metadata. If NameNode crashes, admin has to restore manually
- **In Hadoop 2.x** → NameNode High Availability (HA) with Active + Standby nodes (automatic failover)

### Quick Recap
- **NameNode** → Metadata brain of HDFS
- **DataNode** → Stores actual blocks
- **Block Size** → 128 MB default (tradeoff between parallelism vs NameNode overhead)
- **Replication Factor** → Fault tolerance for DataNode failures
- **Rack Awareness** → Resilience against rack-level outages
- **Secondary NameNode** → Metadata checkpoint, not a backup
- **NameNode Federation** → Scalability for metadata

---

## 2 - Lab Practice:

**Cluster Details:**
Gateway nodes | master node(name node) | worker node(Data node)

---

## 3 - Linux Commands - cd & ls

### Linux Basics – cd & ls

#### Linux File System Hierarchy
Everything starts at the root directory `/` → the parent of all files and folders.

Linux structure is like a tree:
- `/` = root
- `/home` = contains user home directories
- `/home/username` = your personal directory

**Example:**
```bash
[itv021666@g01 ~]$ pwd
/home/itv021666
```

`pwd` → present working directory.

Here, you're inside your home directory.

#### User Identification
`whoami` → prints the currently logged-in user.

```bash
[itv021666@g01 ~]$ whoami
itv021666
```

#### Changing Directories (cd)
`cd` → move between directories.

**Examples:**
```bash
cd /        # go to root
cd ~        # go to home directory
cd ..       # go one level up (parent directory)
cd ../..    # go two levels up
cd -        # go to previous directory
cd .        # stay in current directory (no effect)
```

#### Absolute vs Relative Paths
- **Absolute Path** → Full path from root `/`
  - Example: `/home/itv021666/Documents`
- **Relative Path** → Path relative to your current working directory
  - Example: `cd Documents` (works if you're already in `/home/itv021666`)

#### Listing Files & Directories (ls)
`ls` → basic listing.

**Colors:**
- Blue = directories
- Green = executables
- Black/White = normal files

**Variants:**
```bash
ls -l     # long listing (permissions, owner, size, date)
ls -lt    # sort by time (newest first)
ls -ltr   # sort by time (oldest first)
ls -lr    # reverse dictionary order (Z → A)
ls -R     # recursive list (all sub-directories)
ls -a     # show hidden files (start with .)
ls -latR  # combine: detailed, all files, sorted, recursive
```

Hidden files example: `.bash_history`

---

## 4 - More Linux Commands

### File & Directory Management

#### Create an empty file
```bash
touch file1
```
- Creates a new empty file
- If file exists → updates timestamp

#### Create directory
```bash
mkdir dir1
```

#### Remove directory
```bash
rmdir dir1        # only works if empty  
rm -R dir2        # force remove, even if not empty  
```

#### Remove files
```bash
rm file1
```

#### Copy files & directories
```bash
cp file1 file2        # copy file1 → new file2  
cp file1 dir3/        # copy file1 into dir3  
cp -R dir3 dir4       # copy entire directory (recursive)  
```

#### Move / Rename
```bash
mv file1 dir4/        # move file into dir4 (cut–paste)  
mv old.txt new.txt    # rename file  
```

### File Permissions

Permission format: Owner | Group | Others.

`rwx` → read (4), write (2), execute (1).

**Example:** 664
- Owner = read + write (6)
- Group = read + write (6)
- Others = read only (4)

Change permissions:
```bash
chmod 777 file1
# -rwxrwxrwx (all have full access)
```

### Viewing & Editing Files

#### cat → view contents of file
```bash
cat file1
```

#### head → see first 10 lines
```bash
head file1
```

#### tail → see last 10 lines
```bash
tail file1
```

#### word/line/char count
```bash
wc file1
```

#### merge files
```bash
cat file1 file2 >> merged.txt
```

#### create file with inline input
```bash
cat > orders.txt
# type content, press Ctrl+D to save & quit
```

### Editing with vi (Vim Editor)
```bash
vi samplefile
```
- Press `i` → insert mode → type content
- Press `Esc`, then `:wq` → save & quit
- `:q` → quit without saving

### Disk Usage
`du` = disk usage.
```bash
du -h /data/retail_db
```
`-h` = human-readable (KB, MB, GB).

### Searching
`grep` → search within files.
```bash
grep "word" file.txt
```

---

## 5 - HDFS Commands

### Basics
- `hadoop fs` → Lists all available HDFS commands
- `hdfs dfs` → Same as hadoop fs (just a newer alias)

⚠️ **Important:**
- `ls` (without prefix) = lists local gateway node files
- `hadoop fs -ls` = lists HDFS files

### Directories
```bash
hadoop fs -ls /                  # list root of HDFS
hadoop fs -ls /user/itv021666    # list user's home dir in HDFS
hadoop fs -mkdir /user/itv021666/retail_db
hadoop fs -mkdir -p /user/itv021666/retail_db/new_db  # create nested dirs
hadoop fs -rmdir empty_dir
hadoop fs -rm -R retail_db       # remove non-empty directory
```

**Note:**
- Local home = `/home/username`
- HDFS home = `/user/username`

### Listing Variants
```bash
hadoop fs -ls -t /     # newest files first
hadoop fs -ls -t -r /  # oldest files first
hadoop fs -ls -s /     # sort by size
hadoop fs -ls -s -h /  # human-readable size
hadoop fs -ls -R /     # recursive list
hadoop fs -ls /user | grep keyword   # search results with grep
```

### File Operations

#### Upload (local → HDFS):
```bash
hadoop fs -put bigLog.txt /user/itv021666/
hadoop fs -copyFromLocal bigLog.txt /user/itv021666/
```

#### Download (HDFS → local):
```bash
hadoop fs -get bigLog.txt /localpath/
hadoop fs -copyToLocal bigLog.txt /localpath/
```

#### Within HDFS:
```bash
hadoop fs -cp /src/file /dest/file      # copy
hadoop fs -mv /src/file /dest/file      # move/rename (metadata only, not full rewrite)
```

### File Viewing
```bash
hadoop fs -cat /user/.../file.txt     # show file contents
hadoop fs -tail /user/.../file.txt    # show last 1KB of file
```

⚠️ These talk to DataNodes (since they fetch actual file content), while commands like `-ls` only talk to NameNode.

### Health & Space
```bash
hadoop fs -df -h              # disk usage in human readable format
hdfs fsck /path/file -files -blocks -locations
```

`fsck` → checks file health, shows block info, replication, locations.

---

## 📖 The Story of Facebook & Hadoop

### 1. The Problem
Facebook is exploding with user data — photos, likes, comments, videos.

They need a system that can:
- Store petabytes of data
- Handle failures (a disk crash shouldn't mean data loss)
- Scale as users grow

So they decide: **We need Hadoop Distributed File System (HDFS)**

### 2. Setting up the Cluster
Facebook's infra team sets up a cluster of servers → racks filled with hundreds/thousands of machines.

Inside HDFS:
- **NameNode** → the librarian that knows where every block lives
- **DataNodes** → the actual bookshelves holding data blocks
- **Replication** → each file block is copied to 3 machines, across racks, so if one dies → no problem

So, Facebook now has a distributed storage warehouse: **HDFS**

### 3. How Employees Access It
Now, the question: how do thousands of engineers at Facebook touch this cluster?

You don't want engineers directly logging into NameNode or DataNodes (too risky).

So Facebook sets up a **Gateway Node** → like a reception desk to the warehouse.
- Engineers log into the gateway
- Gateway is just a Linux server with user accounts (like `/home/manny`, `/home/john`)
- From there, they run commands like:

```bash
hadoop fs -ls /user/manny
# → Gateway forwards the request to NameNode, then to DataNodes if needed
```

👉 Gateway is Facebook's responsibility (their cluster access point), not Hadoop's. Hadoop just manages storage logic.

### 4. Where Root & Home Come In

#### In Linux (Gateway machine):
- **Root** `/` = the top of the Linux file system on that gateway server
- **Home** `/home/manny` = your personal folder on the gateway
- You can see `/`, but you can't modify everything — only admins can

#### In HDFS (Cluster storage):
- **Root** `/` = the top of the Hadoop file system (entire cluster's namespace)
- **Home** `/user/manny` = your personal space in HDFS
- You can do anything inside `/user/manny`, but not in system-level directories like `/apps` or `/tmp` unless given permissions

👉 Both Linux and HDFS mirror the same design:
- **Root** = system-wide top
- **Home** = your personal workspace

### 5. Putting It Together in Flow
1. Engineer logs into gateway node
2. Lands in `/home/manny` (local Linux)
3. Engineer runs a Hadoop command
4. `hadoop fs -ls /` → Lists root of HDFS cluster (not local root)
5. NameNode receives the request
6. Looks up metadata: "This file lives on DN5, DN11, DN42."
7. If content is needed, DataNodes send it back through the gateway
8. Engineer only has write rights inside `/user/manny` in HDFS, not in `/` root

### 🔑 Final Analogy
Think of it like a giant university library:
- **HDFS root** `/` = the entire library building
- **HDFS** `/user/manny` = your personal study desk in that library
- **Gateway Node** = the library's reception → you sign in here, not directly at the bookshelves
- **Linux root** `/` = the reception building's own structure
- **Linux** `/home/manny` = your locker at the reception

So both Linux & HDFS have "root" and "home," but they are different universes:
- One is the gateway machine's local filesystem
- The other is the distributed Hadoop filesystem

✅ **Now the circle is complete:**
Facebook → HDFS cluster → racks → NameNode/DataNodes → Gateway Node → Linux root/home → HDFS root/user

---

## 7 - HDFS Vs Cloud Data Lake

### 🔹 HDFS (Distributed File System)
- **Design:** Splits files into fixed-size blocks (e.g., 128 MB) and distributes them across DataNodes
- **Tight coupling:** HDFS storage is bound to the compute cluster (you can't separate storage from cluster size)
- If you want more storage, you must add more DataNodes → which means adding more CPU/RAM too (even if you don't need compute)
- **Cluster-specific:** Each HDFS cluster has its own namespace. Data from Cluster A can't be directly read by Cluster B
- **Metadata:** Maintained by NameNode
- **Best for:** On-prem setups, Hadoop-native workflows

### 🔹 Cloud Data Lake (Object Storage: ADLS, S3, GCS)
- **Design:** Stores files as objects, each object =
  - **ID** → unique identifier
  - **Value** → file content
  - **Metadata** → permissions, type, owner, etc.
- **Loose coupling:** Storage is independent of compute
- You can dump petabytes into S3/ADLS without spinning up Spark/Databricks until you actually need compute
- **Global access:** You don't "tie" data to a single cluster. Any Spark/Databricks/EMR cluster can access the same S3 bucket
- **Scalability:** Practically unlimited, auto-managed by cloud
- **Cost model:** Pay only for storage used + data transfer (and optionally compute when you spin it up)
- **Best for:** Modern architectures (lakehouses, ELT pipelines)

### ✅ Top 7 Major Differences: HDFS vs Cloud Data Lakes

| 🔸 Aspect | 🔹 HDFS | 🔹 Cloud Data Lakes (S3, ADLS) |
|-----------|---------|--------------------------------|
| **1. Storage Type** | Block-based storage | Object-based storage |
| **2. Coupling** | Tightly coupled with compute (storage comes with cluster) | Decoupled from compute (storage is independent) |
| **3. Accessibility** | Data belongs to one cluster only | Data is globally accessible (any cluster, notebook, job) |
| **4. Scalability** | Scale = Add nodes (adds compute too) | Auto-scalable and practically unlimited |
| **5. Cost Efficiency** | Expensive — Pay for both compute & storage, even if idle | Cheaper — Pay only for storage, compute is optional |
| **6. Metadata Handling** | Basic metadata via NameNode | Rich metadata (owner, tags, permissions, schema info) |
| **7. Ecosystem Support** | Designed for traditional Hadoop stack | Built for modern data stacks — Spark, Databricks, Synapse, Snowflake, etc |

### 🧠 Extra Clarifications:

#### Block vs Object:
- **Block** = break file into chunks → store on DataNodes → manage via NameNode
- **Object** = store entire file + metadata in a single unit in a bucket

#### Tight vs Loose coupling:
- **In HDFS,** storage lives inside the cluster
  - → If cluster goes down or deleted, storage goes too
- **In Cloud,** storage is separate
  - → You can stop compute and data still exists

#### Cross-access:
- **In HDFS,** you can't share storage across clusters
- **In Cloud,** your storage can be accessed from anywhere (multi-cluster, notebooks, APIs, dashboards)