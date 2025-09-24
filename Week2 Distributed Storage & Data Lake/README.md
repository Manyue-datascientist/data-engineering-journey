# Week 2 – Distributed Storage & Data Lake

This week focused on **HDFS (Hadoop Distributed File System)**, **Linux Commands**, and **Cloud Data Lakes**.  
We explored how distributed storage works, practiced Linux & HDFS commands, and compared HDFS with modern cloud object storage (S3, ADLS).

---

## 📚 Topics Covered
- HDFS Overview
- Linux Commands (`cd`, `ls`, file operations, permissions, etc.)
- HDFS Commands (upload/download, mkdir, rm, ls, cat, etc.)
- Fault Tolerance in HDFS (Replication, Secondary NN, Federation, Racks)
- HDFS vs Cloud Data Lakes (S3, ADLS, GCS)

---

## 🏗️ HDFS Architecture

```text
                +-------------------+
                |     NameNode      |
                | (stores metadata) |
                +-------------------+
                  /   |    |     \
                 /    |    |      \
        +--------+ +--------+ +--------+
        |DataNode| |DataNode| |DataNode|
        | Block  | | Block  | | Block  |
        +--------+ +--------+ +--------+
```

- **NameNode** = Index page (metadata, mappings of file → block → DataNode)
- **DataNode** = Actual storage of file blocks
- **Default Block Size** = 128 MB
- **Replication Factor** = 3 (fault tolerance)

👉 **Analogy**: Think of a 1000-page book:
- Index page = NameNode
- Actual pages = DataNodes

---

## ⚡ Key Concepts

### 1. Block Size
- **Small (64 MB)** → more parallelism, but more metadata (NameNode overhead)
- **Large (256 MB)** → less parallelism, but lighter NameNode load
- **Default 128 MB** is the balance

### 2. Fault Tolerance
- **Replication Factor** (default 3)
- **Rack Awareness** (copies spread across racks/datacenters)
- **Secondary NameNode** (checkpoints metadata, not a backup)
- **Federation** (multiple NameNodes for scalability)

### 3. Gateway Node
- Engineers log into gateway (Linux VM)
- **Local Linux home**: `/home/username`
- **HDFS home**: `/user/username`
- Commands like `hadoop fs -ls` interact with NameNode via gateway

---

## 💻 Linux Commands (Quick Reference)

| Command | Description | Example |
|---------|-------------|---------|
| `pwd` | Show current directory | `/home/manny` |
| `whoami` | Current user | `manny` |
| `cd /` | Go to root | |
| `cd ~` | Go to home | |
| `ls -l` | Long listing | perms, owner, size |
| `ls -a` | Show hidden files | `.bash_history` |
| `mkdir dir1` | Create directory | |
| `rm -R dir1` | Remove directory (non-empty) | |
| `chmod 777 file1` | Full access to all | |
| `cat file.txt` | View contents | |
| `vi file.txt` | Open in editor | |
| `du -h` | Disk usage in human-readable | |
| `grep "word" file.txt` | Search text | |

---

## 💻 HDFS Commands (Quick Reference)

| Command | Description | Example |
|---------|-------------|---------|
| `hadoop fs -ls /` | List HDFS root | |
| `hadoop fs -mkdir /user/manny` | Create dir | |
| `hadoop fs -put file.txt /user/manny` | Upload file | |
| `hadoop fs -get /user/manny/file.txt .` | Download file | |
| `hadoop fs -cp /src /dest` | Copy within HDFS | |
| `hadoop fs -mv /src /dest` | Move/rename | |
| `hadoop fs -cat file.txt` | Show file content | |
| `hadoop fs -tail file.txt` | Show last KB | |
| `hadoop fs -df -h` | Disk usage | |
| `hdfs fsck file.txt -files -blocks -locations` | Check file health | |

---

## ☁️ HDFS vs Cloud Data Lake

| Aspect | HDFS | Cloud Data Lake (S3, ADLS, GCS) |
|--------|------|----------------------------------|
| **Storage** | Block-based | Object-based |
| **Coupling** | Tightly tied to compute cluster | Decoupled, storage independent |
| **Accessibility** | Cluster-specific | Global, multi-cluster |
| **Scalability** | Add nodes (adds compute too) | Practically unlimited |
| **Cost** | Pay for storage + compute | Pay only for storage (compute optional) |
| **Metadata** | NameNode only | Rich metadata (owner, schema, tags) |
| **Ecosystem** | Hadoop-native | Modern data stack (Spark, Databricks, Synapse, Snowflake) |

---

## 🎯 Key Takeaways

- **HDFS** = block storage tightly coupled with compute
- **Cloud Data Lakes (S3/ADLS)** = object storage, decoupled, more scalable & cost-effective
- **Linux & HDFS commands** are foundational for working with distributed systems
- **Gateway node** = safe entry point for engineers to interact with clusters