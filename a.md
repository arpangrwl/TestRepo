# PATENT APPLICATION DOCUMENT

## SYSTEM AND METHOD FOR FAULT-TOLERANT DATA SYNCHRONIZATION FROM DISTRIBUTED FILE SYSTEMS TO STREAMING PLATFORMS WITHOUT DUPLICATION

---

## TECHNICAL FIELD

The present invention relates to distributed data processing systems, and more particularly, to a method and system for synchronizing data from Hadoop Distributed File System (HDFS) or Hive tables to Apache Kafka topics using Apache Spark with guaranteed exactly-once semantics, preventing both data loss and duplicate records.

---

## BACKGROUND OF THE INVENTION

### Problem Statement

In modern big data architectures, organizations frequently need to synchronize large volumes of data from batch storage systems (such as HDFS or Hive) to real-time streaming platforms (such as Apache Kafka). Current approaches face several critical challenges:

1. **Data Loss During Failures**: When an Apache Spark job fails mid-execution while writing to Kafka, there is no reliable mechanism to track which records were successfully written, leading to potential data loss.

2. **Duplicate Record Generation**: Upon job restart after failure, existing systems typically reprocess entire files, causing duplicate records in Kafka topics, which corrupts downstream analytics and processing.

3. **Large File Processing Limitations**: Processing large files in a single batch without intermediate checkpointing creates an "all-or-nothing" scenario where partial progress cannot be tracked or recovered.

4. **Partition-Level Tracking Complexity**: Kafka topics use multiple partitions for parallel processing, and tracking progress across all partitions during failures is not adequately addressed by existing solutions.

### Limitations of Existing Solutions

Existing approaches typically employ one of the following inadequate strategies:

- **Idempotent Producers**: Kafka's idempotent producer feature prevents duplicates within a single session but does not handle duplicates across job restarts.
- **Transactional Writes**: Kafka transactions provide atomicity but do not track file-level progress or handle job-level failures effectively.
- **External Offset Management**: Manual offset tracking systems exist but lack integration with source file processing state and do not handle hash-based deduplication.

---

## SUMMARY OF THE INVENTION

The present invention provides a novel system and method for synchronizing data from HDFS locations or Hive tables to Kafka topics using Apache Spark with exactly-once semantics, preventing both data loss and duplicate records through an innovative checkpoint-based tracking mechanism combined with hash-based record identification.

### Key Innovations

1. **Dual-State Checkpoint System**: A checkpoint mechanism that maintains both (a) Kafka topic partition offset details, and (b) file processing status with committed offsets.

2. **Hash-Based Record Identification**: Generation of deterministic hash values for each HDFS record, used as Kafka message keys to enable precise duplicate detection and delta computation.

3. **Atomic File Completion Tracking**: Files are marked as "completed" only after all records are successfully written to Kafka and offsets are committed to the checkpoint.

4. **Intelligent Recovery Mechanism**: Upon failure, the system performs left outer join between source file records and already-synced Kafka records (identified by hash keys) to compute and sync only delta records.

5. **Partition-Level Offset Verification**: Before recovery, the system verifies that Kafka partition offsets match checkpoint values, ensuring consistency before delta computation.

---

## DETAILED DESCRIPTION OF THE INVENTION

### System Architecture

The invention comprises the following components:

#### 1. Checkpoint Manager

A persistent storage mechanism that maintains:

```
Checkpoint Structure:
{
  "file_name": "hdfs://data/sales_2024_01_15.parquet",
  "status": "IN_PROGRESS" | "COMPLETED",
  "start_timestamp": "2024-01-15T10:30:00Z",
  "completion_timestamp": "2024-01-15T10:35:22Z",
  "kafka_topic": "sales_stream",
  "partition_offsets": {
    "0": 145230,
    "1": 145189,
    "2": 145267,
    "3": 145201
  },
  "total_records": 581887,
  "hash_algorithm": "SHA-256"
}
```

#### 2. Hash Generation Engine

For each record from HDFS/Hive, a deterministic hash is generated:

```
Hash_Key = HASH_FUNCTION(
  field_1 || field_2 || ... || field_n || source_file_name || record_position
)
```

The hash serves as the Kafka message key, ensuring:
- Deterministic partition assignment
- Unique record identification
- Duplicate detection capability

#### 3. Spark-Based Sync Processor

The Apache Spark job orchestrates the synchronization process with the following workflow:

**Step 1: Initialization**
- Check checkpoint for file processing status
- Verify Kafka partition offsets match checkpoint (if recovery mode)

**Step 2: Data Processing**
- Read records from HDFS/Hive
- Generate hash key for each record
- Create Kafka messages with hash as key and record as value

**Step 3: Kafka Production**
- Write records to Kafka topic in batches
- Track partition offsets for each batch

**Step 4: Checkpoint Update**
- Atomically update checkpoint with latest offsets
- Mark file as "COMPLETED" only after all records synced

#### 4. Recovery and Delta Computation Module

Upon failure recovery:

**Step 1: Validation**
```
For each partition P in Kafka topic:
  current_offset = getKafkaOffset(P)
  checkpoint_offset = getCheckpointOffset(P)
  
  if current_offset != checkpoint_offset:
    raise ConsistencyException
```

**Step 2: Delta Computation**
```
Already_Synced_Keys = readKafkaRecordKeys(topic, all_partitions)
Source_Records = readHDFSFile(file_name)

Delta_Records = Source_Records.leftOuterJoin(
  Already_Synced_Keys,
  on = "hash_key"
).where("Already_Synced_Keys.hash_key IS NULL")
```

**Step 3: Delta Synchronization**
- Sync only Delta_Records to Kafka
- Update checkpoint upon completion

---

## EXAMPLE CALCULATION AND WORKFLOW

### Scenario Setup

- **HDFS File**: `hdfs://warehouse/transactions/txn_2024_01_15.parquet`
- **File Size**: 10 GB
- **Record Count**: 50,000,000 records
- **Kafka Topic**: `transaction_stream`
- **Topic Partitions**: 4 (partitions 0, 1, 2, 3)
- **Hash Algorithm**: SHA-256

### Normal Execution Flow

**Initial State (Checkpoint)**:
```json
{
  "file_name": "hdfs://warehouse/transactions/txn_2024_01_15.parquet",
  "status": "NOT_STARTED",
  "partition_offsets": {
    "0": 1000000,
    "1": 1000050,
    "2": 999980,
    "3": 1000020
  }
}
```

**Processing Records**:

Record 1:
```
Source Data: {id: "TXN001", amount: 250.50, customer: "CUST_12345", timestamp: "2024-01-15T09:15:23"}
Hash Key: SHA-256("TXN001|250.50|CUST_12345|2024-01-15T09:15:23|txn_2024_01_15.parquet|0")
         = "a3f5e2c8d9b1..."
Kafka Partition: hash_key % 4 = 2
```

Record 2:
```
Source Data: {id: "TXN002", amount: 1150.00, customer: "CUST_67890", timestamp: "2024-01-15T09:16:45"}
Hash Key: SHA-256("TXN002|1150.00|CUST_67890|2024-01-15T09:16:45|txn_2024_01_15.parquet|1")
         = "b7c3a1f8e2d4..."
Kafka Partition: hash_key % 4 = 1
```

**Batch Processing** (Example: 1 million records per batch):

Batch 1 (Records 1 to 1,000,000):
- Partition 0: 250,125 records → offsets 1000000 to 1250125
- Partition 1: 250,050 records → offsets 1000050 to 1250100
- Partition 2: 249,980 records → offsets 999980 to 1249960
- Partition 3: 249,845 records → offsets 1000020 to 1249865

Updated Checkpoint after Batch 1:
```json
{
  "file_name": "hdfs://warehouse/transactions/txn_2024_01_15.parquet",
  "status": "IN_PROGRESS",
  "partition_offsets": {
    "0": 1250125,
    "1": 1250100,
    "2": 1249960,
    "3": 1249865
  },
  "records_processed": 1000000
}
```

### Failure Recovery Example

**Failure Scenario**:
- Job fails after processing 35,000,000 records (70% complete)
- Checkpoint shows:

```json
{
  "file_name": "hdfs://warehouse/transactions/txn_2024_01_15.parquet",
  "status": "IN_PROGRESS",
  "partition_offsets": {
    "0": 9750125,
    "1": 9750100,
    "2": 9749960,
    "3": 9749845
  },
  "records_processed": 35000000,
  "total_records": 50000000
}
```

**Recovery Process**:

**Step 1: Offset Verification**
```
Current Kafka Offsets:
- Partition 0: 9750125 ✓ (matches checkpoint)
- Partition 1: 9750100 ✓ (matches checkpoint)
- Partition 2: 9749960 ✓ (matches checkpoint)
- Partition 3: 9749845 ✓ (matches checkpoint)

Status: CONSISTENT - Proceed with recovery
```

**Step 2: Read Already-Synced Records**
```
Kafka_Keys = readKafkaKeys(
  topic = "transaction_stream",
  partitions = [0, 1, 2, 3],
  start_offsets = [1000000, 1000050, 999980, 1000020],
  end_offsets = [9750125, 9750100, 9749960, 9749845]
)

Kafka_Keys.count() = 35,000,000
```

**Step 3: Compute Delta**
```
HDFS_Records = readParquet("hdfs://warehouse/transactions/txn_2024_01_15.parquet")
HDFS_Records.count() = 50,000,000

HDFS_Records_With_Hash = HDFS_Records.withColumn(
  "hash_key",
  SHA256(concat_ws("|", col("id"), col("amount"), col("customer"), 
                   col("timestamp"), lit("txn_2024_01_15.parquet"), 
                   monotonically_increasing_id()))
)

Delta_Records = HDFS_Records_With_Hash
  .join(Kafka_Keys, Seq("hash_key"), "left_outer")
  .filter(col("kafka_key").isNull)
  .drop("kafka_key")

Delta_Records.count() = 15,000,000
```

**Step 4: Sync Delta**
```
Delta Distribution:
- Partition 0: 3,750,125 records → offsets 9750125 to 13500250
- Partition 1: 3,750,100 records → offsets 9750100 to 13500200
- Partition 2: 3,749,960 records → offsets 9749960 to 13499920
- Partition 3: 3,749,815 records → offsets 9749845 to 13499660

Total Delta: 15,000,000 records
```

**Step 5: Final Checkpoint Update**
```json
{
  "file_name": "hdfs://warehouse/transactions/txn_2024_01_15.parquet",
  "status": "COMPLETED",
  "partition_offsets": {
    "0": 13500250,
    "1": 13500200,
    "2": 13499920,
    "3": 13499660
  },
  "records_processed": 50000000,
  "total_records": 50000000,
  "completion_timestamp": "2024-01-15T11:42:15Z"
}
```

### Verification Calculation

**Total Records Synced**:
```
Partition 0: 13500250 - 1000000 = 12,500,250
Partition 1: 13500200 - 1000050 = 12,500,150
Partition 2: 13499920 - 999980  = 12,499,940
Partition 3: 13499660 - 1000020 = 12,499,640

Total: 50,000,000 records ✓
```

**Duplicate Verification**:
```
Distinct Hash Keys in Kafka = 50,000,000
Total Records in Kafka      = 50,000,000

Duplicates = 0 ✓
```

---

## CLAIMS

### Claim 1
A method for synchronizing data from a distributed file system to a streaming platform comprising:
- (a) generating a unique hash key for each record in a source file;
- (b) maintaining a checkpoint with file status and partition offset information;
- (c) writing records to a streaming topic using hash keys as message keys;
- (d) atomically updating the checkpoint upon successful batch writes;
- (e) marking the file as completed only after all records are synced;
- (f) upon failure, verifying partition offsets against checkpoint values;
- (g) computing delta records through left outer join of source records with already-synced records; and
- (h) syncing only delta records to prevent duplicates.

### Claim 2
The method of Claim 1, wherein the distributed file system is HDFS or Hive, and the streaming platform is Apache Kafka.

### Claim 3
The method of Claim 1, wherein the hash key is generated using a deterministic hash function comprising record fields, source file name, and record position.

### Claim 4
The method of Claim 1, wherein the checkpoint maintains a mapping of file names to processing status and partition-specific offset values.

### Claim 5
The method of Claim 1, wherein delta computation is performed only when partition offset verification confirms consistency between checkpoint and actual Kafka offsets.

### Claim 6
A system for fault-tolerant data synchronization comprising:
- (a) a checkpoint manager storing file processing states and partition offsets;
- (b) a hash generation engine creating deterministic identifiers for source records;
- (c) a distributed processing engine for parallel record synchronization;
- (d) a recovery module for offset verification and delta computation; and
- (e) a streaming platform writer with partition-aware offset tracking.

### Claim 7
The system of Claim 6, wherein the distributed processing engine is Apache Spark.

### Claim 8
The method of Claim 1, further comprising partition-level offset tracking where each partition's progress is independently monitored and recorded.

### Claim 9
The method of Claim 1, wherein the hash key serves as both a unique identifier for duplicate detection and a partition assignment key for the streaming platform.

### Claim 10
A non-transitory computer-readable medium containing instructions for performing the method of Claim 1.

---

## ADVANTAGES OF THE INVENTION

1. **Exactly-Once Semantics**: Guarantees no data loss and no duplicates across job failures
2. **Efficient Recovery**: Only processes delta records instead of reprocessing entire files
3. **Scalability**: Leverages Apache Spark's distributed processing capabilities
4. **Partition Awareness**: Tracks progress independently for each Kafka partition
5. **Deterministic Behavior**: Hash-based identification ensures consistent results across retries
6. **Atomic State Management**: Checkpoint updates are atomic, preventing partial state inconsistencies

---

## CONCLUSION

The present invention provides a comprehensive solution to the critical problem of reliable data synchronization from batch storage systems to streaming platforms, ensuring exactly-once delivery semantics through innovative checkpoint management and hash-based duplicate detection.

---

**Date**: December 22, 2025  
**Inventor**: [Your Name]  
**Application Type**: Provisional/Non-Provisional Patent Application

---

*This document is provided as a template for patent filing purposes. Please consult with a patent attorney to ensure compliance with specific jurisdictional requirements and to refine claims based on prior art analysis.*
