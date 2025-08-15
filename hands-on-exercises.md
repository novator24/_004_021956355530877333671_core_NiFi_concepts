# 🔧 Hands-on Exercises

## Exercise 1: Simple File Transfer

**Objective**: Create a flow that monitors a directory, processes files, and moves them to another location.

**Components Needed:**
- GetFile processor
- UpdateAttribute processor  
- PutFile processor

**Step-by-Step Instructions:**

1. **Create Input Directory Structure**
```bash
mkdir -p /tmp/nifi-demo/input
mkdir -p /tmp/nifi-demo/output
mkdir -p /tmp/nifi-demo/processed
```

2. **Build the Flow**
   - Add GetFile processor
   - Configure GetFile:
     ```
     Input Directory: /tmp/nifi-demo/input
     Keep Source File: false
     File Filter: .*\.txt
     ```
   
   - Add UpdateAttribute processor
   - Configure UpdateAttribute:
     ```
     processed_date: ${now():format('yyyy-MM-dd HH:mm:ss')}
     original_filename: ${filename}
     ```
   
   - Add PutFile processor
   - Configure PutFile:
     ```
     Directory: /tmp/nifi-demo/output
     Conflict Resolution: replace
     ```

3. **Create Connections**
   - GetFile → UpdateAttribute (success relationship)
   - UpdateAttribute → PutFile (success relationship)

4. **Test the Flow**
```bash
# Create test files
echo "Test file 1" > /tmp/nifi-demo/input/test1.txt
echo "Test file 2" > /tmp/nifi-demo/input/test2.txt
```

5. **Start Processors and Verify**
   - Start all processors
   - Check output directory for processed files
   - Verify attributes were added

## Exercise 2: Content Modification

**Objective**: Read files, modify their content, and save them with new names.

**Components Needed:**
- GetFile
- ReplaceText
- UpdateAttribute
- PutFile

**Flow Configuration:**

1. **GetFile Processor**
```
Input Directory: /tmp/nifi-demo/input
File Filter: .*\.log
Keep Source File: true
```

2. **ReplaceText Processor**
```
Search Value: ERROR
Replacement Value: WARNING
Replacement Strategy: Regex Replace
```

3. **UpdateAttribute Processor**
```
filename: ${filename:substringBeforeLast('.')}_processed.${filename:substringAfterLast('.')}
```

4. **PutFile Processor**
```
Directory: /tmp/nifi-demo/processed
```

**Test Data:**
```bash
cat > /tmp/nifi-demo/input/app.log << EOF
INFO: Application started
ERROR: Database connection failed
ERROR: Retry attempt 1
INFO: Connection established
ERROR: Authentication failed
EOF
```

## Exercise 3: Conditional Routing

**Objective**: Route files to different destinations based on their properties.

**Components Needed:**
- GetFile
- RouteOnAttribute
- UpdateAttribute (2 instances)
- PutFile (2 instances)

**Routing Logic:**
- Large files (>1KB) → `/tmp/nifi-demo/large`
- Small files (≤1KB) → `/tmp/nifi-demo/small`

**RouteOnAttribute Configuration:**
```
large_file: ${fileSize:gt(1024)}
small_file: ${fileSize:le(1024)}
```

**Create Test Files:**
```bash
# Small file
echo "Small file content" > /tmp/nifi-demo/input/small.txt

# Large file
head -c 2048 < /dev/zero > /tmp/nifi-demo/input/large.txt
```

## Exercise 4: Data Validation and Error Handling

**Objective**: Validate incoming data and handle errors gracefully.

**Components Needed:**
- GetFile
- ValidateRecord
- UpdateAttribute
- PutFile (for valid data)
- PutFile (for invalid data)

**Flow Design:**
1. Read CSV files
2. Validate against schema
3. Route valid records to success path
4. Route invalid records to error path
5. Add appropriate attributes for tracking

**Sample CSV Schema Validation:**
```json
{
  "type": "record",
  "name": "user",
  "fields": [
    {"name": "id", "type": "int"},
    {"name": "name", "type": "string"},
    {"name": "email", "type": "string"}
  ]
}
```

**Test CSV Data:**
```bash
cat > /tmp/nifi-demo/input/users.csv << EOF
id,name,email
1,John Doe,john@example.com
2,Jane Smith,jane@example.com
invalid,Bob Johnson,bob@invalid
3,Alice Brown,alice@example.com
EOF
```

## Exercise 5: Real-time Log Processing

**Objective**: Process streaming log data with parsing and alerting.

**Components Needed:**
- TailFile
- ExtractText
- RouteOnAttribute
- UpdateAttribute
- InvokeHTTP (for alerts)
- PutFile

**Log Format:**
```
2024-01-15 10:30:45 [INFO] user.service.UserController - User login successful
2024-01-15 10:30:46 [ERROR] db.ConnectionPool - Database connection timeout
```

**Flow Design:**

1. **TailFile Configuration**
```
Tailing Mode: Single file
File to Tail: /tmp/nifi-demo/app.log
Rolling Filename Pattern: /tmp/nifi-demo/app.log.*
```

2. **ExtractText Configuration**
```
timestamp: (\d{4}-\d{2}-\d{2} \d{2}:\d{2}:\d{2})
level: \[(.*?)\]
class: (\w+\.\w+\.\w+)
message: - (.*)
```

3. **RouteOnAttribute Configuration**
```
error_logs: ${level:equals('ERROR')}
warning_logs: ${level:equals('WARN')}
info_logs: ${level:equals('INFO')}
```

**Generate Test Logs:**
```bash
# Create continuous log generation script
cat > /tmp/generate_logs.sh << 'EOF'
#!/bin/bash
while true; do
    timestamp=$(date '+%Y-%m-%d %H:%M:%S')
    levels=("INFO" "WARN" "ERROR")
    level=${levels[$RANDOM % ${#levels[@]}]}
    echo "$timestamp [$level] app.service.TestService - Sample log message $RANDOM" >> /tmp/nifi-demo/app.log
    sleep 2
done
EOF

chmod +x /tmp/generate_logs.sh
# Run in background: ./tmp/generate_logs.sh &
```

## Exercise 6: Database Integration

**Objective**: Query database, process results, and update another table.

**Prerequisites:**
- PostgreSQL or MySQL database
- JDBC driver in NiFi lib directory

**Components Needed:**
- DBCPConnectionPool (Controller Service)
- ExecuteSQL
- ConvertRecord
- QueryRecord
- PutDatabaseRecord

**Setup Database:**
```sql
-- Create source table
CREATE TABLE employees (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100),
    department VARCHAR(50),
    salary DECIMAL(10,2),
    hire_date DATE
);

-- Insert sample data
INSERT INTO employees (name, department, salary, hire_date) VALUES
('John Doe', 'Engineering', 85000, '2023-01-15'),
('Jane Smith', 'Marketing', 65000, '2023-02-20'),
('Bob Johnson', 'Engineering', 95000, '2022-12-10'),
('Alice Brown', 'Sales', 70000, '2023-03-05');

-- Create summary table
CREATE TABLE department_summary (
    department VARCHAR(50) PRIMARY KEY,
    employee_count INTEGER,
    avg_salary DECIMAL(10,2),
    max_salary DECIMAL(10,2),
    last_updated TIMESTAMP
);
```

**Flow Configuration:**

1. **DBCPConnectionPool Setup**
```
Database Connection URL: jdbc:postgresql://localhost:5432/testdb
Database Driver Class Name: org.postgresql.Driver
Database User: nifi_user
Password: nifi_password
Max Wait Time: 500 millis
Max Total Connections: 8
```

2. **ExecuteSQL Configuration**
```
Database Connection Pooling Service: DBCPConnectionPool
SQL Query: SELECT * FROM employees WHERE hire_date >= '2023-01-01'
Output Format: Avro
```

3. **QueryRecord Configuration**
```sql
SELECT 
    department,
    COUNT(*) as employee_count,
    AVG(salary) as avg_salary,
    MAX(salary) as max_salary,
    CURRENT_TIMESTAMP as last_updated
FROM FLOWFILE 
GROUP BY department
```

## Exercise 7: REST API Integration

**Objective**: Fetch data from REST API, process it, and send to another API.

**Components Needed:**
- GenerateFlowFile (for triggering)
- InvokeHTTP (GET)
- EvaluateJsonPath
- UpdateAttribute
- InvokeHTTP (POST)

**API Endpoints:**
- Source: https://jsonplaceholder.typicode.com/users
- Destination: Mock webhook service

**Flow Design:**

1. **GenerateFlowFile Configuration**
```
File Size: 1 B
Batch Size: 1
Run Schedule: 60 sec
```

2. **InvokeHTTP (GET) Configuration**
```
HTTP Method: GET
Remote URL: https://jsonplaceholder.typicode.com/users
Accept: application/json
```

3. **EvaluateJsonPath Configuration**
```
user_count: $.length
user_names: $[*].name
company_names: $[*].company.name
```

4. **Create Summary JSON Template**
```json
{
  "summary": {
    "timestamp": "${now():format('yyyy-MM-dd HH:mm:ss')}",
    "user_count": ${user_count},
    "processing_status": "completed"
  },
  "users": ${user_names},
  "companies": ${company_names}
}
```

## Exercise 8: Data Enrichment Pipeline

**Objective**: Enrich incoming data with external reference data.

**Components Needed:**
- GetFile
- LookupRecord
- ConvertRecord
- PutFile

**Scenario**: Enrich customer orders with customer details

**Input Data (orders.csv):**
```csv
order_id,customer_id,product_id,quantity,amount
1001,C001,P001,2,59.98
1002,C002,P002,1,29.99
1003,C001,P003,3,89.97
```

**Reference Data (customers.json):**
```json
[
  {"customer_id": "C001", "name": "John Doe", "email": "john@example.com", "tier": "gold"},
  {"customer_id": "C002", "name": "Jane Smith", "email": "jane@example.com", "tier": "silver"},
  {"customer_id": "C003", "name": "Bob Johnson", "email": "bob@example.com", "tier": "bronze"}
]
```

**Expected Output:**
```json
{
  "order_id": "1001",
  "customer_id": "C001",
  "customer_name": "John Doe",
  "customer_email": "john@example.com", 
  "customer_tier": "gold",
  "product_id": "P001",
  "quantity": 2,
  "amount": 59.98
}
```

**Controller Services Setup:**

1. **SimpleCsvReader**
```
Schema Access Strategy: Use 'Schema Text' Property
Schema Text: 
{
  "type": "record",
  "name": "Order",
  "fields": [
    {"name": "order_id", "type": "string"},
    {"name": "customer_id", "type": "string"},
    {"name": "product_id", "type": "string"},
    {"name": "quantity", "type": "int"},
    {"name": "amount", "type": "double"}
  ]
}
```

2. **JsonTreeReader (for lookup)**
```
Schema Access Strategy: Infer Schema
```

3. **SimpleJsonSetRecordWriter**
```
Schema Write Strategy: Set 'schema.name' Attribute
Pretty Print JSON: true
```

4. **SimpleKeyValueLookupService**
```
Configuration File: /tmp/nifi-demo/customers.properties
# customers.properties content:
# C001={"name":"John Doe","email":"john@example.com","tier":"gold"}
# C002={"name":"Jane Smith","email":"jane@example.com","tier":"silver"}
```

## Exercise 9: Error Handling and Recovery

**Objective**: Implement comprehensive error handling with retry logic.

**Components Needed:**
- GetFile
- InvokeHTTP
- RouteOnAttribute
- UpdateAttribute
- Wait/Notify
- PutFile

**Error Scenarios:**
- Network timeouts
- API rate limiting
- Invalid data formats
- Server errors (5xx)

**Flow Design:**

1. **Main Processing Path**
```
GetFile → InvokeHTTP → RouteOnAttribute
```

2. **Error Handling Paths**
```
# Retry logic for temporary failures
failure → UpdateAttribute (increment retry_count) → Wait → Notify → Main Flow

# Dead letter queue for permanent failures  
failure → RouteOnAttribute (check retry_count) → PutFile (errors)
```

**Retry Logic Implementation:**
```
# UpdateAttribute (Retry Counter)
retry_count: ${retry_count:toNumber():plus(1)}
max_retries_reached: ${retry_count:toNumber():ge(3)}
retry_delay: ${retry_count:toNumber():multiply(5)} # Exponential backoff

# RouteOnAttribute (Retry Decision)
should_retry: ${max_retries_reached:equals('false')}
permanent_failure: ${max_retries_reached:equals('true')}

# Wait Processor
Release Signal Identifier: retry_${uuid}
Wait Mode: Wait for signal
Target Signal Count: 1

# Notify Processor (with delay)
Release Signal Identifier: retry_${uuid}
Signal Counter Name: retry_signal
```

## Exercise 10: Performance Monitoring Flow

**Objective**: Create a flow that monitors its own performance and generates alerts.

**Components Needed:**
- MonitorActivity
- ExecuteScript
- InvokeHTTP
- LogMessage

**Monitoring Metrics:**
- FlowFile processing rate
- Queue depths
- Error rates
- Processing times

**Implementation:**

1. **MonitorActivity Configuration**
```
Threshold Duration: 5 min
Monitoring Scope: Entire flow
Continually Send Messages: true
```

2. **ExecuteScript (Performance Collector)**
```python
# Groovy script to collect performance metrics
import groovy.json.JsonBuilder

// Get flow status
def flowStatus = context.controllerServiceLookup.getControllerService('flow-status-service')
def stats = [
    timestamp: new Date().time,
    activeThreads: flowStatus.activeThreadCount,
    queuedFlowFiles: flowStatus.queuedCount,
    bytesQueued: flowStatus.queuedContentSize
]

// Create JSON output
def json = new JsonBuilder(stats)
flowFile = session.putAttribute(flowFile, 'performance.stats', json.toString())
session.transfer(flowFile, REL_SUCCESS)
```

3. **Alert Conditions**
```
# RouteOnAttribute for alerting
high_queue_depth: ${queued_flowfiles:toNumber():gt(1000)}
low_throughput: ${processing_rate:toNumber():lt(100)}
high_error_rate: ${error_rate:toNumber():gt(0.05)}
```

These exercises progress from basic file operations to complex data processing scenarios, providing hands-on experience with NiFi's key features and best practices.
