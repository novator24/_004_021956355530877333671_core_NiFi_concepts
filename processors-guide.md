# ⚙️ Essential Processors Guide

Understanding processor categories is crucial for building effective data flows. Here are the most important categories and examples:

## File System Processors

### GetFile
**Purpose**: Monitors a directory and picks up files as they appear
**Key Properties:**
- `Input Directory`: Directory to monitor
- `File Filter`: Regex pattern to match filenames
- `Keep Source File`: Whether to leave original file
- `Minimum File Age`: Wait time before processing new files
- `Polling Interval`: How often to check for new files

**Common Use Cases:**
- File ingestion from network shares
- Processing log files as they're created
- Batch file processing

**Configuration Example:**
```
Input Directory: /data/input
File Filter: .*\.csv
Keep Source File: false
Minimum File Age: 5 sec
```

### PutFile
**Purpose**: Writes FlowFile content to the file system
**Key Properties:**
- `Directory`: Destination directory
- `Conflict Resolution`: What to do if file exists (replace, ignore, fail)
- `Create Missing Directories`: Auto-create directory structure
- `Maximum File Count`: Limit files per directory
- `Permissions`: File system permissions to set

**Configuration Example:**
```
Directory: /data/output/${now():format('yyyy/MM/dd')}
Conflict Resolution: replace
Create Missing Directories: true
```

### TailFile
**Purpose**: Monitors a file and ingests new content as it's appended
**Key Properties:**
- `Tailing mode`: Single file or rolling files
- `File to Tail`: Path to file being monitored
- `Rolling Filename Pattern`: Pattern for rolled files
- `State File`: Location to store tailing state

**Use Cases:**
- Real-time log processing
- Monitoring application logs
- Processing continuously growing files

## Execution Processors

### ExecuteShell
**Purpose**: Executes shell commands and scripts
**Key Properties:**
- `Command`: Shell command to execute
- `Command Arguments`: Arguments for the command
- `Working Directory`: Directory to run command from
- `Environment Variables`: Custom environment variables

**Security Considerations:**
- Use with caution in production
- Validate inputs to prevent command injection
- Run NiFi with limited user privileges

**Example Usage:**
```
Command: python3
Command Arguments: /scripts/process_data.py
Working Directory: /opt/processing
```

### ExecuteSQL
**Purpose**: Executes SQL queries against databases
**Key Properties:**
- `Database Connection Pooling Service`: Controller service for DB connection
- `SQL select query`: SQL query to execute
- `Output Format`: Avro or JSON output format

**Configuration Steps:**
1. Create Database Connection Pool controller service
2. Configure SQL query in processor properties
3. Handle results in downstream processors

### ExecuteScript
**Purpose**: Execute custom scripts in various languages (Python, Groovy, etc.)
**Key Properties:**
- `Script Engine`: Language engine (python, groovy, javascript)
- `Script File`: External script file path
- `Script Body`: Inline script code
- `Module Directory`: Directory for script modules

**Python Script Example:**
```python
import json
from org.apache.nifi.processor.io import StreamCallback

class PyStreamCallback(StreamCallback):
    def __init__(self):
        pass
    def process(self, inputStream, outputStream):
        text = inputStream.read()
        data = json.loads(text)
        data['processed_by'] = 'nifi_python'
        outputStream.write(json.dumps(data))

flowFile = session.get()
if flowFile != None:
    flowFile = session.write(flowFile, PyStreamCallback())
    session.transfer(flowFile, REL_SUCCESS)
```

## Data Generation Processors

### GenerateFlowFile
**Purpose**: Creates FlowFiles for testing and data generation
**Key Properties:**
- `File Size`: Size of generated FlowFiles
- `Batch Size`: Number of FlowFiles to generate at once
- `Data Format`: Format of generated data
- `Unique FlowFiles`: Whether each FlowFile should be unique

**Testing Applications:**
- Flow validation and testing
- Performance testing with synthetic data
- Development and debugging

**Configuration for Testing:**
```
File Size: 1 KB
Batch Size: 10
Data Format: Text
Unique FlowFiles: true
```

### GenerateTableFetch
**Purpose**: Generates SQL queries for incremental database processing
**Key Properties:**
- `Database Connection Pooling Service`: DB connection
- `Table Name`: Table to query
- `Maximum-value Columns`: Columns for incremental tracking
- `Where Clause`: Additional filtering

**Incremental Processing Pattern:**
```
GenerateTableFetch → ExecuteSQL → ProcessRecord → PutDatabaseRecord
```

## HTTP Processors

### GetHTTP
**Purpose**: Retrieve data from HTTP/HTTPS endpoints
**Key Properties:**
- `URL`: HTTP endpoint to call
- `Filename`: Name for retrieved FlowFiles
- `SSL Context Service`: For HTTPS connections
- `Connect Timeout`: Connection timeout
- `Read Timeout`: Read timeout

**Authentication Options:**
- Basic Authentication
- Certificate-based authentication
- Custom headers for API keys

**Configuration Example:**
```
URL: https://api.example.com/data
Accept: application/json
Authorization: Bearer ${api.token}
```

### InvokeHTTP
**Purpose**: More advanced HTTP client with full request control
**Key Properties:**
- `HTTP Method`: GET, POST, PUT, DELETE, etc.
- `Remote URL`: Target endpoint
- `Send Request Body`: Whether to send FlowFile content as request body
- `Attributes to Send`: FlowFile attributes to include as HTTP headers

**REST API Integration:**
```
HTTP Method: POST
Remote URL: https://api.example.com/webhooks
Content-Type: application/json
Authorization: Bearer ${api.token}
```

### PostHTTP
**Purpose**: Send FlowFiles via HTTP POST
**Key Properties:**
- `URL`: Destination URL
- `Send as FlowFile`: Send entire FlowFile or just content
- `Content Type`: MIME type for the content
- `Username/Password`: Basic authentication

## Content Processors

### UpdateAttribute
**Purpose**: Adds or updates FlowFile attributes
**Key Properties:**
- Dynamic properties for each attribute
- `Delete Attributes Expression`: Remove specific attributes
- `Store State`: Whether to maintain state between FlowFiles

**Common Patterns:**
```
# Add timestamp
timestamp: ${now():format('yyyy-MM-dd HH:mm:ss')}

# Extract filename without extension
filename_base: ${filename:substringBeforeLast('.')}

# Add processing status
processing_status: processed
```

### ReplaceText
**Purpose**: Performs find-and-replace operations on FlowFile content
**Key Properties:**
- `Search Value`: Text or regex to find
- `Replacement Value`: Text to replace with
- `Character Set`: Encoding for text processing
- `Maximum Buffer Size`: Memory limit for processing

**Replacement Strategies:**
- `Prepend`: Add text to beginning
- `Append`: Add text to end
- `Regex Replace`: Use regular expressions
- `Literal Replace`: Simple text replacement

### ModifyBytes
**Purpose**: Modify byte content of FlowFiles
**Key Properties:**
- `Start Offset`: Where to start modification
- `End Offset`: Where to end modification
- `Remove`: Remove specified byte range

## Routing Processors

### RouteOnAttribute
**Purpose**: Route FlowFiles based on attribute values
**Key Properties:**
- Dynamic properties for routing conditions
- `Route Strategy`: Route to Property name or Route to 'matched' if all match

**Routing Examples:**
```bash
# File size routing
small_file: ${fileSize:lt(1048576)}
medium_file: ${fileSize:ge(1048576):and(${fileSize:lt(10485760)})}
large_file: ${fileSize:ge(10485760)}

# Time-based routing
business_hours: ${now():format('HH'):toNumber():ge(9):and(${now():format('HH'):toNumber():lt(17)})}
weekend: ${now():format('u'):toNumber():gt(5)}

# Content-based routing (requires prior attribute extraction)
high_priority: ${priority:equals('urgent')}
error_condition: ${status:equals('error')}
```

### RouteOnContent
**Purpose**: Route based on FlowFile content patterns
**Key Properties:**
- `Match Requirement`: content must match ALL or ANY properties
- `Character Set`: Encoding for content processing
- Dynamic properties for content patterns

**Content Routing Patterns:**
```bash
# Route based on log levels
error_logs: (?i).*ERROR.*
warning_logs: (?i).*WARN.*
info_logs: (?i).*INFO.*

# Route based on data formats
json_data: ^\s*[\{\[].*
xml_data: ^\s*<.*
csv_data: ^[^<{]*,[^<{]*
```

### DistributeLoad
**Purpose**: Distribute FlowFiles across multiple relationships for load balancing
**Key Properties:**
- `Number of Relationships`: How many output relationships to create
- `Distribution Strategy`: round_robin, next_available, or flowfile_attribute

## Split and Merge Processors

### SplitText
**Purpose**: Split large text files into smaller chunks
**Key Properties:**
- `Line Split Count`: Number of lines per output FlowFile
- `Maximum Fragment Size`: Maximum size of output FlowFiles
- `Header Line Count`: Number of header lines to include in each fragment
- `Remove Trailing Newlines`: Clean up line endings

### SplitJson
**Purpose**: Split JSON arrays into individual JSON objects
**Key Properties:**
- `JsonPath Expression`: JSONPath to the array to split
- `Null Value Representation`: How to handle null values

### MergeContent
**Purpose**: Combine multiple FlowFiles into single FlowFile
**Key Properties:**
- `Merge Strategy`: Defragment, Bin-Packing Algorithm, Attribute Strategy
- `Merge Format`: Binary Concatenation, TAR, ZIP, FlowFile Stream
- `Minimum Entries`: Minimum number of FlowFiles to merge
- `Maximum Entries`: Maximum number of FlowFiles to merge

**Merge Strategies:**
- **Defragment**: Reassemble split FlowFiles using fragment attributes
- **Bin-Packing**: Group FlowFiles based on size and count constraints
- **Attribute Strategy**: Group FlowFiles with same attribute values

## Data Format Processors

### ConvertRecord
**Purpose**: Convert between different data formats using record readers/writers
**Key Properties:**
- `Record Reader`: Controller service to parse input format
- `Record Writer`: Controller service to write output format
- `Include Zero Record FlowFiles`: Handle empty datasets

**Format Conversions:**
- CSV → JSON
- JSON → Avro
- Avro → Parquet
- XML → JSON

### ValidateRecord
**Purpose**: Validate records against a schema
**Key Properties:**
- `Record Reader`: Parse incoming records
- `Schema Access Strategy`: How to access the schema
- `Allow Extra Fields`: Whether to allow fields not in schema
- `Strict Type Checking`: Enforce data type validation

## Database Processors

### PutDatabaseRecord
**Purpose**: Insert records into database tables
**Key Properties:**
- `Database Connection Pooling Service`: DB connection
- `Statement Type`: INSERT, UPDATE, UPSERT
- `Table Name`: Target table
- `Update Keys`: Columns to use for updates/upserts

### QueryDatabaseTable
**Purpose**: Incrementally query database tables
**Key Properties:**
- `Database Connection Pooling Service`: DB connection
- `Database Type`: Vendor-specific optimizations
- `Table Name`: Table to query
- `Maximum-value Columns`: Columns for incremental processing
- `Where Clause`: Additional filtering

**Incremental Processing:**
```
Table Name: user_events
Maximum-value Columns: event_timestamp
Where Clause: event_type = 'purchase'
```

## Monitoring and Utility Processors

### LogAttribute
**Purpose**: Log FlowFile attributes and optionally payload to NiFi logs
**Key Properties:**
- `Log Level`: DEBUG, INFO, WARN, ERROR
- `Attributes to Log`: Which attributes to include
- `Attributes to Ignore`: Which attributes to exclude
- `Log Payload`: Whether to log FlowFile content

### LogMessage
**Purpose**: Log custom messages with expression language
**Key Properties:**
- `Log Level`: Logging level
- `Log Message`: Message to log (supports expression language)

### MonitorActivity
**Purpose**: Monitor flow activity and trigger actions on inactivity
**Key Properties:**
- `Threshold Duration`: How long to wait before considering flow inactive
- `Monitoring Scope`: Monitor entire flow or just this processor's input
- `Copy Attributes`: Copy attributes from monitored FlowFiles

### Wait/Notify Processors
**Purpose**: Coordinate processing between different parts of a flow
**Wait Properties:**
- `Release Signal Identifier`: Unique identifier for the signal
- `Target Signal Count`: Number of signals to wait for
- `Wait Buffer Count`: Number of FlowFiles to buffer while waiting

**Notify Properties:**
- `Release Signal Identifier`: Identifier of signal to release
- `Signal Counter Name`: Name of the counter to increment
