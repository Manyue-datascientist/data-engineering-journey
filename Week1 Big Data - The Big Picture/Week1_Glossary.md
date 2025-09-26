# Week 1 Big Data Glossary - Quick Reference

*A comprehensive glossary following the natural learning flow from traditional systems to modern big data architectures*

---

# WEEK 1 GLOSSARY

---

## 🔥 The Big Data Problem

**5 Vs of Big Data**
- **Volume** → Massive amounts of data that broke traditional databases
- **Variety** → Mix of structured (tables) + unstructured (JSON, logs, images, videos)
- **Velocity** → Real-time data streams flooding in continuously
- **Veracity** → Data quality and trustworthiness challenges
- **Value** → The actual business insight extractable from raw data

**Vertical Scaling** → Adding more CPU/RAM/disk to single machine (like feeding steroids to one athlete)

**Horizontal Scaling** → Using many small machines working together (team of average players)

**Distributed Systems** → Multiple machines sharing workload instead of one super-machine

---

## 🐘 Hadoop Era - The Pioneer

**Hadoop** → First distributed system that said "use 100 small computers instead of 1 big one"

**HDFS (Hadoop Distributed File System)** → Stores huge files by splitting across multiple nodes

**MapReduce** → Parallel batch processing engine (slow, writes everything to disk)

**YARN** → Resource manager that assigns cluster resources to applications

**Hadoop Evolution**
- **2007** → Hadoop 1.0 (HDFS + MapReduce)
- **2012** → Hadoop 2.0 (Added YARN)
- **Now** → Hadoop 3.x (More scalable, container support)

**Hadoop Ecosystem (Legacy Tools)**
- **Sqoop** → Move data between databases and HDFS
- **Hive** → SQL queries on Hadoop data
- **Pig** → Scripting for data processing
- **Oozie** → Workflow scheduler
- **HBase** → NoSQL database on HDFS

**Key Lesson** → Hadoop proved distributed systems work, but was complex and slow

---

## ☁️ Cloud Revolution

**Cloud Computing** → Rent servers worldwide with credit card instead of buying hardware

**Why Cloud Wins**
- **Scalability** → Add/remove resources instantly
- **Agility** → Setup time: Weeks → Minutes
- **Geo-distribution** → Place servers close to users
- **Disaster Recovery** → Built-in redundancy
- **Cost** → CapEx (big upfront) → OpEx (pay-as-you-go)

**Cloud Types**
- **Private** → Own infrastructure (banks, high security)
- **Public** → AWS, Azure, GCP
- **Hybrid** → Mix of private + public

**Modern Cloud Replacements**
- Sqoop → ADF/Glue
- HBase → CosmosDB/DynamoDB
- Oozie → Airflow
- MapReduce → Spark

---

## ⚡ Spark Revolution

**Apache Spark** → Game-changer that keeps data in memory (like chef cooking continuously)

**MapReduce Problem** → Wrote everything to disk after each step (like cleaning kitchen after each cooking step)

**Spark Solution** → In-memory processing = 10-100x faster

**Spark Architecture**
- **Driver** → The brain that plans workflow
- **Cluster Manager** → Assigns resources (YARN, Kubernetes)
- **Executors** → Workers that execute tasks and hold data in memory

**Spark Capabilities** → Batch + Streaming + Machine Learning + Graph processing

**Key Point** → Spark replaced MapReduce, not entire Hadoop (still uses HDFS)

---

## 🏗️ Storage Evolution Story

**Database (OLTP)** → The cash counter
- **Purpose** → Live transactions (insert, update, lookup)
- **Schema** → On Write (define structure before data)
- **Data Type** → Structured only
- **Strength** → Fast transactions
- **Weakness** → Not good for historical analysis

**Data Warehouse (OLAP)** → The accountant's office
- **Purpose** → Historical analytics on years of data
- **Schema** → On Write (structured)
- **Data Type** → Structured only
- **Strength** → Optimized for aggregations
- **Weakness** → Rigid, expensive

**Data Lake** → The storage warehouse
- **Purpose** → Store everything raw (receipts, photos, JSON, logs)
- **Schema** → On Read (apply structure when analyzing)
- **Data Type** → All types (structured + unstructured)
- **Strength** → Flexible, cheap storage
- **Weakness** → Can become data swamp without governance

---

## 🔄 Architecture Evolution

**Traditional (On-Prem Hadoop)**
```text
MySQL → Sqoop → HDFS → MapReduce → Hive/HBase
```

**Modern Cloud (Azure)**
```text
Sources → ADF → ADLS → Databricks/Synapse → SQL/CosmosDB
```

**Modern Cloud (AWS)**
```text
Sources → Glue → S3 → Databricks/Athena/Redshift → RDS/DynamoDB
```

---

## 🧱 HDFS Deep Dive

**NameNode** → The metadata brain (knows where every block lives)

**DataNodes** → Store actual data blocks

**Block Size** → 128 MB default (balance between parallelism and metadata overhead)

**Replication Factor** → 3 copies of each block for fault tolerance

**Heartbeats** → DataNodes send "I'm alive" signals every 3 seconds

**Self-Healing** → If node dies, HDFS automatically recreates lost blocks

**High Availability**
- **Hadoop 1.x** → NameNode = Single Point of Failure
- **Hadoop 2.x+** → Active + Standby NameNodes with Zookeeper

**Secondary NameNode** → Creates metadata checkpoints (NOT a backup!)

---

## 👷 Data Engineer's Role

**Core Responsibilities**
- Build scalable, resilient data pipelines
- Understand entire data flow from source to consumption
- Choose right tools for specific problems
- Design fault-tolerant distributed systems

**Key Skills**
- Understand WHY each tool exists, not just WHAT it does
- Make architectural decisions based on scale, speed, cost
- Avoid treating tools as buzzwords - see them as problem solvers

---

## 🎯 Week 1 Mental Models

**Scaling Mindset** → Monolith (one big machine) → Distributed (many small machines)

**Storage Evolution** → Rigid structures → Flexible raw storage → Smart processing

**Processing Evolution** → Disk-based batch → In-memory real-time

**Infrastructure Evolution** → Buy hardware → Rent in cloud

**Tool Selection** → Understand the problem first, then pick the right tool

---

## 🔑 Key Takeaways

1. **Big Data broke traditional systems** due to volume, variety, velocity
2. **Hadoop pioneered distributed computing** but was slow and complex
3. **Cloud solved infrastructure problems** with elasticity and pay-per-use
4. **Spark revolutionized processing** with in-memory computing
5. **Storage evolved** from rigid databases to flexible data lakes
6. **Modern architectures** are cloud-native, faster, and simpler
7. **Data Engineers** must understand the entire ecosystem flow

---

## 🚀 Success Metrics

After mastering Week 1, you should be able to:
- ✅ Explain why traditional databases failed at scale
- ✅ Describe how distributed systems solve big data challenges
- ✅ Compare Hadoop vs Spark vs Cloud solutions
- ✅ Choose appropriate storage (DB vs DWH vs Data Lake) for use cases
- ✅ Understand the evolution story and make architectural decisions

---

*This glossary captures the complete Week 1 journey from problem identification to solution architecture*