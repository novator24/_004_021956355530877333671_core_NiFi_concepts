# 🎯 Demo Project: E-commerce Data Processing Pipeline

## Project Overview

**Scenario**: Real-time e-commerce order processing system that handles customer orders, enriches them with product and customer data, validates for fraud, and routes to appropriate downstream systems.

**Business Requirements:**
- Process incoming orders in real-time
- Enrich orders with customer and product information
- Detect potentially fraudulent transactions
- Route to different systems based on order characteristics
- Monitor performance and generate alerts
- Maintain data lineage and audit trail

## Architecture Overview

```
Data Sources:
├── Order Files (SFTP) → CSV files with new orders
├── Customer Database → PostgreSQL with customer details
├── Product API → REST API with product information
└── Real-time Events → Kafka stream for inventory updates

Processing Pipeline:
├── Data Ingestion → Multi-source data collection
├── Data Validation → Schema and business rule validation
├── Data Enrichment → Customer and product data joining
├── Fraud Detection → Pattern-based fraud analysis
├── Order Classification → Route based on order characteristics
└── Data Distribution → Send to downstream systems

Destinations:
├── Order Fulfillment System → High-priority orders
├── Batch Processing Queue → Standard orders
├── Fraud Investigation → Suspicious orders
├── Data Warehouse → All orders for analytics
└── Monitoring Dashboard → Real-time metrics
```

## Demo Flow Implementation

### 1. Data Sources Setup

**Order Files (CSV Format):**
```csv
order_id,customer_id,product_id,quantity,unit_price,order_date,shipping_address,payment_method
ORD001,CUST001,PROD001,2,29.99,2024-01-15T10:30:00Z,"123 Main St, City, State",credit_card
ORD002,CUST002,PROD002,1,199.99,2024-01-15T10:31:00Z,"456 Oak Ave, City, State",debit_card
ORD003,CUST001,PROD003,5,9.99,2024-01-15T10:32:00Z,"123 Main St, City, State",credit_card
```

**Customer Database Schema:**
```sql
CREATE TABLE customers (
    customer_id VARCHAR(20) PRIMARY KEY,
    first_name VARCHAR(50),
    last_name VARCHAR(50),
    email VARCHAR(100),
    phone VARCHAR(20),
    registration_date DATE,
    customer_tier VARCHAR(20),
    credit_limit DECIMAL(10,2),
    total_orders INTEGER,
    last_order_date DATE
);

-- Sample data
INSERT INTO customers VALUES
('CUST001', 'John', 'Doe', 'john.doe@email.com', '555-0101', '2023-01-15', 'GOLD', 5000.00, 25, '2024-01-10'),
('CUST002', 'Jane', 'Smith', 'jane.smith@email.com', '555-0102', '2023-06-20', 'SILVER', 2000.00, 8, '2024-01-08'),
('CUST003', 'Bob', 'Johnson', 'bob.johnson@email.com', '555-0103', '2023-12-01', 'BRONZE', 1000.00, 2, '2024-01-05');
```

**Product API Response:**
```json
{
  "product_id": "PROD001",
  "name": "Wireless Headphones",
  "category": "Electronics",
  "brand": "TechBrand",
  "price": 29.99,
  "cost": 15.00,
  "inventory_count": 150,
  "weight": 0.5,
  "dimensions": "6x4x2 inches",
  "is_active": true
}
```

### 2. NiFi Flow Configuration

#### Process Group: Order Ingestion

**Components:**
- GetSFTP (Order file pickup)
- ValidateRecord (Schema validation)
- SplitRecord (Split CSV into individual orders)
- UpdateAttribute (Add processing metadata)

**GetSFTP Configuration:**
```
Hostname: sftp.ecommerce.com
Port: 22
Username: nifi_user
Remote Path: /orders/incoming
File Filter Regex: order_\d{8}_\d{6}\.csv
Keep Source File: false
```

**ValidateRecord Configuration:**
```json
{
  "type": "record",
  "name": "Order",
  "fields": [
    {"name": "order_id", "type": "string"},
    {"name": "customer_id", "type": "string"},
    {"name": "product_id", "type": "string"},
    {"name": "quantity", "type": "int"},
    {"name": "unit_price", "type": "double"},
    {"name": "order_date", "type": "string"},
    {"name": "shipping_address", "type": "string"},
    {"name": "payment_method", "type": "string"}
  ]
}
```

#### Process Group: Data Enrichment

**Customer Enrichment:**
```sql
-- ExecuteSQL for customer lookup
SELECT 
    c.*,
    CASE 
        WHEN c.total_orders > 20 THEN 'VIP'
        WHEN c.total_orders > 10 THEN 'REGULAR'
        ELSE 'NEW'
    END as customer_segment
FROM customers c 
WHERE c.customer_id = ?
```

**Product Enrichment:**
```
# InvokeHTTP Configuration
HTTP Method: GET
Remote URL: https://api.ecommerce.com/products/${product_id}
Authorization: Bearer ${api.token}
Accept: application/json
```

**Data Joining with LookupRecord:**
```json
{
  "type": "record", 
  "name": "EnrichedOrder",
  "fields": [
    {"name": "order_id", "type": "string"},
    {"name": "customer_id", "type": "string"},
    {"name": "customer_name", "type": "string"},
    {"name": "customer_email", "type": "string"},
    {"name": "customer_tier", "type": "string"},
    {"name": "customer_segment", "type": "string"},
    {"name": "product_id", "type": "string"},
    {"name": "product_name", "type": "string"},
    {"name": "product_category", "type": "string"},
    {"name": "quantity", "type": "int"},
    {"name": "unit_price", "type": "double"},
    {"name": "total_amount", "type": "double"},
    {"name": "profit_margin", "type": "double"},
    {"name": "order_date", "type": "string"},
    {"name": "processing_timestamp", "type": "string"}
  ]
}
```

#### Process Group: Fraud Detection

**Fraud Rules Implementation:**
```bash
# UpdateAttribute - Calculate fraud indicators
order_total: ${quantity:toNumber():multiply(${unit_price:toNumber()})}
high_value_order: ${order_total:toNumber():gt(1000)}
multiple_orders_same_day: ${customer_orders_today:toNumber():gt(3)}
new_customer_high_value: ${customer_segment:equals('NEW'):and(${order_total:toNumber():gt(500)})}
unusual_shipping: ${shipping_address:contains('PO Box'):or(${shipping_address:contains('General Delivery')})}

# Calculate fraud score
fraud_score: ${high_value_order:ifElse(25, 0):plus(${multiple_orders_same_day:ifElse(20, 0)}):plus(${new_customer_high_value:ifElse(30, 0)}):plus(${unusual_shipping:ifElse(15, 0)})}

# Fraud classification
fraud_risk: ${fraud_score:toNumber():ge(50):ifElse('HIGH', ${fraud_score:toNumber():ge(25):ifElse('MEDIUM', 'LOW')})}
```

**Fraud Detection Flow:**
```
RouteOnAttribute Configuration:
high_risk_fraud: ${fraud_risk:equals('HIGH')}
medium_risk_fraud: ${fraud_risk:equals('MEDIUM')}
low_risk_normal: ${fraud_risk:equals('LOW')}
```

#### Process Group: Order Classification

**Business Rules for Routing:**
```bash
# VIP customer expedited processing
vip_order: ${customer_tier:equals('GOLD'):or(${customer_segment:equals('VIP')})}

# High-value orders
high_value: ${order_total:toNumber():gt(500)}

# Same-day delivery eligible
same_day_eligible: ${product_category:equals('Electronics'):and(${order_total:toNumber():gt(100)})}

# International shipping
international: ${shipping_address:matches('.*(?:Canada|Mexico|UK|Germany|France).*')}

# Bulk orders
bulk_order: ${quantity:toNumber():gt(10)}
```

**Routing Logic:**
```
# RouteOnAttribute priorities (evaluated in order)
1. fraud_investigation: ${fraud_risk:equals('HIGH')}
2. vip_expedited: ${vip_order:equals('true'):and(${fraud_risk:equals('LOW')})}
3. same_day_delivery: ${same_day_eligible:equals('true'):and(${fraud_risk:equals('LOW')})}
4. international_processing: ${international:equals('true')}
5. bulk_processing: ${bulk_order:equals('true')}
6. standard_processing: (default)
```

### 3. Downstream Systems Integration

#### Order Fulfillment System
```bash
# PutHTTP Configuration for fulfillment
HTTP Method: POST
Remote URL: https://fulfillment.ecommerce.com/api/orders
Content-Type: application/json
Authorization: Bearer ${fulfillment.api.token}

# JSON Template
{
  "order_id": "${order_id}",
  "customer_id": "${customer_id}",
  "priority": "${customer_tier}",
  "products": [
    {
      "product_id": "${product_id}",
      "quantity": ${quantity},
      "unit_price": ${unit_price}
    }
  ],
  "shipping_address": "${shipping_address}",
  "special_instructions": "${same_day_eligible:equals('true'):ifElse('SAME_DAY_DELIVERY', '')}"
}
```

#### Data Warehouse Loading
```bash
# PutDatabaseRecord Configuration
Table Name: fact_orders
Statement Type: INSERT
Database Connection Pooling Service: DataWarehouse_ConnectionPool

# Record-to-column mapping handled automatically based on schema
```

#### Real-time Analytics
```bash
# PublishKafka Configuration
Topic Name: order-events
Kafka Brokers: kafka-cluster:9092
Message Key Field: customer_id
Partition Strategy: Round Robin

# Message format
{
  "event_type": "order_processed",
  "timestamp": "${processing_timestamp}",
  "order_id": "${order_id}",
  "customer_id": "${customer_id}",
  "customer_tier": "${customer_tier}",
  "order_total": ${order_total},
  "fraud_risk": "${fraud_risk}",
  "processing_path": "${processing_path}"
}
```

### 4. Monitoring and Alerting

#### Performance Monitoring
```bash
# MonitorActivity Configuration
Threshold Duration: 5 minutes
Monitoring Scope: Entire flow
Activity Restored Message: Order processing resumed
Activity Lost Message: Order processing stopped - investigate immediately

# Custom Performance Metrics
# ExecuteScript (Groovy) for metrics collection
def metrics = [
    timestamp: System.currentTimeMillis(),
    orders_processed_last_hour: session.getCountersLastHour().get('orders.processed'),
    fraud_detected_count: session.getCountersLastHour().get('fraud.detected'),
    average_processing_time: session.getAverageProcessingTime('order_enrichment'),
    queue_depths: [
        enrichment_queue: session.getQueueSize('enrichment_input'),
        fraud_check_queue: session.getQueueSize('fraud_check_input'),
        fulfillment_queue: session.getQueueSize('fulfillment_input')
    ]
]

flowFile = session.putAttribute(flowFile, 'metrics.json', JsonOutput.toJson(metrics))
```

#### Alerting Rules
```bash
# Alert conditions using RouteOnAttribute
critical_alert: ${queue_depths.enrichment_queue:toNumber():gt(1000):or(${fraud_detected_count:toNumber():gt(10)})}
warning_alert: ${queue_depths.enrichment_queue:toNumber():gt(500):or(${average_processing_time:toNumber():gt(30000)})}
info_alert: ${orders_processed_last_hour:toNumber():lt(100)}

# Alert notification via InvokeHTTP to Slack/Teams webhook
{
  "text": "NiFi Alert: ${alert_level}",
  "attachments": [
    {
      "color": "${alert_level:equals('critical'):ifElse('danger', 'warning')}",
      "fields": [
        {
          "title": "Queue Depths",
          "value": "Enrichment: ${queue_depths.enrichment_queue}, Fraud Check: ${queue_depths.fraud_check_queue}",
          "short": false
        },
        {
          "title": "Processing Stats", 
          "value": "Orders/hour: ${orders_processed_last_hour}, Avg time: ${average_processing_time}ms",
          "short": false
        }
      ]
    }
  ]
}
```

### 5. Error Handling and Recovery

#### Retry Logic for External Services
```bash
# UpdateAttribute for retry management
retry_count: ${retry_count:toNumber():plus(1)}
max_retries_reached: ${retry_count:toNumber():ge(3)}
backoff_delay: ${retry_count:toNumber():multiply(${retry_count:toNumber()}):multiply(1000)} # Exponential backoff

# RouteOnAttribute for retry decision
should_retry: ${max_retries_reached:equals('false'):and(${http.status.code:matches('5\\d\\d|429')})}
permanent_failure: ${max_retries_reached:equals('true'):or(${http.status.code:matches('4\\d\\d'):and(${http.status.code:notEquals('429')})})}
success: ${http.status.code:matches('2\\d\\d')}

# Wait/Notify pattern for delays
# Wait processor
Release Signal Identifier: retry_${order_id}
Target Signal Count: 1
Wait Mode: Wait for signal

# Notify processor (with calculated delay)
Release Signal Identifier: retry_${order_id} 
Signal Counter Name: retry_signal
```

#### Dead Letter Queue
```bash
# Failed order handling
# UpdateAttribute for error metadata
error_timestamp: ${now():format('yyyy-MM-dd HH:mm:ss')}
error_reason: ${exception.message}
error_processor: ${processor.name}
retry_attempts: ${retry_count}

# PutFile for manual investigation
Directory: /data/failed_orders/${now():format('yyyy/MM/dd')}
Filename: failed_order_${order_id}_${error_timestamp}.json
```

### 6. Data Lineage and Audit

#### Provenance Tracking
```bash
# Custom provenance events using ExecuteScript
// Record custom lineage events
def provenanceEventBuilder = session.getProvenanceReporter()

// Record data enrichment event
provenanceEventBuilder.create()
    .setEventType(ProvenanceEventType.MODIFY)
    .setFlowFile(flowFile)
    .setDetails("Order enriched with customer data from database and product data from API")
    .setAttributes([
        'enrichment.customer.source': 'customer_database',
        'enrichment.product.source': 'product_api',
        'enrichment.timestamp': new Date().toString()
    ])

// Record fraud detection event  
provenanceEventBuilder.create()
    .setEventType(ProvenanceEventType.ROUTE)
    .setFlowFile(flowFile)
    .setDetails("Fraud risk assessment completed: ${fraud_risk}")
    .setAttributes([
        'fraud.score': fraud_score,
        'fraud.rules.applied': 'high_value,multiple_orders,new_customer,unusual_shipping',
        'fraud.assessment.timestamp': new Date().toString()
    ])
```

### 7. Performance Optimization

#### Concurrent Processing Configuration
```bash
# Processor thread allocation based on workload
GetSFTP: 1 concurrent task (file pickup)
ValidateRecord: 2 concurrent tasks (I/O bound)
ExecuteSQL: 4 concurrent tasks (match DB connection pool)
InvokeHTTP: 8 concurrent tasks (network bound)
LookupRecord: 4 concurrent tasks (CPU intensive)
RouteOnAttribute: 2 concurrent tasks (CPU bound)
PutDatabaseRecord: 4 concurrent tasks (match DB connection pool)
```

#### Queue Configuration
```bash
# Connection back pressure settings
Enrichment Input Queue:
  Back Pressure Object Threshold: 1000
  Back Pressure Data Size Threshold: 100 MB
  FlowFile Expiration: 1 hour

Fraud Check Queue:
  Back Pressure Object Threshold: 500  
  Back Pressure Data Size Threshold: 50 MB
  FlowFile Expiration: 30 minutes

Output Queues:
  Back Pressure Object Threshold: 2000
  Back Pressure Data Size Threshold: 200 MB
  FlowFile Expiration: 4 hours
```

## Demo Deployment Instructions

### 1. Environment Setup
```bash
# Create demo directories
mkdir -p /opt/nifi-demo/{input,output,config,logs,failed}
mkdir -p /opt/nifi-demo/input/{orders,customers,products}
mkdir -p /opt/nifi-demo/output/{fulfillment,warehouse,fraud,monitoring}

# Set permissions
chown -R nifi:nifi /opt/nifi-demo
chmod -R 755 /opt/nifi-demo
```

### 2. Controller Services Configuration
```bash
# Database Connection Pool
Database Connection URL: jdbc:postgresql://localhost:5432/ecommerce
Database Driver Class Name: org.postgresql.Driver
Database User: nifi_user
Password: nifi_password
Max Total Connections: 8
Validation Query: SELECT 1

# SSL Context Service (for HTTPS APIs)
Keystore Filename: /opt/nifi/conf/keystore.jks
Keystore Password: keystorePassword
Keystore Type: JKS
Truststore Filename: /opt/nifi/conf/truststore.jks
Truststore Password: truststorePassword
```

### 3. Sample Data Generation
```bash
# Generate sample order files
cat > /opt/nifi-demo/generate_orders.py << 'EOF'
import csv
import random
from datetime import datetime, timedelta
import json

# Generate orders
orders = []
for i in range(100):
    order = {
        'order_id': f'ORD{i+1:06d}',
        'customer_id': f'CUST{random.randint(1,50):03d}',
        'product_id': f'PROD{random.randint(1,20):03d}',
        'quantity': random.randint(1,10),
        'unit_price': round(random.uniform(9.99, 299.99), 2),
        'order_date': (datetime.now() - timedelta(minutes=random.randint(0,1440))).isoformat(),
        'shipping_address': f'{random.randint(100,9999)} {random.choice(["Main St", "Oak Ave", "Pine Rd"])}, City, State',
        'payment_method': random.choice(['credit_card', 'debit_card', 'paypal'])
    }
    orders.append(order)

# Write to CSV
with open('/opt/nifi-demo/input/orders/order_20240115_120000.csv', 'w', newline='') as f:
    writer = csv.DictWriter(f, fieldnames=orders[0].keys())
    writer.writeheader()
    writer.writerows(orders)

print(f"Generated {len(orders)} sample orders")
EOF

python3 /opt/nifi-demo/generate_orders.py
```

### 4. Flow Import
```bash
# Export the demo flow template
# (This would be done through NiFi UI)
# Template includes all process groups and configurations

# Import template via NiFi UI:
# 1. Templates menu (toolbar)
# 2. Browse and select demo-flow-template.xml
# 3. Drag template to canvas
# 4. Configure controller services
# 5. Start flow
```

## Expected Results

### Processing Flow Output
```json
{
  "order_id": "ORD000001",
  "customer_id": "CUST001", 
  "customer_name": "John Doe",
  "customer_email": "john.doe@email.com",
  "customer_tier": "GOLD",
  "customer_segment": "VIP",
  "product_id": "PROD001",
  "product_name": "Wireless Headphones",
  "product_category": "Electronics",
  "quantity": 2,
  "unit_price": 29.99,
  "total_amount": 59.98,
  "profit_margin": 14.99,
  "fraud_score": 5,
  "fraud_risk": "LOW",
  "processing_path": "vip_expedited",
  "processing_timestamp": "2024-01-15T10:35:22Z",
  "processing_duration_ms": 1250
}
```

### Performance Metrics
```json
{
  "processing_summary": {
    "total_orders_processed": 1000,
    "processing_time_avg_ms": 1200,
    "processing_time_95th_percentile_ms": 2500,
    "throughput_orders_per_minute": 50,
    "error_rate_percentage": 0.2
  },
  "fraud_detection": {
    "high_risk_orders": 12,
    "medium_risk_orders": 45, 
    "low_risk_orders": 943,
    "false_positive_rate": 0.05
  },
  "system_resources": {
    "cpu_utilization_percent": 45,
    "memory_usage_gb": 6.2,
    "disk_io_mb_per_sec": 15.5,
    "network_io_mb_per_sec": 8.2
  }
}
```

This demo project showcases a production-ready NiFi implementation with real-world complexity, demonstrating all major NiFi concepts while solving practical business problems.
