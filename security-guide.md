# 🔒 Security Features and Implementation

NiFi provides comprehensive security features to protect data flows, ensure proper authentication, and maintain data privacy.

## Authentication Methods

### Username/Password Authentication

**Single User Provider (Development):**
```xml
<!-- conf/login-identity-providers.xml -->
<loginIdentityProviders>
    <provider>
        <identifier>single-user-provider</identifier>
        <class>org.apache.nifi.authentication.single.user.SingleUserLoginIdentityProvider</class>
        <property name="Username">admin</property>
        <property name="Password">$2b$12$hWmVXYAMZZsLWqb</property>
    </provider>
</loginIdentityProviders>
```

**LDAP Authentication (Enterprise):**
```xml
<provider>
    <identifier>ldap-provider</identifier>
    <class>org.apache.nifi.ldap.LdapProvider</class>
    <property name="Authentication Strategy">SIMPLE</property>
    <property name="Manager DN">cn=admin,dc=company,dc=com</property>
    <property name="Manager Password">adminPassword</property>
    <property name="TLS - Keystore">/opt/nifi/conf/keystore.jks</property>
    <property name="TLS - Keystore Password">keystorePassword</property>
    <property name="TLS - Keystore Type">JKS</property>
    <property name="Url">ldaps://ldap.company.com:636</property>
    <property name="User Search Base">ou=users,dc=company,dc=com</property>
    <property name="User Search Filter">uid={0}</property>
</provider>
```

### Certificate-Based Authentication

**Client Certificate Setup:**
```bash
# Generate client certificate
keytool -genkeypair -alias client-cert \
  -keyalg RSA -keysize 2048 \
  -keystore client-keystore.jks \
  -storepass clientPassword \
  -dname "CN=client,OU=IT,O=Company,L=City,ST=State,C=US"

# Export client certificate
keytool -exportcert -alias client-cert \
  -keystore client-keystore.jks \
  -storepass clientPassword \
  -file client-cert.crt

# Import to NiFi truststore
keytool -importcert -alias client-cert \
  -keystore truststore.jks \
  -storepass truststorePassword \
  -file client-cert.crt
```

**NiFi Configuration:**
```properties
# nifi.properties
nifi.security.keystore=/opt/nifi/conf/keystore.jks
nifi.security.keystoreType=JKS
nifi.security.keystorePasswd=keystorePassword
nifi.security.keyPasswd=keyPassword
nifi.security.truststore=/opt/nifi/conf/truststore.jks
nifi.security.truststoreType=JKS
nifi.security.truststorePasswd=truststorePassword
nifi.security.needClientAuth=true
```

## Authorization and Access Control

### User Groups and Roles

**Common Role Definitions:**
```
Roles:
├── Administrator
│   ├── Full system access
│   ├── User management
│   ├── Policy management
│   └── System configuration
├── Data Engineer
│   ├── Create/modify flows
│   ├── Manage controller services
│   ├── Access provenance data
│   └── Monitor system performance
├── Data Analyst
│   ├── View flows (read-only)
│   ├── Access data provenance
│   ├── Run reports
│   └── View system status
└── Auditor
    ├── Read-only access to all flows
    ├── Full provenance access
    ├── Security event monitoring
    └── Compliance reporting
```

### Access Policies

**Policy Types and Scope:**
```xml
<!-- authorizers.xml -->
<authorizers>
    <userGroupProvider>
        <identifier>file-user-group-provider</identifier>
        <class>org.apache.nifi.authorization.FileUserGroupProvider</class>
        <property name="Users File">conf/users.xml</property>
        <property name="Legacy Authorized Users File"></property>
        <property name="Initial User Identity 1">CN=admin,OU=IT,O=Company</property>
    </userGroupProvider>
    
    <accessPolicyProvider>
        <identifier>file-access-policy-provider</identifier>
        <class>org.apache.nifi.authorization.FileAccessPolicyProvider</class>
        <property name="User Group Provider">file-user-group-provider</property>
        <property name="Authorizations File">conf/authorizations.xml</property>
        <property name="Initial Admin Identity">CN=admin,OU=IT,O=Company</property>
    </accessPolicyProvider>
</authorizers>
```

**Granular Access Control:**
```
Resource Types:
├── /flow → Overall data flow
├── /data/{component-type}/{component-id} → Specific components
├── /provenance → Data provenance events
├── /data-transfer/{port-id} → Remote process group ports
├── /site-to-site → Site-to-site communications
├── /system → System-level operations
├── /restricted-components → Restricted processor usage
├── /policies/{resource} → Policy management
└── /tenants → User and group management

Actions:
├── READ → View resource
├── WRITE → Modify resource
└── DELETE → Remove resource
```

## Data Encryption

### Data at Rest Encryption

**Content Repository Encryption:**
```properties
# nifi.properties
nifi.content.repository.implementation=org.apache.nifi.controller.repository.crypto.EncryptedFileSystemRepository
nifi.content.repository.directory.default=./content_repository

# Encryption properties
nifi.content.repository.encryption.key.provider.implementation=org.apache.nifi.security.kms.StaticKeyProvider
nifi.content.repository.encryption.key.provider.keystore.location=/opt/nifi/conf/repository-keystore.jks
nifi.content.repository.encryption.key.provider.keystore.password=repositoryPassword
```

**Provenance Repository Encryption:**
```properties
nifi.provenance.repository.implementation=org.apache.nifi.provenance.EncryptedWriteAheadProvenanceRepository
nifi.provenance.repository.encryption.key.provider.implementation=org.apache.nifi.security.kms.StaticKeyProvider
nifi.provenance.repository.encryption.key.provider.keystore.location=/opt/nifi/conf/provenance-keystore.jks
```

### Data in Transit Encryption

**SSL/TLS Configuration:**
```properties
# HTTPS configuration
nifi.web.https.port=8443
nifi.web.https.host=0.0.0.0
nifi.security.keystore=/opt/nifi/conf/keystore.jks
nifi.security.keystoreType=JKS
nifi.security.keystorePasswd=keystorePassword
nifi.security.keyPasswd=keyPassword
nifi.security.truststore=/opt/nifi/conf/truststore.jks
nifi.security.truststoreType=JKS
nifi.security.truststorePasswd=truststorePassword

# Site-to-Site encryption
nifi.remote.input.secure=true
nifi.remote.input.socket.port=10443
```

## Sensitive Property Protection

**Property Encryption:**
```xml
<!-- bootstrap.conf -->
java.arg.15=-Dnifi.bootstrap.sensitive.key=sensitiveKeyHere
```

**Automatic Encryption:**
```properties
# Any property marked as sensitive is automatically encrypted
nifi.sensitive.props.key=mySecretKey
nifi.sensitive.props.key.protected=aes/gcm/256
nifi.sensitive.props.algorithm=NIFI_PBKDF2_AES_GCM_256
```

## Security Best Practices

### Hardening Checklist

**System Level:**
```bash
# 1. Run NiFi as non-root user
useradd -r -s /bin/false nifi
chown -R nifi:nifi /opt/nifi

# 2. Restrict file permissions
chmod 750 /opt/nifi
chmod 640 /opt/nifi/conf/*.properties
chmod 600 /opt/nifi/conf/*.jks

# 3. Network security
# - Use firewall to restrict access to NiFi ports
# - Implement network segmentation
# - Use VPN for remote access
```

**Configuration Security:**
```properties
# 1. Disable HTTP (use HTTPS only)
nifi.web.http.port=
nifi.web.https.port=8443

# 2. Strong SSL configuration
nifi.security.needClientAuth=true
nifi.web.https.ciphersuites.include=TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384

# 3. Session timeout
nifi.security.user.web.session.timeout=12 hours

# 4. Content viewing restrictions
nifi.content.viewer.url=./conf/content-viewer.properties
```

### Audit and Compliance

**Security Event Logging:**
```xml
<!-- logback.xml -->
<appender name="SECURITY_FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
    <file>logs/nifi-security.log</file>
    <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
        <fileNamePattern>logs/nifi-security.%d{yyyy-MM-dd}.%i.log</fileNamePattern>
        <maxFileSize>100MB</maxFileSize>
        <maxHistory>30</maxHistory>
    </rollingPolicy>
    <encoder>
        <pattern>%date %level [%thread] %logger{40} %msg%n</pattern>
    </encoder>
</appender>

<logger name="org.apache.nifi.web.security" level="INFO" additivity="false">
    <appender-ref ref="SECURITY_FILE"/>
</logger>
```

**Compliance Reporting:**
```sql
-- Example compliance queries for security events
SELECT 
    event_time,
    user_identity,
    action,
    resource,
    source_ip
FROM security_audit_log 
WHERE event_time >= NOW() - INTERVAL '24 HOURS'
AND action IN ('read', 'write', 'delete')
ORDER BY event_time DESC;
```
