# ⚡ Performance Optimization Guide

Optimizing NiFi performance involves understanding bottlenecks, tuning configuration parameters, and implementing best practices for scaling.

## Performance Analysis

### Identifying Bottlenecks

**Common Bottleneck Types:**
```
Performance Bottlenecks:
├── CPU Bound
│   ├── Complex transformations
│   ├── Regular expressions
│   ├── Encryption/decryption
│   └── JSON/XML parsing
├── I/O Bound  
│   ├── File system operations
│   ├── Network communications
│   ├── Database queries
│   └── Remote API calls
├── Memory Bound
│   ├── Large FlowFile content
│   ├── Inefficient buffering
│   ├── Memory leaks
│   └── Garbage collection pressure
└── Concurrency Bound
    ├── Thread pool exhaustion
    ├── Lock contention
    ├── Resource competition
    └── Synchronization overhead
```

**Performance Profiling:**
```bash
# JVM profiling flags
-XX:+PrintGC
-XX:+PrintGCDetails  
-XX:+PrintGCTimeStamps
-XX:+UseG1GC
-XX:MaxGCPauseMillis=200

# Enable JFR (Java Flight Recorder)
-XX:+FlightRecorder
-XX:StartFlightRecording=duration=60s,filename=nifi-profile.jfr
```

### Metrics Collection and Analysis

**Key Performance Indicators:**
```bash
Throughput Metrics:
- FlowFiles processed per second
- Bytes processed per second  
- Records processed per second
- Transactions per second

Latency Metrics:
- Average processing time per FlowFile
- End-to-end flow latency
- Queue wait times
- Network round-trip times

Resource Metrics:
- CPU utilization per processor
- Memory allocation rates
- Disk I/O bandwidth utilization
- Network bandwidth utilization
```

## JVM and System Tuning

### Heap Configuration

**Memory Allocation:**
```bash
# nifi-env.sh or bootstrap.conf
export JAVA_OPTS="
    -Xms8g                          # Initial heap size
    -Xmx8g                          # Maximum heap size  
    -XX:+UseG1GC                    # G1 garbage collector
    -XX:MaxGCPauseMillis=200        # GC pause target
    -XX:G1HeapRegionSize=16m        # G1 region size
    -XX:+UseStringDeduplication     # String deduplication
    -XX:+UnlockExperimentalVMOptions
    -XX:+UseCGroupMemoryLimitForHeap # Container awareness
"
```

**Garbage Collection Tuning:**
```bash
# G1GC optimization for NiFi workloads
-XX:+UseG1GC
-XX:MaxGCPauseMillis=200
-XX:G1HeapRegionSize=16m
-XX:G1NewSizePercent=20
-XX:G1MaxNewSizePercent=30
-XX:G1MixedGCCountTarget=8
-XX:G1MixedGCLiveThresholdPercent=85
-XX:G1OldCSetRegionThresholdPercent=20

# GC logging  
-Xloggc:logs/nifi-gc.log
-XX:+PrintGCDetails
-XX:+PrintGCTimeStamps
-XX:+UseGCLogFileRotation
-XX:NumberOfGCLogFiles=5
-XX:GCLogFileSize=100M
```

### Off-Heap Memory Configuration

**Direct Memory Allocation:**
```bash
# Increase direct memory for large content processing
-XX:MaxDirectMemorySize=4g

# Content repository configuration
nifi.content.claim.max.appendable.size=50 MB
nifi.content.claim.max.flow.files=1000
```

**Repository Configuration:**
```properties
# FlowFile Repository (metadata)
nifi.flowfile.repository.implementation=org.apache.nifi.controller.repository.WriteAheadFlowFileRepository
nifi.flowfile.repository.wal.implementation=org.apache.nifi.wali.SequentialAccessWriteAheadLog
nifi.flowfile.repository.directory=./flowfile_repository
nifi.flowfile.repository.checkpoint.interval=20 secs
nifi.flowfile.repository.always.sync=false

# Content Repository (actual data)
nifi.content.repository.implementation=org.apache.nifi.controller.repository.FileSystemRepository
nifi.content.repository.directory.default=./content_repository
nifi.content.repository.archive.max.retention.period=7 days
nifi.content.repository.archive.max.usage.percentage=50%
```

## Processor-Level Optimization

### Concurrent Task Configuration

**Thread Pool Sizing:**
```bash
Sizing Guidelines:
├── CPU-intensive processors: Number of CPU cores
├── I/O-intensive processors: 2-4x CPU cores  
├── Network-bound processors: Based on connection limits
└── Database processors: Match connection pool size

Example Configurations:
- GetFile: 1-2 concurrent tasks
- PutFile: 1-4 concurrent tasks  
- InvokeHTTP: 10-20 concurrent tasks
- ExecuteSQL: Match DB pool size (typically 8-16)
```

**Scheduling Configuration:**
```bash
# Timer Driven (most common)
Run Schedule: 0 sec (continuous) to 60 sec (batch)
Run Duration: 25 millis to 1 sec

# CRON Driven (time-based)
Run Schedule: 0 0 2 * * ? (daily at 2 AM)

# Event Driven (experimental)
# Automatically triggered by upstream events
```

### Batching Strategies

**Batch Size Optimization:**
```bash
# File-based processors
Batch Size: 10-100 files per iteration
Consider: File size distribution, processing complexity

# Record-based processors  
Records per FlowFile: 1000-10000 records
Consider: Record size, transformation complexity

# Database operations
Batch Size: 100-1000 records per transaction
Consider: Transaction size limits, rollback implications
```

**MergeContent Configuration:**
```bash
# Optimize for downstream processing
Merge Strategy: Bin-Packing Algorithm
Minimum Entries: 10
Maximum Entries: 100  
Minimum Group Size: 1 MB
Maximum Group Size: 100 MB
Max Bin Age: 30 sec

# Header/Footer handling
Header File: ${fragment.identifier:equals('1'):ifElse('headers.txt', '')}
Footer File: ${fragment.identifier:equals(${fragment.count}):ifElse('footers.txt', '')}
```

### Connection Optimization

**Queue Configuration:**
```bash
# Back pressure settings
Back Pressure Object Threshold: 10,000 FlowFiles
Back Pressure Data Size Threshold: 1 GB

# FlowFile expiration
FlowFile Expiration: 24 hours (prevent infinite accumulation)

# Prioritization
Priority: FirstInFirstOut (FIFO) - default
         NewestFlowFileFirst - for real-time processing
         OldestFlowFileFirst - for batch processing
         PriorityAttributePrioritizer - custom priority
```

**Load Distribution:**
```bash
# Cluster load balancing
Load Balance Strategy: 
├── DO_NOT_LOAD_BALANCE (default)
├── PARTITION_BY_ATTRIBUTE (consistent hashing)
├── ROUND_ROBIN (even distribution)
└── SINGLE_NODE (route to one node)

# Partition by attribute example
Partition Attribute: customer_id
# Ensures all data for same customer goes to same node
```

## Scaling Strategies

### Horizontal Scaling (Clustering)

**Cluster Configuration:**
```properties
# nifi.properties for each node
nifi.cluster.is.node=true
nifi.cluster.node.address=node1.company.com
nifi.cluster.node.protocol.port=11443
nifi.cluster.node.protocol.max.threads=50
nifi.cluster.node.event.history.size=25
nifi.cluster.node.connection.timeout=5 sec
nifi.cluster.node.read.timeout=5 sec
nifi.cluster.firewall.file=
nifi.cluster.flow.election.max.wait.time=5 mins
nifi.cluster.flow.election.max.candidates=

# ZooKeeper configuration
nifi.zookeeper.connect.string=zk1:2181,zk2:2181,zk3:2181
nifi.zookeeper.connect.timeout=10 secs
nifi.zookeeper.session.timeout=10 secs
nifi.zookeeper.root.node=/nifi
```

**Load Balancing Strategies:**
```bash
# Strategy Selection Guide
Use ROUND_ROBIN when:
- Even processing load distribution needed
- Processors are stateless
- No data affinity requirements

Use PARTITION_BY_ATTRIBUTE when:  
- Related data should stay together
- Stateful processing required
- Order preservation needed within partitions

Use SINGLE_NODE when:
- Sequential processing required
- External system has connection limits
- Order preservation across all data needed
```

### Vertical Scaling

**Resource Allocation:**
```bash
# Memory scaling
Heap Size: 25-50% of available RAM
Direct Memory: 10-25% of available RAM
OS Cache: 25-50% of available RAM

# CPU scaling  
NiFi Threads: 2-4x CPU cores for mixed workloads
Repository threads: 2-4 threads per repository
Timer threads: 10% of total NiFi threads

# Storage scaling
Content Repository: High-speed SSD preferred
FlowFile Repository: SSD for metadata operations  
Provenance Repository: Can use slower storage
Archive: Tape or object storage for long-term retention
```

### Performance Testing

**Load Testing Framework:**
```bash
# Synthetic data generation
GenerateFlowFile Configuration:
- File Size: Match production data size distribution
- Batch Size: 100-1000 FlowFiles per iteration
- Custom Text: Representative of actual data format

# Performance measurement
1. Baseline single-node performance
2. Scale to target cluster size  
3. Measure linear scaling efficiency
4. Identify breaking points
5. Validate error handling under load
```

**Performance Benchmarking:**
```bash
# Example test scenarios
Scenario 1: High-throughput file processing
- 10,000 files/hour, 1MB average size
- Measure: Throughput, latency, resource usage

Scenario 2: Real-time stream processing  
- 1,000 events/second, 1KB average size
- Measure: End-to-end latency, backpressure behavior

Scenario 3: Large file processing
- 100 files/hour, 100MB average size  
- Measure: Memory usage, GC behavior, disk I/O

Scenario 4: Database integration
- 10,000 database records/minute
- Measure: Connection pool efficiency, transaction throughput
```

## Best Practices Summary

### Configuration Best Practices
```bash
1. Size JVM heap to 25-50% of available RAM
2. Use G1GC for better pause time characteristics
3. Configure repository directories on fast storage
4. Set appropriate back pressure thresholds
5. Use connection load balancing in clusters
6. Monitor and tune concurrent task counts
7. Implement proper error handling and retry logic
8. Use batch processing for high-volume data
9. Optimize processor scheduling intervals
10. Regular performance monitoring and tuning
```

### Operational Best Practices
```bash
1. Establish performance baselines
2. Monitor key performance indicators
3. Implement automated alerting
4. Regular capacity planning reviews
5. Performance testing before production changes
6. Document configuration decisions
7. Use version control for flow configurations
8. Implement proper backup and recovery procedures
9. Regular security reviews and updates
10. Continuous optimization based on metrics
```
