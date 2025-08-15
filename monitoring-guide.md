# 📊 Monitoring and Troubleshooting Guide

Effective monitoring and troubleshooting are essential for maintaining healthy NiFi data flows and ensuring optimal performance.

## System Monitoring

### Key Performance Metrics

**Processor Metrics:**
```
Performance Indicators:
├── Throughput (FlowFiles/sec)
├── Processing Time (average, max)
├── CPU Utilization (%)
├── Memory Usage (MB)
├── Queue Depth (FlowFiles in queue)
├── Back Pressure Events
├── Error Rate (failures/total)
└── Active Threads
```

**System-Level Metrics:**
```
Resource Utilization:
├── JVM Heap Usage (current/max)
├── JVM Non-Heap Usage  
├── CPU Usage (system-wide)
├── Disk I/O (read/write rates)
├── Network I/O (bytes/sec)
├── Content Repository Size
├── FlowFile Repository Size
└── Provenance Repository Size
```

### Built-in Monitoring Tools

**NiFi Summary Page:**
- Access via hamburger menu → Summary
- Real-time view of all components
- Sortable by various metrics
- Drill-down capabilities

**Status History:**
```bash
# View processor status history
Right-click processor → View status history

Metrics Available:
- FlowFiles In/Out
- Bytes In/Out  
- Processing Time
- CPU Usage
- Read/Write rates
```

**Data Provenance:**
```bash
# Track FlowFile lineage
Main menu → Data Provenance

Search Capabilities:
- FlowFile UUID
- Component ID
- Event Type (CREATE, RECEIVE, SEND, etc.)
- Time Range
- Attribute values
```

### External Monitoring Integration

**Prometheus Integration:**
```xml
<!-- Add to nifi.properties -->
nifi.analytics.predict.enabled=true
nifi.analytics.predict.interval=3 mins
nifi.analytics.query.interval=5 mins
nifi.analytics.connection.model.implementation=org.apache.nifi.controller.repository.analytics.ConnectionStatusAnalytics
```

**Custom Metrics Reporting:**
```java
// Custom reporting task example
@ReportingTask("CustomMetricsReporter")
public class CustomMetricsReporter extends AbstractReportingTask {
    
    @Override
    public void onTrigger(ReportingContext context) {
        ProcessGroupStatus rootGroup = context.getEventAccess().getControllerStatus();
        
        // Extract metrics
        int activeThreads = rootGroup.getActiveThreadCount();
        long queuedFlowFiles = rootGroup.getQueuedCount();
        
        // Send to external system (e.g., Grafana, CloudWatch)
        sendMetrics(activeThreads, queuedFlowFiles);
    }
}
```

## Flow Debugging

### Debugging Techniques

**LogAttribute Processor:**
```bash
# Essential debugging processor
Log Level: INFO
Attributes to Log: All Attributes
Attributes to Ignore: (leave empty)
Log Payload: false (for security)

# Sample output:
Standard FlowFile Attributes
Key: 'entryDate'
    Value: 'Tue Jan 15 10:30:45 EST 2024'
Key: 'lineageStartDate'  
    Value: 'Tue Jan 15 10:30:45 EST 2024'
Key: 'fileSize'
    Value: '1024'
```

**LogMessage Processor:**
```bash
# Custom logging with expression language
Log Level: WARN
Log Message: Processing file ${filename} with size ${fileSize} bytes

# Conditional logging
Log Message: ${fileSize:gt(10485760):ifElse('Large file detected: ${filename}', '')}
```

### Common Debugging Scenarios

**Scenario 1: FlowFiles Stuck in Queue**
```bash
Diagnosis Steps:
1. Check queue size and back pressure
2. Examine downstream processor status
3. Review processor scheduling settings
4. Check for processor validation errors

Common Causes:
- Back pressure thresholds reached
- Downstream processor stopped/invalid
- Resource constraints (CPU, memory)
- Blocking operations (network, disk)
```

**Scenario 2: Performance Degradation**
```bash
Investigation Process:
1. Compare current vs baseline metrics
2. Identify bottleneck processors
3. Check resource utilization
4. Review recent configuration changes

Performance Optimization:
- Increase concurrent threads
- Adjust batch sizes
- Optimize processor scheduling
- Scale horizontally (clustering)
```

**Scenario 3: Data Quality Issues**
```bash
Data Validation Approach:
1. Add data validation processors
2. Log sample data at key points
3. Track attribute changes
4. Monitor error relationships

Validation Processors:
- ValidateRecord
- ValidateXml
- EvaluateJsonPath (for validation)
- RouteOnAttribute (for filtering)
```

## Error Handling Strategies

### Error Relationship Management

**Standard Error Relationships:**
```bash
Common Relationships:
├── success → Normal processing path
├── failure → Permanent errors (don't retry)
├── retry → Temporary errors (should retry)
├── original → Original FlowFile (for splitting scenarios)
├── matched/unmatched → Conditional routing results
└── invalid → Data validation failures
```

**Error Routing Patterns:**
```bash
Pattern 1: Retry with Backoff
[Processor] → failure → [UpdateAttribute] → [PenalizeFlowFile] → [Retry Queue]

Pattern 2: Dead Letter Queue
[Processor] → failure → [UpdateAttribute] → [PutFile (errors)]

Pattern 3: Circuit Breaker
[Processor] → failure → [RouteOnAttribute] → [MonitorActivity] → [Alternative Path]
```

### RetryFlowFile Processor
```bash
# Configure retry logic
Retry Attribute: retry_count
Maximum Retries: 3
Penalize on Failure: true
Requeue Penalty: 30 sec

# Implementation
retry_count: ${retry_count:toNumber():plus(1)}
max_retries_reached: ${retry_count:toNumber():gt(3)}
```

## Alerting and Notifications

### Built-in Alerting

**Bulletin Board Monitoring:**
```bash
# Access via Main menu → Bulletin Board
Alert Types:
- ERROR: Critical issues requiring attention
- WARN: Potential problems
- INFO: General information

# Bulletin retention
Default: 5 minutes
Configurable via: nifi.ui.banner.text.retention
```

### External Alerting Integration

**Email Notifications:**
```xml
<!-- reporting-tasks.xml example -->
<reportingTask>
    <id>email-notification</id>
    <class>org.apache.nifi.reporting.email.EmailNotificationReportingTask</class>
    <schedulingPeriod>1 min</schedulingPeriod>
    <schedulingStrategy>TIMER_DRIVEN</schedulingStrategy>
    <properties>
        <property name="SMTP Server">smtp.company.com</property>
        <property name="SMTP Port">587</property>
        <property name="From">nifi-alerts@company.com</property>
        <property name="To">ops-team@company.com</property>
        <property name="Subject">NiFi Alert: ${alert.type}</property>
    </properties>
</reportingTask>
```

**Webhook Integration:**
```bash
# Custom reporting task for webhook notifications
POST /webhook/nifi-alerts
Content-Type: application/json

{
    "timestamp": "2024-01-15T10:30:00Z",
    "severity": "ERROR",
    "component": "DatabaseProcessor",
    "message": "Database connection failed after 3 attempts",
    "metrics": {
        "failed_flowfiles": 150,
        "queue_depth": 500
    }
}
```

### Monitoring Best Practices

**Proactive Monitoring:**
```bash
Key Thresholds:
├── Queue Depth > 1000 FlowFiles (WARNING)
├── Queue Depth > 5000 FlowFiles (CRITICAL) 
├── CPU Usage > 80% for 5 minutes (WARNING)
├── Memory Usage > 85% (WARNING)
├── Disk Usage > 90% (CRITICAL)
├── Error Rate > 5% (WARNING)
└── Processing Time > 2x baseline (WARNING)
```

**Monitoring Automation:**
```bash
# Example monitoring script
#!/bin/bash

NIFI_API="https://nifi.company.com:8443/nifi-api"
ALERT_THRESHOLD=1000

# Check queue depths
QUEUE_COUNT=$(curl -s -k "$NIFI_API/flow/status" | jq '.controllerStatus.queued')

if [ "$QUEUE_COUNT" -gt "$ALERT_THRESHOLD" ]; then
    echo "ALERT: Queue depth exceeded threshold: $QUEUE_COUNT"
    # Send notification
    curl -X POST webhook.company.com/alerts \
        -H "Content-Type: application/json" \
        -d "{\"message\": \"NiFi queue depth: $QUEUE_COUNT\"}"
fi
```
