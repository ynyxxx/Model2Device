# LLM-IoT Protocol: Architecture Specification

## 1. System Overview

The LLM-IoT Protocol provides a standardized communication framework between Large Language Models (LLMs) and IoT hardware ecosystems (MCUs, sensors, actuators). The architecture supports dynamic device discovery, real-time telemetry, and LLM-driven device control.

### 1.1 Core Components

```
┌─────────────────────────────────────────────────────────────────┐
│                        LLM Interface Layer                       │
│  (Claude, GPT-4, etc.) - Natural language commands/queries      │
└────────────────────┬────────────────────────────────────────────┘
                     │
                     │ LLM-IoT Protocol API
                     │ (JSON-RPC 2.0 over HTTP/WebSocket)
                     │
┌────────────────────▼────────────────────────────────────────────┐
│                    Protocol Server (Hub)                         │
│  ┌──────────────┐  ┌──────────────┐  ┌─────────────────────┐   │
│  │   Device     │  │   Command    │  │   Schema Registry   │   │
│  │  Discovery   │  │   Executor   │  │   & Validation      │   │
│  │   Manager    │  │              │  │                     │   │
│  └──────────────┘  └──────────────┘  └─────────────────────┘   │
│  ┌──────────────┐  ┌──────────────┐  ┌─────────────────────┐   │
│  │  Telemetry   │  │  Connection  │  │   Database Layer    │   │
│  │  Aggregator  │  │   Manager    │  │  (Time-series DB)   │   │
│  └──────────────┘  └──────────────┘  └─────────────────────┘   │
└────────────────────┬────────────────────────────────────────────┘
                     │
                     │ Device Communication Layer
                     │ (MQTT, CoAP, HTTP, WebSocket)
                     │
┌────────────────────▼────────────────────────────────────────────┐
│                    MCU Gateway Layer                             │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐               │
│  │   MCU #1   │  │   MCU #2   │  │   MCU #N   │               │
│  │  (ESP32)   │  │  (RPi Pico)│  │  (Arduino) │               │
│  └─────┬──────┘  └─────┬──────┘  └─────┬──────┘               │
│        │               │               │                        │
└────────┼───────────────┼───────────────┼────────────────────────┘
         │               │               │
    ┌────▼────┐     ┌───▼────┐     ┌───▼────┐
    │ Sensors │     │Sensors │     │Sensors │
    │ I2C/SPI │     │I2C/SPI │     │I2C/SPI │
    │ Analog  │     │Analog  │     │Analog  │
    └─────────┘     └────────┘     └────────┘
```

## 2. Communication Protocol

### 2.1 Protocol Stack

The protocol uses **JSON-RPC 2.0** for standardized request/response patterns with multiple transport options:

- **HTTP/HTTPS**: Request-response for commands and queries
- **WebSocket**: Bidirectional for real-time telemetry and streaming
- **MQTT**: Publish-subscribe for sensor data and event notifications
- **CoAP**: Lightweight protocol for resource-constrained devices

### 2.2 Message Format

All messages follow JSON-RPC 2.0 structure:

**Request:**
```json
{
  "jsonrpc": "2.0",
  "method": "string",
  "params": {
    "device_id": "string (UUID)",
    "payload": {}
  },
  "id": "number or string"
}
```

**Response:**
```json
{
  "jsonrpc": "2.0",
  "result": {},
  "id": "number or string"
}
```

**Error Response:**
```json
{
  "jsonrpc": "2.0",
  "error": {
    "code": "number",
    "message": "string",
    "data": {}
  },
  "id": "number or string"
}
```

### 2.3 Error Codes

Standard error codes follow JSON-RPC 2.0 with extensions:

| Code  | Meaning                        | Description                          |
|-------|--------------------------------|--------------------------------------|
| -32700| Parse error                    | Invalid JSON received                |
| -32600| Invalid Request                | JSON-RPC structure invalid           |
| -32601| Method not found               | Method does not exist                |
| -32602| Invalid params                 | Invalid method parameters            |
| -32603| Internal error                 | Server internal error                |
| -32000| Device not found               | Device ID not registered             |
| -32001| Device offline                 | Device not responding                |
| -32002| Device busy                    | Device executing another command     |
| -32003| Communication timeout          | No response within timeout           |
| -32004| Schema validation failed       | Data doesn't match schema            |
| -32005| Unauthorized                   | Authentication/authorization failed  |
| -32006| Resource exhausted             | Memory/storage limit reached         |
| -32007| Hardware error                 | Sensor/actuator malfunction          |

## 3. Device Lifecycle

### 3.1 Device Registration Flow

```
MCU Device                Protocol Server              Schema Registry
    │                           │                             │
    │──1. Connect Request──────>│                             │
    │   (device capabilities)   │                             │
    │                           │──2. Validate Schema────────>│
    │                           │<─3. Schema Valid────────────│
    │                           │──4. Assign UUID─────────────>│
    │<──5. Registration Confirm─│   (store static schema)     │
    │   (UUID, config)          │                             │
    │                           │                             │
    │──6. Heartbeat (periodic)─>│                             │
    │<──7. ACK──────────────────│                             │
    │                           │                             │
```

**Registration Request:**
```json
{
  "jsonrpc": "2.0",
  "method": "device.register",
  "params": {
    "device_type": "mcu",
    "static_schema": {
      "manufacturer": "Espressif",
      "model": "ESP32-S3",
      "firmware_version": "1.2.0",
      "communication_protocols": {...},
      "memory": {...}
    }
  },
  "id": 1
}
```

**Registration Response:**
```json
{
  "jsonrpc": "2.0",
  "result": {
    "device_id": "550e8400-e29b-41d4-a716-446655440000",
    "assigned_at": "2025-09-30T10:30:00Z",
    "heartbeat_interval": 60,
    "telemetry_interval": 10
  },
  "id": 1
}
```

### 3.2 Sensor Discovery Flow

```
Sensor          MCU Gateway         Protocol Server       Schema Registry
   │                 │                     │                     │
   │──I2C Scan──────>│                     │                     │
   │<─Address 0x47───│                     │                     │
   │                 │──Sensor Detected───>│                     │
   │                 │   (I2C addr, type)  │                     │
   │                 │                     │──Query Schema──────>│
   │                 │                     │<─Schema Template────│
   │<─Read ID Reg────│                     │                     │
   │──ID Response───>│                     │                     │
   │                 │──Full Discovery────>│                     │
   │                 │   (complete schema) │                     │
   │                 │<─Assign Sensor UUID─│                     │
   │                 │                     │                     │
```

**Sensor Discovery Request:**
```json
{
  "jsonrpc": "2.0",
  "method": "sensor.discover",
  "params": {
    "mcu_id": "550e8400-e29b-41d4-a716-446655440000",
    "static_schema": {
      "manufacturer": "Bosch",
      "model": "BMP580",
      "communication": {
        "primary_protocol": "I2C",
        "i2c": {"address": "0x47"}
      },
      "measurements": [...]
    }
  },
  "id": 2
}
```

### 3.3 Device Disconnection Flow

```
Device              Protocol Server           Database
   │                       │                      │
   │──Heartbeat timeout───>│                      │
   │   (no response)       │                      │
   │                       │──Update Status──────>│
   │                       │  (mark offline)      │
   │                       │──Trigger Alert──────>│
   │                       │  (notify LLM)        │
   │                       │                      │
   │──Reconnect───────────>│                      │
   │<──Resume Session──────│                      │
   │   (restore config)    │                      │
```

## 4. Data Flow Architecture

### 4.1 Telemetry Data Flow (Push Model)

```
Sensor → MCU → Protocol Server → Time-Series DB
                      ↓
                LLM Interface (WebSocket)
```

**Telemetry Message:**
```json
{
  "jsonrpc": "2.0",
  "method": "telemetry.update",
  "params": {
    "device_id": "550e8400-e29b-41d4-a716-446655440001",
    "timestamp": "2025-09-30T10:35:00Z",
    "data": {
      "operational_status": {"power_state": "on"},
      "live_readings": [
        {
          "measurement_type": "temperature",
          "value": 23.5,
          "unit": "celsius",
          "quality": "good"
        }
      ]
    }
  }
}
```

### 4.2 Command Execution Flow (Pull Model)

```
LLM → Protocol Server → MCU → Sensor/Actuator
         ↓                ↓
     Validation      Execution
         ↓                ↓
    Response ←────── Result
```

**Command Request:**
```json
{
  "jsonrpc": "2.0",
  "method": "device.execute",
  "params": {
    "device_id": "550e8400-e29b-41d4-a716-446655440001",
    "command": "configure",
    "arguments": {
      "oversampling": "16X",
      "output_data_rate": 10
    }
  },
  "id": 3
}
```

**Command Response:**
```json
{
  "jsonrpc": "2.0",
  "result": {
    "status": "success",
    "execution_time_ms": 245,
    "applied_config": {
      "oversampling": "16X",
      "output_data_rate": 10
    }
  },
  "id": 3
}
```

### 4.3 Query Flow

```
LLM → Protocol Server → Database
         ↓
     Aggregation
         ↓
    Response (JSON)
```

**Query Request:**
```json
{
  "jsonrpc": "2.0",
  "method": "data.query",
  "params": {
    "device_ids": ["550e8400-e29b-41d4-a716-446655440001"],
    "measurement_types": ["temperature", "humidity"],
    "time_range": {
      "start": "2025-09-30T00:00:00Z",
      "end": "2025-09-30T23:59:59Z"
    },
    "aggregation": "avg",
    "interval": "1h"
  },
  "id": 4
}
```

## 5. Real-Time Communication

### 5.1 WebSocket Subscription Model

LLMs can subscribe to real-time device updates via WebSocket:

**Subscribe:**
```json
{
  "jsonrpc": "2.0",
  "method": "subscribe",
  "params": {
    "channels": [
      "device.550e8400-e29b-41d4-a716-446655440001.telemetry",
      "device.*.alerts"
    ]
  },
  "id": 5
}
```

**Stream Update:**
```json
{
  "jsonrpc": "2.0",
  "method": "stream.update",
  "params": {
    "channel": "device.550e8400-e29b-41d4-a716-446655440001.telemetry",
    "timestamp": "2025-09-30T10:36:00Z",
    "data": {
      "live_readings": [...]
    }
  }
}
```

### 5.2 MQTT Topic Structure

For MQTT-based communication:

```
llm-iot/
├── devices/
│   ├── {device_id}/
│   │   ├── telemetry        # Sensor data updates
│   │   ├── status           # Connection/health status
│   │   ├── command/request  # Incoming commands
│   │   └── command/response # Command results
├── discovery/
│   ├── announce             # New device announcements
│   └── query                # Discovery requests
└── alerts/
    ├── error                # Error notifications
    └── warning              # Warning notifications
```

## 6. Security Architecture

### 6.1 Authentication

- **Device Authentication**: TLS client certificates or API keys per device
- **LLM Authentication**: OAuth 2.0 or API key authentication
- **Message Signing**: HMAC-SHA256 signatures for critical commands

### 6.2 Authorization

Role-based access control (RBAC):

| Role          | Permissions                                      |
|---------------|--------------------------------------------------|
| device        | Register, send telemetry, receive commands       |
| llm_read      | Query data, subscribe to telemetry               |
| llm_write     | Execute commands, configure devices              |
| llm_admin     | Full access, including device management         |
| user_viewer   | Read-only access to dashboards                   |
| user_operator | Read + execute safe commands                     |

### 6.3 Data Encryption

- **Transport**: TLS 1.3 for all HTTP/WebSocket/MQTT connections
- **Storage**: AES-256 encryption for sensitive configuration data

## 7. Scalability Considerations

### 7.1 Horizontal Scaling

- **Protocol Server**: Stateless design allows multiple server instances behind load balancer
- **Database**: Distributed time-series database (e.g., TimescaleDB, InfluxDB)
- **Message Queue**: MQTT broker clustering for high-throughput telemetry

### 7.2 Performance Targets

| Metric                    | Target                  |
|---------------------------|-------------------------|
| Device registration       | < 500ms                 |
| Command execution         | < 1s (end-to-end)       |
| Telemetry ingestion       | 10,000 messages/sec     |
| Query response (1 device) | < 200ms                 |
| WebSocket latency         | < 50ms                  |
| Concurrent devices        | 100,000+                |

## 8. Protocol Extensions

### 8.1 Custom Method Registration

Devices can register custom methods:

```json
{
  "jsonrpc": "2.0",
  "method": "device.register_method",
  "params": {
    "device_id": "550e8400-e29b-41d4-a716-446655440000",
    "method_name": "greenhouse.adjust_irrigation",
    "schema": {
      "arguments": {
        "zone": "string",
        "duration_seconds": "number"
      }
    }
  },
  "id": 6
}
```

### 8.2 Batch Operations

Execute commands on multiple devices:

```json
{
  "jsonrpc": "2.0",
  "method": "device.batch_execute",
  "params": {
    "device_ids": ["uuid1", "uuid2", "uuid3"],
    "command": "read_sensor",
    "arguments": {"sensor_type": "temperature"}
  },
  "id": 7
}
```

## 9. Comparison with MCP

| Feature                  | MCP (Model Context Protocol) | LLM-IoT Protocol              |
|--------------------------|------------------------------|-------------------------------|
| Primary Use Case         | Static tools/resources       | Dynamic IoT device networks   |
| Device Discovery         | Manual configuration         | Automatic discovery           |
| State Management         | Stateless                    | Stateful (telemetry tracking) |
| Real-time Updates        | Request-response only        | WebSocket/MQTT streaming      |
| Schema Versioning        | Not specified                | Semantic versioning           |
| Hardware Abstraction     | Not applicable               | Comprehensive schemas         |
| Time-series Data         | Not supported                | Built-in database integration |

## 10. Implementation Roadmap

**Phase 1: Core Protocol (Weeks 1-4)**
- JSON-RPC 2.0 implementation
- Device registration and discovery
- Basic telemetry ingestion

**Phase 2: Data Layer (Weeks 5-8)**
- Time-series database integration
- Query engine
- Schema validation

**Phase 3: Real-time Features (Weeks 9-12)**
- WebSocket support
- MQTT broker integration
- Subscription management

**Phase 4: Advanced Features (Weeks 13-16)**
- Batch operations
- Custom method registration
- LLM tool integration

**Phase 5: Production Hardening (Weeks 17-20)**
- Security audit
- Performance optimization
- Monitoring and observability