# Week 2 Mini Project – HDFS & Linux Practice

This exercise simulates a **real-world data ingestion pipeline**:

1. A third-party drops raw data into a *landing* zone
2. Data is filtered/cleaned and moved to a *staging* zone  
3. Processed results are written back to HDFS and later pulled to local
4. File permissions, movement, and cleanup are also performed

---

## 🏗️ Architecture Flow

```text
(Local Gateway Linux)                        (HDFS Distributed Storage)
───────────────────────────────        ────────────────────────────────────
Landing (raw)   ──▶   Staging (filtered)   ──▶   HDFS /data/landing
                                                   │
                                                   ▼
                                          HDFS /data/staging
                                                   │
                                                   ▼
                                          Spark Job Output → HDFS /data/results
                                                   │
                                                   ▼
(Local Gateway Linux) ◀── final results pulled back from HDFS /data/results
```

---

## 📌 Step-by-Step Workflow

### 1. Gateway Setup
```bash
cd ~
mkdir landing
mkdir staging
```

### 2. Copy raw orders file into landing
```bash
cp /data/retail_db/orders/part-00000 landing/
```

### 3. Filter records for PENDING_PAYMENT
```bash
grep PENDING_PAYMENT ~/landing/part-00000 >> ~/staging/orders_filtered.csv
```

### 4. Create directory hierarchy in HDFS
```bash
hadoop fs -mkdir -p data/landing
```

### 5. Upload filtered file to HDFS
```bash
hadoop fs -put ~/staging/orders_filtered.csv data/landing/
```

### 6. Validate records count in HDFS
```bash
hadoop fs -cat data/landing/orders_filtered.csv | wc -l
```

### 7. List files in HDFS (with size)
```bash
hadoop fs -ls -h -S data/landing
```

### 8. Change file permissions
```bash
hadoop fs -chmod 764 data/landing/orders_filtered.csv
```

### 9. Move file to staging in HDFS
```bash
hadoop fs -mkdir -p data/staging
hadoop fs -mv data/landing/orders_filtered.csv data/staging/
```

### 10. Simulate Spark output in local
```bash
vi ~/orders_result.csv
# Insert 2 processed rows:
# 3617,2013-08-15 00:00:00.0,8889,PENDING_PAYMENT
# 68714,2013-09-06 00:00:00.0,8889,PENDING_PAYMENT
```

### 11. Upload Spark result to HDFS
```bash
hadoop fs -mkdir data/results
hadoop fs -put ~/orders_result.csv data/results/
```

### 12. Bring processed results back to local
```bash
mkdir -p ~/data/results
hadoop fs -get data/results/orders_result.csv ~/data/results/
```

### 13. Rename file locally
```bash
mv ~/data/results/orders_result.csv ~/data/results/final_results.csv
```

### 14. Cleanup
```bash
rm -R landing staging data/results orders_result.csv
hadoop fs -rm -R data/landing data/staging data/results
```

---

## 🎯 Key Learnings

- **Hands-on practice** of Linux commands for file management
- How to move data between **Local (Gateway) ↔ HDFS**
- Practical use of `grep`, `wc`, `chmod`, `ls` variations
- Simulated **end-to-end ingestion pipeline**: Landing → Staging → Results
- Reinforced concept of **gateway as entry point** into Hadoop cluster
