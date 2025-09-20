# Week 1 Summary: Big Data – The Big Picture

---

## 🌐 Why Big Data?

Traditional databases hit limits with:
- **Volume** (too much data),
- **Variety** (structured + unstructured),
- **Velocity** (real-time streams),
- **Veracity** (quality issues),
- **Value** (extracting meaning).

They scaled **vertically** (bigger machines), but that broke at scale. So, the industry shifted to **distributed systems**: multiple small machines sharing the load.

---

## 🐘 Hadoop: The Pioneer

Hadoop introduced:
- **HDFS** → Distributed storage.
- **MapReduce** → Parallel batch processing.
- **YARN** → Resource management.

💡 *Lesson*: Distributed processing is possible and scalable — but Hadoop was complex and slow.

---

## ☁️ Cloud: The Next Gear

Cloud computing solved infrastructure bottlenecks:
- **Elastic scaling**
- **Pay-as-you-go**
- **Geo-distribution**
- **Disaster recovery**

It replaced on-prem Hadoop clusters with faster, cheaper, flexible cloud services (AWS, Azure, GCP).

---

## ⚡ Spark: Speed Upgrade

Spark fixed MapReduce’s slowness by keeping data **in memory**:
- 10x–100x faster.
- Batch + Streaming + ML.
- Became the go-to engine for Big Data.

---

## 🏗️ DB → DWH → Data Lake

| Layer        | Purpose              | Schema       | Data Type    |
|--------------|----------------------|--------------|--------------|
| **Database** | Real-time operations | On Write     | Structured   |
| **DWH**      | Analytics            | On Write     | Structured   |
| **Data Lake**| Store everything     | On Read      | All types    |

Modern stacks now merge DWH + Lakes (Delta Lake, Iceberg, etc.)

---

## 🧱 HDFS Architecture

- **NameNode** = Metadata.
- **DataNodes** = Store actual data blocks.
- **Replication** ensures fault tolerance.
- **Heartbeats** & **Self-healing** make it robust.

---

## 🔁 On-Prem vs Cloud Pipelines

**Then (Hadoop era)**:  
MySQL → Sqoop → HDFS → MapReduce → Hive

**Now (Cloud era)**:  
Azure: Sources → ADF → ADLS → Databricks → Synapse  
AWS: Sources → Glue → S3 → Databricks → Athena

---

## 🧠 Week 1 Mindset Shift

> "From monoliths to distributed, from storage to insight — Big Data is not just a toolset, it’s a mindset shift."

---

