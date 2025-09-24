# Week 2 Summary: Distributed Storage & Data Lake

## 🎯 Key Learning Objectives
- Understanding HDFS architecture and components
- Mastering essential Linux commands for file management
- Learning HDFS commands for distributed file operations
- Comparing HDFS vs Cloud Data Lakes

---

## 📊 HDFS Architecture Overview

### Core Components
- **NameNode (Master)**: Stores metadata mapping files to blocks and DataNodes
- **DataNode (Slaves)**: Store actual data blocks (default 128MB each)
- **Replication Factor**: 3 copies of each block across different DataNodes/racks

### Key Concepts
- **Block-based storage**: Files split into 128MB blocks for parallelism
- **Rack awareness**: Blocks distributed across different server racks for fault tolerance
- **Metadata management**: NameNode tracks all file-to-block mappings

---

## 🐧 Essential Linux Commands

### Navigation & Listing
```bash
pwd                    # Show current directory
cd ~                   # Go to home directory
cd ..                  # Go up one level
ls -ltr               # List files by time (oldest first)
ls -a                 # Show hidden files
```

### File Operations
```bash
touch file1           # Create empty file
mkdir dir1            # Create directory
cp file1 file2        # Copy file
mv file1 newname      # Move/rename file
rm -R directory       # Remove directory recursively
```

### File Content
```bash
cat file1             # View file content
head file1            # First 10 lines
tail file1            # Last 10 lines
grep "text" file1     # Search within file
wc file1              # Word/line/character count
```

### Permissions
```bash
chmod 777 file1       # Change permissions (rwx for all)
# Format: Owner|Group|Others (r=4, w=2, x=1)
```

---

## 🗄️ HDFS Commands

### Basic Operations
```bash
hadoop fs -ls /                    # List HDFS root
hadoop fs -mkdir -p /path/to/dir   # Create directories
hadoop fs -rm -R directory         # Remove directory
```

### File Transfer
```bash
# Local to HDFS
hadoop fs -put localfile /hdfs/path/
hadoop fs -copyFromLocal localfile /hdfs/path/

# HDFS to Local  
hadoop fs -get /hdfs/file localpath/
hadoop fs -copyToLocal /hdfs/file localpath/

# Within HDFS
hadoop fs -cp /src/file /dest/file
hadoop fs -mv /src/file /dest/file
```

### File Viewing & Analysis
```bash
hadoop fs -cat /path/file          # View file content
hadoop fs -tail /path/file         # Last 1KB of file
hadoop fs -ls -s -h /path          # List with human-readable sizes
hdfs fsck /path -files -blocks     # Check file health & blocks
```

---

## 🌐 Gateway Node Concept

### Architecture Flow
```
Engineer → Gateway Node → NameNode → DataNodes
         (Linux)        (Metadata)   (Actual Data)
```

### Key Points
- **Gateway**: Entry point to Hadoop cluster (Linux server with user accounts)
- **Local filesystem**: `/home/username` (on gateway machine)
- **HDFS filesystem**: `/user/username` (in distributed cluster)
- Engineers never directly access NameNode/DataNodes

---

## ☁️ HDFS vs Cloud Data Lakes

| Aspect | HDFS | Cloud Data Lakes |
|--------|------|------------------|
| **Storage Type** | Block-based (128MB chunks) | Object-based (entire files) |
| **Coupling** | Tightly coupled (storage + compute) | Loosely coupled (separate storage/compute) |
| **Scalability** | Add nodes = add storage + compute | Auto-scaling storage, on-demand compute |
| **Cost** | Pay for both storage & compute always | Pay for storage + compute only when used |
| **Access** | Single cluster access | Global access across multiple services |
| **Use Case** | Traditional Hadoop workloads | Modern data stacks (Spark, Databricks, etc.) |

---

## 🔧 Fault Tolerance Features

### HDFS Resilience
- **Replication Factor 3**: Each block stored on 3 different DataNodes
- **Rack Awareness**: Replicas distributed across different server racks
- **NameNode HA**: Active/Standby setup in Hadoop 2.x (eliminates single point of failure)
- **Secondary NameNode**: Checkpoints metadata (not a true backup)

---

## 📈 Scalability Solutions

### Horizontal Scaling
- **DataNodes**: Easy to add for more storage capacity
- **NameNode Federation**: Multiple NameNodes manage different namespaces
- **Block Size Optimization**: 128MB balances parallelism vs metadata overhead

---

## 💡 Real-World Application

### Data Pipeline Flow
1. **Landing Zone**: Raw data ingestion
2. **Staging Zone**: Data filtering and cleaning
3. **Processing**: Spark jobs transform data
4. **Results**: Final processed data stored back to HDFS
5. **Consumption**: Results pulled to local systems

### Commands in Practice
```bash
# Typical workflow
grep "PENDING_PAYMENT" rawdata.csv > filtered.csv
hadoop fs -put filtered.csv /data/staging/
hadoop fs -mv /data/staging/filtered.csv /data/processed/
hadoop fs -get /data/results/output.csv ~/final_results/
```

---

## 🎯 Key Takeaways for Data Engineers

1. **HDFS is foundation** for big data storage in Hadoop ecosystem
2. **Linux proficiency** essential for gateway node operations  
3. **Block-based design** enables massive scalability and fault tolerance
4. **Cloud alternatives** offer more flexibility for modern architectures
5. **Understanding both** systems crucial for hybrid environments
6. **Gateway nodes** provide secure, managed access to distributed clusters

---

## 📚 Commands Reference Card

### Most Used Linux Commands
```bash
ls -ltr    cd ~    mkdir    cp    mv    rm -R
cat    head    tail    grep    chmod    pwd
```

### Most Used HDFS Commands  
```bash
hadoop fs -ls    -mkdir -p    -put    -get
hadoop fs -cat    -mv    -cp    -rm -R
hdfs fsck    hadoop fs -df -h
```

This summary covers the essential concepts and practical skills needed for distributed storage management in big data environments.