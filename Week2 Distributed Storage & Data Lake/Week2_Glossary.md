# Week 2 Distributed Storage & Data Lake Glossary

*A comprehensive reference covering HDFS architecture, Linux/HDFS commands, and cloud data lake comparisons*

---

# WEEK 2 GLOSSARY

---

## 🏗️ HDFS Architecture Deep Dive

**Master-Slave Architecture**
- **NameNode (Master)** → Stores metadata mapping (file → blocks → DataNodes)
- **DataNodes (Slaves)** → Store actual data blocks

**File-to-Block Example**
```text
file1.txt (500 MB) splits into:
- Block B1: 128 MB → stored on DataNode1
- Block B2: 128 MB → stored on DataNode2  
- Block B3: 128 MB → stored on DataNode3
- Block B4: 116 MB → stored on DataNode4
```

**Node = Physical/Virtual Machine**
- DataNode = server running DataNode software
- 4000-node cluster = 4000 physical machines

---

## 🔧 Block Size Optimization

**Default: 128 MB** → Sweet spot between parallelism and metadata overhead

**Smaller Blocks (64 MB)**
- ✅ More parallelism (more tasks can run simultaneously)
- ❌ More metadata entries (NameNode burden increases)
- Example: 1 GB file = 16 blocks = 16 metadata entries

**Larger Blocks (256 MB)**  
- ✅ Less NameNode overhead (fewer metadata entries)
- ❌ Less parallelism (fewer concurrent tasks)
- Example: 1 GB file = 4 blocks = 4 metadata entries

---

## 🛡️ Fault Tolerance Mechanisms

**Replication Factor** → Default 3 copies of each block

**Rack Awareness** → Replicas spread across different racks/datacenters

**Heartbeats** → DataNodes send "alive" signals every 3 seconds

**Self-Healing** → Automatic recreation of lost blocks when nodes fail

**NameNode High Availability**
- **Hadoop 1.x** → Single Point of Failure
- **Secondary NameNode** → Checkpoints metadata (NOT a backup!)
- **Hadoop 2.x+** → Active + Standby NameNodes with automatic failover

**NameNode Federation** → Multiple NameNodes for scalability (sales data → NN1, finance data → NN2)

---

## 🌐 Gateway Node Concept

**Gateway Node** → Reception desk to the Hadoop warehouse

**Purpose** → Safe entry point for engineers (prevents direct NameNode/DataNode access)

**Architecture Flow**
```text
Engineer → Gateway Node → NameNode → DataNodes
         (Linux VM)     (Metadata)   (Actual Data)
```

**File System Separation**
- **Local Linux Home** → `/home/username` (on gateway machine)
- **HDFS Home** → `/user/username` (in distributed cluster)

---

## 🐧 Essential Linux Commands

### Navigation & Listing
```bash
pwd                     # Present working directory
whoami                  # Current user
cd / cd ~ cd .. cd -   # Change directories (root/home/up/previous)
ls -l -t -r -a -R      # List variations (long/time/reverse/all/recursive)
```

### File & Directory Operations
```bash
touch file1            # Create empty file
mkdir dir1             # Create directory
mkdir -p path/to/dir   # Create nested directories
cp file1 file2         # Copy file
cp -R dir1 dir2        # Copy directory recursively
mv file1 file2         # Move/rename
rm file1               # Remove file
rm -R dir1             # Remove directory recursively
rmdir dir1             # Remove empty directory
```

### File Content & Analysis
```bash
cat file1              # View entire file
head file1             # First 10 lines
tail file1             # Last 10 lines
wc file1               # Word/line/character count
grep "word" file1      # Search within file
du -h                  # Disk usage human-readable
```

### Permissions
```bash
chmod 777 file1        # Full access to all (rwx = 4+2+1)
# Format: Owner|Group|Others
# r(read)=4, w(write)=2, x(execute)=1
```

### File Creation & Editing
```bash
cat > file.txt         # Create file with inline input
vi filename            # Vim editor (i=insert, Esc+:wq=save)
```

---

## 🗄️ HDFS Commands Mastery

### Basic HDFS Operations
```bash
hadoop fs              # List all HDFS commands
hdfs dfs               # Same as hadoop fs (newer alias)
hadoop fs -ls /        # List HDFS root (NOT local root)
hadoop fs -ls /user/username  # List HDFS home directory
```

### Directory Management
```bash
hadoop fs -mkdir /path/dir         # Create directory
hadoop fs -mkdir -p /nested/path   # Create nested directories
hadoop fs -rmdir empty_dir         # Remove empty directory
hadoop fs -rm -R directory         # Remove non-empty directory
```

### File Transfer (Local ↔ HDFS)
```bash
# Upload: Local → HDFS
hadoop fs -put localfile.txt /hdfs/path/
hadoop fs -copyFromLocal localfile.txt /hdfs/path/

# Download: HDFS → Local  
hadoop fs -get /hdfs/file.txt /local/path/
hadoop fs -copyToLocal /hdfs/file.txt /local/path/

# Within HDFS
hadoop fs -cp /src/file /dest/file    # Copy
hadoop fs -mv /src/file /dest/file    # Move/rename
```

### File Viewing & Analysis
```bash
hadoop fs -cat /path/file.txt       # View file content
hadoop fs -tail /path/file.txt      # Last 1KB of file
hadoop fs -ls -s -h /path          # List with sizes (human-readable)
hadoop fs -ls -t /path             # Sort by time (newest first)
hadoop fs -ls -t -r /path          # Sort by time (oldest first)
hadoop fs -ls -R /path             # Recursive listing
```

### Health & Space Monitoring
```bash
hadoop fs -df -h                            # HDFS disk usage
hdfs fsck /path/file -files -blocks -locations  # File health check
```

---

## 📊 HDFS vs Cloud Data Lakes

### Storage Architecture
| Aspect | HDFS | Cloud Data Lakes |
|--------|------|------------------|
| **Storage Type** | Block-based (splits files into 128MB blocks) | Object-based (stores entire files as objects) |
| **Object Structure** | N/A | ID + Value + Metadata |

### Coupling & Scalability  
| Aspect | HDFS | Cloud Data Lakes |
|--------|------|------------------|
| **Coupling** | Tightly coupled (storage bound to compute cluster) | Loosely coupled (storage independent of compute) |
| **Scaling** | Add DataNodes = Add storage + CPU + RAM | Auto-scaling storage, on-demand compute |
| **Access Pattern** | Single cluster namespace | Global access across multiple clusters/services |

### Cost & Operational Model
| Aspect | HDFS | Cloud Data Lakes |
|--------|------|------------------|
| **Cost Model** | Pay for storage + compute always (even when idle) | Pay for storage + compute only when used |
| **Metadata** | Basic metadata via NameNode | Rich metadata (owner, tags, permissions, schema) |
| **Best For** | On-prem, Hadoop-native workflows | Modern data stacks (Spark, Databricks, Synapse, Snowflake) |

---

## 🔄 Real-World Pipeline Example

### Data Flow Architecture
```text
Landing Zone → Staging Zone → Processing → Results → Consumption
(raw data)   (filtered)     (Spark)     (output)   (local/apps)
```

### Command Sequence
```bash
# 1. Setup local directories
mkdir landing staging

# 2. Get raw data
cp /data/retail_db/orders/part-00000 landing/

# 3. Filter data  
grep PENDING_PAYMENT ~/landing/part-00000 >> ~/staging/orders_filtered.csv

# 4. Upload to HDFS
hadoop fs -mkdir -p data/landing
hadoop fs -put ~/staging/orders_filtered.csv data/landing/

# 5. Process and move in HDFS
hadoop fs -mkdir data/staging data/results
hadoop fs -mv data/landing/orders_filtered.csv data/staging/

# 6. Download results
hadoop fs -get data/results/output.csv ~/final_results/

# 7. Cleanup
rm -R landing staging
hadoop fs -rm -R data/landing data/staging data/results
```

---

## 🎯 Command Categories for Quick Reference

### Most Critical Linux Commands
```bash
ls -ltr    cd ~    mkdir -p    cp -R    mv    rm -R
cat    head    tail    grep    chmod    pwd    du -h
```

### Most Critical HDFS Commands
```bash
hadoop fs -ls    -mkdir -p    -put    -get    -cat
hadoop fs -mv    -cp    -rm -R    -df -h
hdfs fsck
```

### File System Navigation
- **Local**: `/` (Linux root), `/home/user` (Linux home)
- **HDFS**: `/` (HDFS root), `/user/username` (HDFS home)

---

## 🧠 Key Conceptual Models

**University Library Analogy**
- **HDFS root** `/` = Entire library building
- **HDFS** `/user/manny` = Your personal study desk
- **Gateway Node** = Library reception (sign in here)
- **Linux root** `/` = Reception building structure  
- **Linux** `/home/manny` = Your locker at reception

**Block Distribution Strategy**
- **File splitting** → Enable parallelism
- **Replication** → Ensure fault tolerance  
- **Rack awareness** → Survive datacenter failures
- **Metadata centralization** → Enable fast lookups

---

## 📈 Fault Tolerance Layers

1. **Block Level** → 3 replicas per block
2. **Rack Level** → Replicas across different racks
3. **Datacenter Level** → Geographic distribution
4. **NameNode Level** → Active/Standby with automatic failover
5. **Application Level** → Self-healing and re-replication

---

## 🔑 Week 2 Success Metrics

After mastering Week 2, you should be able to:
- ✅ Navigate both Linux and HDFS file systems confidently
- ✅ Transfer files between local and distributed storage
- ✅ Understand HDFS fault tolerance mechanisms  
- ✅ Compare block-based vs object-based storage architectures
- ✅ Execute complete data pipeline workflows
- ✅ Choose between HDFS and cloud storage based on requirements
- ✅ Debug file system issues using health check commands

---

## 🚀 Modern Context

**Why Learn HDFS?** → Foundation for understanding distributed storage principles

**Current Reality** → Most new projects use cloud data lakes (S3, ADLS, GCS)

**Career Value** → Many enterprises still run hybrid environments requiring HDFS knowledge

**Conceptual Bridge** → HDFS concepts directly apply to modern distributed systems

---

*This glossary provides comprehensive coverage of distributed storage concepts with practical command references for immediate application*