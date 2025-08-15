# 📚 NiFi Registry Guide

Apache NiFi Registry is a complementary application that provides versioning and storage for shared resources such as templates and configurations.

## Registry Concepts

### Versioned Flows
**Purpose**: Track changes to process groups over time
**Benefits:**
- Version control for data flows
- Collaboration among team members
- Rollback capabilities
- Change tracking and auditing

### Buckets
**Purpose**: Organizational containers for versioned flows
**Structure:**
```
Registry
├── Development Bucket
│   ├── Customer Processing Flow v1.0
│   ├── Order Processing Flow v2.1
│   └── Data Validation Flow v1.3
├── Production Bucket
│   ├── Customer Processing Flow v1.2
│   └── Order Processing Flow v2.0
└── Templates Bucket
    ├── Standard Processors Template
    └── Security Template
```

## Setting Up NiFi Registry

### Installation and Configuration

1. **Download and Install**
```bash
wget https://archive.apache.org/dist/nifi/nifi-registry/1.23.2/nifi-registry-1.23.2-bin.tar.gz
tar -xzf nifi-registry-1.23.2-bin.tar.gz
cd nifi-registry-1.23.2
```

2. **Configure Registry**
```properties
# nifi-registry.properties
nifi.registry.web.http.port=18080
nifi.registry.web.http.host=localhost

# Database configuration (H2 for development)
nifi.registry.db.url=jdbc:h2:./database/nifi-registry
nifi.registry.db.driver.class=org.h2.Driver
```

3. **Start Registry**
```bash
./bin/nifi-registry.sh start
```

4. **Access Registry UI**
- URL: http://localhost:18080/nifi-registry

### Connecting NiFi to Registry

1. **Configure Registry Client in NiFi**
   - Go to Controller Settings (hamburger menu)
   - Select Registry Clients tab
   - Add new Registry Client
   - Configure URL: http://localhost:18080

2. **Create Bucket in Registry**
   - Access Registry UI
   - Create new bucket (e.g., "Development")
   - Set appropriate permissions

## Working with Versioned Flows

### Creating a Versioned Flow

1. **Prepare Process Group**
   - Create a process group with your flow
   - Ensure all components are properly configured
   - Stop all processors in the group

2. **Start Version Control**
   - Right-click process group
   - Select "Version" → "Start version control"
   - Choose registry client and bucket
   - Provide flow name and description
   - Click "Save"

3. **Version Information**
```
Flow Name: Customer Data Processing
Version: 1.0
Description: Initial version with basic customer data ingestion and validation
Registry: Development Registry
Bucket: Customer Flows
```

### Managing Flow Versions

**Creating New Versions:**
1. Make changes to your flow
2. Right-click process group
3. Select "Version" → "Commit local changes"
4. Provide version comments
5. Save new version

**Reverting Changes:**
1. Right-click process group
2. Select "Version" → "Revert local changes"
3. Confirm reversion

**Changing Flow Version:**
1. Right-click process group
2. Select "Version" → "Change version"
3. Select desired version
4. Apply changes

### Flow Import/Export

**Exporting Flows:**
```bash
# Export flow from registry
curl -X GET http://localhost:18080/nifi-registry-api/buckets/{bucket-id}/flows/{flow-id}/versions/{version} \
  -H "Accept: application/json" > flow-export.json
```

**Importing Flows:**
1. Create new process group
2. Right-click and select "Upload template"
3. Select exported flow file
4. Configure version control if needed

## Registry Security

### User Authentication
```properties
# nifi-registry.properties - LDAP Authentication
nifi.registry.security.user.login.identity.provider=ldap-provider

# providers.xml configuration
<loginIdentityProviders>
    <provider>
        <identifier>ldap-provider</identifier>
        <class>org.apache.nifi.registry.security.ldap.LdapProvider</class>
        <property name="Authentication Strategy">SIMPLE</property>
        <property name="Manager DN">cn=admin,dc=example,dc=com</property>
        <property name="Manager Password">password</property>
        <property name="TLS - Keystore">/opt/nifi-registry/conf/keystore.jks</property>
        <property name="TLS - Keystore Password">keystorePassword</property>
        <property name="TLS - Keystore Type">JKS</property>
        <property name="Url">ldap://ldap.example.com:389</property>
        <property name="User Search Base">ou=users,dc=example,dc=com</property>
        <property name="User Search Filter">uid={0}</property>
    </provider>
</loginIdentityProviders>
```

### Authorization Policies
**Policy Types:**
- **Bucket policies**: Control access to specific buckets
- **Flow policies**: Control access to individual flows
- **Proxy policies**: Control proxy user capabilities

**Setting Policies:**
1. Access Registry UI as admin
2. Go to Settings → Access Policies
3. Select policy type and resource
4. Add users/groups and permissions

## Best Practices for Registry Usage

### Naming Conventions
```
Flow Names: {Domain}_{Function}_{Version}
- Customer_DataIngestion_v1
- Order_Processing_v2
- Payment_Validation_v1

Bucket Names: {Environment}_{Team}
- Production_DataEngineering
- Development_Analytics
- Testing_QualityAssurance
```

### Version Management Strategy
```
Version Numbering: {Major}.{Minor}.{Patch}
- Major: Breaking changes or significant architecture updates
- Minor: New features, processor additions
- Patch: Bug fixes, configuration updates

Example:
v1.0.0 → Initial release
v1.1.0 → Added error handling
v1.1.1 → Fixed configuration bug
v2.0.0 → Redesigned for performance
```

### Development Workflow
```
1. Development Environment
   ├── Create feature branch flow
   ├── Develop and test locally
   ├── Commit to development bucket
   └── Peer review

2. Testing Environment  
   ├── Import from development bucket
   ├── Run integration tests
   ├── Performance validation
   └── Security testing

3. Production Environment
   ├── Import tested version
   ├── Gradual rollout
   ├── Monitor performance
   └── Document deployment
```

## Advanced Registry Features

### Extension Registry
**Purpose**: Manage custom processors and controller services
**Configuration:**
```properties
# Extension registry configuration
nifi.registry.extension.repository.directory=./extension_repository
nifi.registry.extension.repository.enabled=true
```

### Database Backend
**PostgreSQL Configuration:**
```properties
# PostgreSQL backend for production
nifi.registry.db.url=jdbc:postgresql://localhost:5432/nifi_registry
nifi.registry.db.driver.class=org.postgresql.Driver
nifi.registry.db.username=nifi_registry_user
nifi.registry.db.password=nifi_registry_password
nifi.registry.db.driver.directory=/opt/nifi-registry/drivers
```

### High Availability
**Load Balancer Configuration:**
```nginx
# Nginx configuration for HA Registry
upstream nifi_registry {
    server registry1.company.com:18080;
    server registry2.company.com:18080;
    server registry3.company.com:18080;
}

server {
    listen 80;
    server_name registry.company.com;
    
    location / {
        proxy_pass http://nifi_registry;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

## API Integration

### REST API Examples

**List All Buckets:**
```bash
curl -X GET http://localhost:18080/nifi-registry-api/buckets \
  -H "Accept: application/json"
```

**Create New Bucket:**
```bash
curl -X POST http://localhost:18080/nifi-registry-api/buckets \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Production_Flows",
    "description": "Production environment flows",
    "allowBundleRedeploy": false,
    "allowPublicRead": false
  }'
```

**List Flows in Bucket:**
```bash
curl -X GET http://localhost:18080/nifi-registry-api/buckets/{bucket-id}/flows \
  -H "Accept: application/json"
```

**Get Flow Version:**
```bash
curl -X GET http://localhost:18080/nifi-registry-api/buckets/{bucket-id}/flows/{flow-id}/versions/{version} \
  -H "Accept: application/json"
```

## Troubleshooting

### Common Issues

**Issue 1: Connection Problems**
```bash
# Check registry connectivity
curl -v http://localhost:18080/nifi-registry

# Verify registry client configuration in NiFi
# Controller Settings → Registry Clients
```

**Issue 2: Version Control Conflicts**
```bash
# Symptoms: Cannot commit changes
# Solution: Check for concurrent modifications
# 1. Revert local changes
# 2. Update to latest version
# 3. Reapply changes
# 4. Commit new version
```

**Issue 3: Authentication Failures**
```bash
# Check authentication provider configuration
# Verify user credentials
# Review security logs in Registry
tail -f logs/nifi-registry-security.log
```

### Maintenance Tasks

**Database Maintenance:**
```sql
-- PostgreSQL maintenance queries
-- Check registry database size
SELECT pg_size_pretty(pg_database_size('nifi_registry'));

-- Analyze flow version storage
SELECT bucket_name, flow_name, COUNT(*) as version_count
FROM flow_snapshot
GROUP BY bucket_name, flow_name
ORDER BY version_count DESC;
```

**Backup Procedures:**
```bash
# Backup registry database
pg_dump nifi_registry > nifi_registry_backup_$(date +%Y%m%d).sql

# Backup registry configuration
tar -czf nifi_registry_config_$(date +%Y%m%d).tar.gz conf/

# Backup flow snapshots (if using file-based storage)
tar -czf flow_storage_$(date +%Y%m%d).tar.gz flow_storage/
```
