# LLM-IoT Protocol: Server Architecture Specification

## 1. Architecture Overview

The LLM-IoT Protocol Server acts as the central hub that manages device connections, executes commands, stores telemetry data, and provides interfaces for LLM integration.

### 1.1 High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                         API Gateway Layer                        │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │
│  │ HTTP/REST    │  │  WebSocket   │  │  MQTT Broker │          │
│  │ Load Balancer│  │  Gateway     │  │  (Eclipse    │          │
│  │ (nginx)      │  │              │  │   Mosquitto) │          │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘          │
└─────────┼──────────────────┼──────────────────┼──────────────────┘
          │                  │                  │
┌─────────▼──────────────────▼──────────────────▼──────────────────┐
│                      Application Layer                            │
│  ┌───────────────────────────────────────────────────────────┐   │
│  │            Protocol Server (Node.js / Python)             │   │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐    │   │
│  │  │   Device     │  │   Command    │  │    Schema    │    │   │
│  │  │  Registry    │  │   Executor   │  │   Validator  │    │   │
│  │  │   Manager    │  │              │  │              │    │   │
│  │  └──────────────┘  └──────────────┘  └──────────────┘    │   │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐    │   │
│  │  │  Telemetry   │  │ Connection   │  │    LLM       │    │   │
│  │  │  Processor   │  │   Manager    │  │  Interface   │    │   │
│  │  │              │  │              │  │   Handler    │    │   │
│  │  └──────────────┘  └──────────────┘  └──────────────┘    │   │
│  └───────────────────────────────────────────────────────────┘   │
└───────────────────────────┬───────────────────────────────────────┘
                            │
┌───────────────────────────▼───────────────────────────────────────┐
│                      Data Layer                                   │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐           │
│  │  PostgreSQL  │  │  TimescaleDB │  │    Redis     │           │
│  │   (Device    │  │ (Time-series │  │   (Cache &   │           │
│  │   Registry,  │  │   Telemetry  │  │   Session    │           │
│  │   Schemas)   │  │     Data)    │  │   Store)     │           │
│  └──────────────┘  └──────────────┘  └──────────────┘           │
│  ┌──────────────┐  ┌──────────────┐                             │
│  │   S3 / Minio │  │  Elasticsearch│                             │
│  │  (Long-term  │  │   (Logs &    │                             │
│  │   Storage)   │  │   Search)    │                             │
│  └──────────────┘  └──────────────┘                             │
└───────────────────────────────────────────────────────────────────┘
```

## 2. Core Components

### 2.1 Device Registry Manager

**Responsibilities:**
- Device registration and UUID assignment
- Static schema storage and versioning
- Device capability discovery
- Device lifecycle management (online/offline tracking)

**Key Functions:**
```javascript
class DeviceRegistryManager {
  async registerDevice(deviceType, staticSchema)
  async deregisterDevice(deviceId)
  async getDevice(deviceId)
  async listDevices(filters)
  async updateDeviceSchema(deviceId, schema, version)
  async validateSchema(schema, version)
}
```

**Database Schema (PostgreSQL):**
```sql
-- Devices table
CREATE TABLE devices (
  device_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  device_type VARCHAR(50) NOT NULL CHECK (device_type IN ('mcu', 'sensor', 'actuator')),
  manufacturer VARCHAR(100),
  model VARCHAR(100),
  firmware_version VARCHAR(50),
  connection_status VARCHAR(20) DEFAULT 'offline',
  registered_at TIMESTAMP DEFAULT NOW(),
  last_seen TIMESTAMP,
  schema_version VARCHAR(20),
  static_schema JSONB NOT NULL,
  INDEX idx_device_type (device_type),
  INDEX idx_connection_status (connection_status)
);

-- Schema versions table
CREATE TABLE schema_versions (
  version VARCHAR(20) PRIMARY KEY,
  schema_definition JSONB NOT NULL,
  created_at TIMESTAMP DEFAULT NOW(),
  deprecated BOOLEAN DEFAULT FALSE
);

-- Device relationships table (MCU to sensors)
CREATE TABLE device_relationships (
  relationship_id SERIAL PRIMARY KEY,
  parent_device_id UUID REFERENCES devices(device_id) ON DELETE CASCADE,
  child_device_id UUID REFERENCES devices(device_id) ON DELETE CASCADE,
  connection_type VARCHAR(50),
  physical_port VARCHAR(50),
  created_at TIMESTAMP DEFAULT NOW(),
  UNIQUE(parent_device_id, child_device_id)
);
```

### 2.2 Connection Manager

**Responsibilities:**
- Maintain active connections with devices
- Heartbeat monitoring
- Connection pooling
- Automatic reconnection handling
- Connection health monitoring

**Key Functions:**
```javascript
class ConnectionManager {
  async handleHeartbeat(deviceId, timestamp, quickStatus)
  async markDeviceOnline(deviceId)
  async markDeviceOffline(deviceId, reason)
  async getConnectionStatus(deviceId)
  async sendMessage(deviceId, message)
  async broadcastMessage(deviceIds, message)
}
```

**Redis Data Structures:**
```
# Connection status (hash)
device:{device_id}:connection -> {
  status: "online" | "offline",
  last_heartbeat: timestamp,
  connection_type: "websocket" | "mqtt" | "http",
  socket_id: "string"
}

# Active connections (set)
active_connections -> Set of device_ids

# Heartbeat tracking (sorted set)
heartbeat_tracker -> {device_id: last_heartbeat_timestamp}
```

**Heartbeat Monitor (Background Process):**
```javascript
class HeartbeatMonitor {
  constructor(timeout = 120) {
    this.timeout = timeout; // seconds
  }

  async checkHeartbeats() {
    const now = Date.now();
    const deadlineTimestamp = now - (this.timeout * 1000);

    // Get devices with expired heartbeats
    const expiredDevices = await redis.zrangebyscore(
      'heartbeat_tracker',
      0,
      deadlineTimestamp
    );

    for (const deviceId of expiredDevices) {
      await this.handleTimeout(deviceId);
    }
  }

  async handleTimeout(deviceId) {
    await connectionManager.markDeviceOffline(deviceId, 'heartbeat_timeout');
    await notificationService.sendAlert({
      type: 'device_offline',
      deviceId: deviceId,
      reason: 'heartbeat_timeout'
    });
  }
}
```

### 2.3 Command Executor

**Responsibilities:**
- Validate and route commands to devices
- Track command execution status
- Handle command timeouts
- Implement command queuing for busy devices
- Command authorization

**Key Functions:**
```javascript
class CommandExecutor {
  async executeCommand(deviceId, method, params, timeout = 30000)
  async getCommandStatus(commandId)
  async cancelCommand(commandId)
  async queueCommand(deviceId, command)
  async executeBatch(commands)
}
```

**Database Schema:**
```sql
-- Commands table
CREATE TABLE commands (
  command_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  device_id UUID REFERENCES devices(device_id),
  method VARCHAR(100) NOT NULL,
  params JSONB,
  status VARCHAR(20) DEFAULT 'pending'
    CHECK (status IN ('pending', 'executing', 'completed', 'failed', 'timeout', 'cancelled')),
  issued_by VARCHAR(100),
  issued_at TIMESTAMP DEFAULT NOW(),
  started_at TIMESTAMP,
  completed_at TIMESTAMP,
  result JSONB,
  error JSONB,
  timeout_ms INTEGER,
  INDEX idx_device_commands (device_id, status),
  INDEX idx_command_status (status)
);
```

**Command Execution Flow:**
```javascript
async executeCommand(deviceId, method, params, timeout) {
  // 1. Validate device exists and is online
  const device = await deviceRegistry.getDevice(deviceId);
  if (!device) throw new Error('Device not found');
  if (device.connection_status !== 'online') {
    throw new Error('Device offline');
  }

  // 2. Create command record
  const commandId = await db.insertCommand({
    device_id: deviceId,
    method: method,
    params: params,
    timeout_ms: timeout
  });

  // 3. Send command to device
  try {
    await connectionManager.sendMessage(deviceId, {
      jsonrpc: '2.0',
      method: method,
      params: params,
      id: commandId
    });

    await db.updateCommandStatus(commandId, 'executing');

    // 4. Wait for response with timeout
    const result = await this.waitForResponse(commandId, timeout);

    await db.updateCommand(commandId, {
      status: 'completed',
      result: result,
      completed_at: new Date()
    });

    return result;

  } catch (error) {
    await db.updateCommand(commandId, {
      status: error.name === 'TimeoutError' ? 'timeout' : 'failed',
      error: error.message,
      completed_at: new Date()
    });
    throw error;
  }
}
```

### 2.4 Telemetry Processor

**Responsibilities:**
- Ingest real-time telemetry data from devices
- Validate telemetry against schemas
- Store data in time-series database
- Compute aggregations
- Trigger alerts based on thresholds

**Key Functions:**
```javascript
class TelemetryProcessor {
  async ingestTelemetry(deviceId, telemetryData)
  async processBatch(telemetryBatch)
  async computeAggregations(deviceId, interval)
  async checkAlertRules(deviceId, telemetryData)
}
```

**TimescaleDB Schema:**
```sql
-- Telemetry data (hypertable for time-series)
CREATE TABLE telemetry (
  time TIMESTAMPTZ NOT NULL,
  device_id UUID NOT NULL,
  measurement_type VARCHAR(50) NOT NULL,
  value DOUBLE PRECISION,
  unit VARCHAR(20),
  quality VARCHAR(20),
  metadata JSONB
);

-- Create hypertable
SELECT create_hypertable('telemetry', 'time');

-- Create indexes
CREATE INDEX idx_telemetry_device ON telemetry (device_id, time DESC);
CREATE INDEX idx_telemetry_type ON telemetry (measurement_type, time DESC);

-- Aggregations table
CREATE MATERIALIZED VIEW telemetry_hourly
WITH (timescaledb.continuous) AS
SELECT
  time_bucket('1 hour', time) AS bucket,
  device_id,
  measurement_type,
  AVG(value) as avg_value,
  MIN(value) as min_value,
  MAX(value) as max_value,
  COUNT(*) as sample_count
FROM telemetry
GROUP BY bucket, device_id, measurement_type;

-- Refresh policy
SELECT add_continuous_aggregate_policy('telemetry_hourly',
  start_offset => INTERVAL '3 hours',
  end_offset => INTERVAL '1 hour',
  schedule_interval => INTERVAL '1 hour');

-- Retention policy (keep raw data for 30 days)
SELECT add_retention_policy('telemetry', INTERVAL '30 days');
```

**Ingestion Pipeline:**
```javascript
async ingestTelemetry(deviceId, telemetryData) {
  // 1. Validate against device schema
  const device = await deviceRegistry.getDevice(deviceId);
  const isValid = await schemaValidator.validate(
    telemetryData,
    device.schema_version
  );

  if (!isValid) {
    throw new Error('Telemetry data validation failed');
  }

  // 2. Extract readings
  const readings = telemetryData.live_readings || [];

  // 3. Batch insert into TimescaleDB
  const insertData = readings.map(reading => ({
    time: telemetryData.timestamp,
    device_id: deviceId,
    measurement_type: reading.measurement_type,
    value: reading.value,
    unit: reading.unit,
    quality: reading.quality
  }));

  await timescaleDB.insert('telemetry', insertData);

  // 4. Update device status in Redis
  await redis.hset(`device:${deviceId}:status`, {
    last_telemetry: telemetryData.timestamp,
    last_readings: JSON.stringify(readings)
  });

  // 5. Check alert rules
  await this.checkAlertRules(deviceId, readings);

  // 6. Publish to subscribers via WebSocket/MQTT
  await eventBus.publish(`device.${deviceId}.telemetry`, telemetryData);
}
```

### 2.5 Schema Validator

**Responsibilities:**
- Validate device schemas against versioned definitions
- Validate telemetry data
- Validate command parameters
- Schema evolution and migration

**Key Functions:**
```javascript
class SchemaValidator {
  async validate(data, schemaVersion)
  async validateStaticSchema(deviceType, schema)
  async validateDynamicData(deviceId, data)
  async getSchemaDefinition(version)
}
```

**Implementation (using JSON Schema):**
```javascript
const Ajv = require('ajv');

class SchemaValidator {
  constructor() {
    this.ajv = new Ajv({ allErrors: true });
    this.schemas = new Map();
  }

  async loadSchemas() {
    const versions = await db.query('SELECT * FROM schema_versions');
    for (const version of versions) {
      this.schemas.set(version.version, version.schema_definition);
      this.ajv.addSchema(version.schema_definition, version.version);
    }
  }

  async validate(data, schemaVersion) {
    const validate = this.ajv.getSchema(schemaVersion);
    if (!validate) {
      throw new Error(`Schema version ${schemaVersion} not found`);
    }

    const valid = validate(data);
    if (!valid) {
      return {
        valid: false,
        errors: validate.errors
      };
    }

    return { valid: true };
  }
}
```

### 2.6 LLM Interface Handler

**Responsibilities:**
- Process natural language commands from LLMs
- Map LLM requests to device operations
- Provide device discovery for LLMs
- Format device data for LLM consumption
- Implement tool-calling interface (MCP-style)

**Key Functions:**
```javascript
class LLMInterfaceHandler {
  async discoverDevicesForLLM(filters)
  async interpretCommand(naturalLanguageCommand, context)
  async executeNaturalLanguageCommand(command, dryRun)
  async formatDeviceDataForLLM(deviceId)
  async generateToolDefinitions()
}
```

**Tool Definition Generation:**
```javascript
async generateToolDefinitions() {
  const devices = await deviceRegistry.listDevices({ status: 'online' });

  const tools = [];

  // Generate tool for each device capability
  for (const device of devices) {
    tools.push({
      name: `read_sensor_${device.device_id}`,
      description: `Read sensor data from ${device.model} (${device.manufacturer})`,
      input_schema: {
        type: 'object',
        properties: {
          measurement_types: {
            type: 'array',
            items: { type: 'string' },
            description: 'Types of measurements to read'
          }
        }
      }
    });
  }

  // Generate general tools
  tools.push({
    name: 'discover_devices',
    description: 'Discover all available IoT devices',
    input_schema: {
      type: 'object',
      properties: {
        device_type: {
          type: 'string',
          enum: ['all', 'mcu', 'sensor', 'actuator']
        }
      }
    }
  });

  tools.push({
    name: 'query_sensor_history',
    description: 'Query historical sensor data',
    input_schema: {
      type: 'object',
      properties: {
        device_id: { type: 'string' },
        measurement_type: { type: 'string' },
        start_time: { type: 'string', format: 'date-time' },
        end_time: { type: 'string', format: 'date-time' },
        aggregation: {
          type: 'string',
          enum: ['avg', 'min', 'max', 'sum']
        }
      },
      required: ['device_id', 'measurement_type', 'start_time', 'end_time']
    }
  });

  return tools;
}
```

## 3. API Gateway Configuration

### 3.1 HTTP/REST API (nginx configuration)

```nginx
upstream protocol_server {
  least_conn;
  server 127.0.0.1:3000 max_fails=3 fail_timeout=30s;
  server 127.0.0.1:3001 max_fails=3 fail_timeout=30s;
  server 127.0.0.1:3002 max_fails=3 fail_timeout=30s;
}

server {
  listen 443 ssl http2;
  server_name api.llm-iot.local;

  ssl_certificate /etc/ssl/certs/llm-iot.crt;
  ssl_certificate_key /etc/ssl/private/llm-iot.key;

  # JSON-RPC endpoint
  location /rpc {
    proxy_pass http://protocol_server;
    proxy_http_version 1.1;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

    # Rate limiting
    limit_req zone=api_limit burst=20 nodelay;

    # Timeouts
    proxy_connect_timeout 5s;
    proxy_send_timeout 60s;
    proxy_read_timeout 60s;
  }

  # WebSocket endpoint
  location /ws {
    proxy_pass http://protocol_server;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
    proxy_set_header Host $host;

    # WebSocket timeouts
    proxy_read_timeout 3600s;
    proxy_send_timeout 3600s;
  }

  # Health check
  location /health {
    proxy_pass http://protocol_server;
    access_log off;
  }
}

# Rate limiting zone
limit_req_zone $binary_remote_addr zone=api_limit:10m rate=100r/s;
```

### 3.2 MQTT Broker Configuration (Mosquitto)

```conf
# mosquitto.conf

# Listeners
listener 1883 0.0.0.0
protocol mqtt

listener 8883 0.0.0.0
protocol mqtt
cafile /etc/mosquitto/ca_certificates/ca.crt
certfile /etc/mosquitto/certs/server.crt
keyfile /etc/mosquitto/certs/server.key
require_certificate true

listener 9001 0.0.0.0
protocol websockets

# Authentication
allow_anonymous false
password_file /etc/mosquitto/passwd

# ACL
acl_file /etc/mosquitto/acl

# Persistence
persistence true
persistence_location /var/lib/mosquitto/

# Logging
log_dest file /var/log/mosquitto/mosquitto.log
log_type all

# Message size limits
message_size_limit 1048576

# Connection limits
max_connections 10000
max_keepalive 300
```

**ACL Configuration:**
```
# Device permissions
user device-*
topic readwrite llm-iot/devices/+/telemetry
topic readwrite llm-iot/devices/+/status
topic read llm-iot/devices/+/command/request
topic write llm-iot/devices/+/command/response

# LLM permissions
user llm-*
topic read llm-iot/devices/#
topic write llm-iot/devices/+/command/request
topic read llm-iot/devices/+/command/response
```

## 4. Deployment Architecture

### 4.1 Docker Compose Configuration

```yaml
version: '3.8'

services:
  # API Gateway
  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf
      - ./ssl:/etc/ssl
    depends_on:
      - protocol-server

  # Protocol Server (3 instances for load balancing)
  protocol-server-1:
    build: ./server
    environment:
      - NODE_ENV=production
      - PORT=3000
      - DATABASE_URL=postgresql://user:pass@postgres:5432/llm_iot
      - REDIS_URL=redis://redis:6379
      - MQTT_URL=mqtt://mosquitto:1883
    depends_on:
      - postgres
      - timescaledb
      - redis
      - mosquitto

  protocol-server-2:
    build: ./server
    environment:
      - NODE_ENV=production
      - PORT=3001
      - DATABASE_URL=postgresql://user:pass@postgres:5432/llm_iot
      - REDIS_URL=redis://redis:6379
      - MQTT_URL=mqtt://mosquitto:1883
    depends_on:
      - postgres
      - redis

  protocol-server-3:
    build: ./server
    environment:
      - NODE_ENV=production
      - PORT=3002
      - DATABASE_URL=postgresql://user:pass@postgres:5432/llm_iot
      - REDIS_URL=redis://redis:6379
      - MQTT_URL=mqtt://mosquitto:1883
    depends_on:
      - postgres
      - redis

  # PostgreSQL (Device registry)
  postgres:
    image: postgres:15-alpine
    environment:
      - POSTGRES_DB=llm_iot
      - POSTGRES_USER=user
      - POSTGRES_PASSWORD=pass
    volumes:
      - postgres-data:/var/lib/postgresql/data
    ports:
      - "5432:5432"

  # TimescaleDB (Telemetry data)
  timescaledb:
    image: timescale/timescaledb:latest-pg15
    environment:
      - POSTGRES_DB=telemetry
      - POSTGRES_USER=user
      - POSTGRES_PASSWORD=pass
    volumes:
      - timescale-data:/var/lib/postgresql/data
    ports:
      - "5433:5432"

  # Redis (Cache and session store)
  redis:
    image: redis:7-alpine
    command: redis-server --appendonly yes
    volumes:
      - redis-data:/data
    ports:
      - "6379:6379"

  # MQTT Broker
  mosquitto:
    image: eclipse-mosquitto:2
    volumes:
      - ./mosquitto.conf:/mosquitto/config/mosquitto.conf
      - mosquitto-data:/mosquitto/data
      - mosquitto-logs:/mosquitto/log
    ports:
      - "1883:1883"
      - "8883:8883"
      - "9001:9001"

  # Elasticsearch (Logs and search)
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.11.0
    environment:
      - discovery.type=single-node
      - xpack.security.enabled=false
    volumes:
      - elastic-data:/usr/share/elasticsearch/data
    ports:
      - "9200:9200"

  # MinIO (Object storage for exports)
  minio:
    image: minio/minio:latest
    command: server /data --console-address ":9001"
    environment:
      - MINIO_ROOT_USER=admin
      - MINIO_ROOT_PASSWORD=adminpassword
    volumes:
      - minio-data:/data
    ports:
      - "9000:9000"
      - "9001:9001"

volumes:
  postgres-data:
  timescale-data:
  redis-data:
  mosquitto-data:
  mosquitto-logs:
  elastic-data:
  minio-data:
```

### 4.2 Kubernetes Deployment (Production)

```yaml
# protocol-server-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: protocol-server
spec:
  replicas: 5
  selector:
    matchLabels:
      app: protocol-server
  template:
    metadata:
      labels:
        app: protocol-server
    spec:
      containers:
      - name: protocol-server
        image: llm-iot/protocol-server:latest
        ports:
        - containerPort: 3000
        env:
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: url
        resources:
          requests:
            memory: "512Mi"
            cpu: "500m"
          limits:
            memory: "1Gi"
            cpu: "1000m"
        livenessProbe:
          httpGet:
            path: /health
            port: 3000
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /ready
            port: 3000
          initialDelaySeconds: 10
          periodSeconds: 5

---
apiVersion: v1
kind: Service
metadata:
  name: protocol-server-service
spec:
  selector:
    app: protocol-server
  ports:
  - protocol: TCP
    port: 80
    targetPort: 3000
  type: LoadBalancer

---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: protocol-server-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: protocol-server
  minReplicas: 3
  maxReplicas: 20
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
```

## 5. Monitoring and Observability

### 5.1 Metrics (Prometheus)

```yaml
# Key metrics to track
- device_total{status="online|offline"}
- telemetry_ingestion_rate
- command_execution_duration_seconds
- api_request_duration_seconds
- websocket_connections_active
- mqtt_messages_received_total
- database_query_duration_seconds
- heartbeat_timeout_total
```

### 5.2 Logging (Structured JSON)

```javascript
const logger = require('winston');

logger.info('Device registered', {
  device_id: deviceId,
  device_type: deviceType,
  manufacturer: manufacturer,
  timestamp: new Date().toISOString()
});

logger.error('Command execution failed', {
  command_id: commandId,
  device_id: deviceId,
  method: method,
  error: error.message,
  stack: error.stack,
  timestamp: new Date().toISOString()
});
```

### 5.3 Distributed Tracing (OpenTelemetry)

```javascript
const { trace } = require('@opentelemetry/api');

async function executeCommand(deviceId, method, params) {
  const tracer = trace.getTracer('protocol-server');

  return tracer.startActiveSpan('execute_command', async (span) => {
    span.setAttribute('device.id', deviceId);
    span.setAttribute('command.method', method);

    try {
      const result = await commandExecutor.execute(deviceId, method, params);
      span.setStatus({ code: SpanStatusCode.OK });
      return result;
    } catch (error) {
      span.recordException(error);
      span.setStatus({ code: SpanStatusCode.ERROR });
      throw error;
    } finally {
      span.end();
    }
  });
}
```

## 6. Security Implementation

### 6.1 Authentication Middleware

```javascript
const jwt = require('jsonwebtoken');

async function authenticateDevice(req, res, next) {
  const token = req.headers['authorization']?.split(' ')[1];

  if (!token) {
    return res.status(401).json({ error: 'No token provided' });
  }

  try {
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    req.deviceId = decoded.device_id;
    req.deviceType = decoded.device_type;
    next();
  } catch (error) {
    return res.status(401).json({ error: 'Invalid token' });
  }
}
```

### 6.2 Authorization (RBAC)

```javascript
const permissions = {
  'device': ['register', 'heartbeat', 'send_telemetry'],
  'llm_read': ['discover', 'query', 'subscribe'],
  'llm_write': ['discover', 'query', 'execute', 'configure'],
  'llm_admin': ['*']
};

function authorize(role, action) {
  return (req, res, next) => {
    const allowedActions = permissions[role];
    if (allowedActions.includes('*') || allowedActions.includes(action)) {
      next();
    } else {
      res.status(403).json({ error: 'Forbidden' });
    }
  };
}
```

## 7. Backup and Recovery

### 7.1 Database Backup Strategy

```bash
# PostgreSQL backup (daily)
pg_dump -U user llm_iot > backup_$(date +%Y%m%d).sql

# TimescaleDB backup with compression
pg_dump -U user -F c -Z 9 telemetry > telemetry_backup_$(date +%Y%m%d).dump

# Redis backup (automatic with AOF)
redis-cli BGSAVE
```

### 7.2 Disaster Recovery

- **RPO (Recovery Point Objective)**: 1 hour
- **RTO (Recovery Time Objective)**: 30 minutes
- **Backup retention**: 30 days for databases, 90 days for critical exports
- **Geographic redundancy**: Multi-region database replication

## 8. Performance Optimization

### 8.1 Caching Strategy

```javascript
// Cache device schemas (1 hour TTL)
await redis.setex(`device:${deviceId}:schema`, 3600, JSON.stringify(schema));

// Cache aggregated telemetry (5 minutes TTL)
await redis.setex(`telemetry:${deviceId}:hourly`, 300, JSON.stringify(data));

// Cache LLM tool definitions (10 minutes TTL)
await redis.setex('llm:tools', 600, JSON.stringify(toolDefinitions));
```

### 8.2 Database Optimization

- Partition telemetry table by month
- Create materialized views for common aggregations
- Use connection pooling (max 100 connections)
- Implement read replicas for queries

### 8.3 Message Queue for Telemetry

```javascript
// Use Redis Streams for high-throughput telemetry buffering
await redis.xadd(
  'telemetry_stream',
  '*',
  'device_id', deviceId,
  'data', JSON.stringify(telemetryData)
);

// Consumer group processes batches
const messages = await redis.xreadgroup(
  'GROUP', 'telemetry_processors', 'consumer1',
  'BLOCK', 1000,
  'COUNT', 100,
  'STREAMS', 'telemetry_stream', '>'
);
```