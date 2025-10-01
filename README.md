# LLM-IoT Protocol Specification

A comprehensive protocol specification for standardizing connections and interactions between Large Language Models (LLMs) and IoT device ecosystems (MCUs, sensors, and actuators).

## Overview

The LLM-IoT Protocol provides a dynamic, scalable framework similar to the Model Context Protocol (MCP) but specifically designed for IoT environments with constantly changing hardware quantities and operational states. It enables automatic device discovery, real-time telemetry streaming, and LLM-driven device control.

## Key Features

- **Dynamic Device Discovery**: Automatic registration and UUID assignment for new devices
- **Comprehensive Hardware Schemas**: Capture both static specifications and dynamic operational states
- **Versioned Schema System**: Semantic versioning for schema evolution
- **Real-time Telemetry**: WebSocket and MQTT streaming for live sensor data
- **Standardized Device Interfaces**: Uniform JSON-RPC 2.0 endpoints across all devices
- **Time-series Database Integration**: Historical data storage and aggregation
- **LLM Tool Integration**: MCP-style tool definitions for natural language device control
- **Scalable Architecture**: Supports mutipile concurrent devices

## Use Cases

### Smart Greenhouse Monitoring
- Multiple MCUs managing different greenhouse zones
- Environmental sensors (temperature, humidity, soil moisture)
- Automatic climate control based on sensor readings
- LLM-driven analysis: "What's the average temperature in Zone A for the past 24 hours?"

### Multi-MCU Sensor Network
- ESP32, Raspberry Pi Pico, Arduino boards working together
- Sensors dynamically connect/disconnect
- Centralized data aggregation and control
- LLM commands: "Read all BMP580 temperature sensors" or "Configure all soil sensors to low-power mode"

## Architecture

```
┌─────────────┐
│  LLM Layer  │  (Claude, GPT-4, etc.)
└──────┬──────┘
       │ JSON-RPC 2.0
┌──────▼──────────────────────┐
│  Protocol Server (Hub)      │
│  - Device Registry          │
│  - Command Executor         │
│  - Telemetry Processor      │
│  - Schema Validator         │
└──────┬──────────────────────┘
       │ MQTT / WebSocket / HTTP
┌──────▼──────────────────────┐
│  MCU Gateway Layer          │
│  (ESP32, RPi Pico, Arduino) │
└──────┬──────────────────────┘
       │ I2C / SPI / Analog
┌──────▼──────────────────────┐
│  Sensors & Actuators        │
│  (BMP580, AS5600, etc.)     │
└─────────────────────────────┘
```

## Documentation Structure

This specification consists of four core documents:

### 1. [Hardware Schema Specification](hardware_schema_specification.md)
Defines the complete schema system for MCUs and sensors:
- **MCU Static Schema**: Processor specs, memory, communication protocols, pin definitions
- **MCU Dynamic Schema**: Operational status, resource usage, connected devices
- **Sensor Static Schema**: Measurement capabilities, communication protocols, configuration options
- **Sensor Dynamic Schema**: Live readings, calibration status, health metrics
- **Validation Examples**: Real-world schemas from BMP580, AS5600, and soil moisture sensors

### 2. [Protocol Architecture](protocol_architecture.md)
Describes the communication protocol and data flow:
- **JSON-RPC 2.0 Message Format**: Request/response patterns
- **Device Lifecycle**: Registration, heartbeat, disconnection flows
- **Communication Layers**: HTTP, WebSocket, MQTT, CoAP
- **Telemetry Pipeline**: Push/pull models for sensor data
- **Command Execution**: Synchronous and asynchronous operations
- **Security**: Authentication, authorization, encryption
- **Scalability**: Performance targets and horizontal scaling

### 3. [Device Interface Specification](device_interface_specification.md)
Comprehensive list of all standardized endpoints:
- **MCU Interfaces**:
  - Device management (register, heartbeat, deregister)
  - Program management (upload, execute scripts)
  - Configuration (get/set config, reboot)
  - Status queries (resources, connected sensors)
- **Sensor Interfaces**:
  - Discovery (scan bus, register sensor)
  - Data operations (read, stream, read raw)
  - Configuration (configure, calibrate, reset)
  - Health monitoring
- **Data Query**: Historical queries, exports, real-time subscriptions
- **Batch Operations**: Multi-device commands
- **LLM Integration**: Natural language command execution

### 4. [Server Architecture](server_architecture.md)
Server implementation details and deployment:
- **Core Components**:
  - Device Registry Manager (PostgreSQL)
  - Connection Manager (Redis)
  - Command Executor (async processing)
  - Telemetry Processor (TimescaleDB)
  - Schema Validator (JSON Schema)
  - LLM Interface Handler (tool generation)
- **Database Schemas**: SQL definitions for devices, telemetry, commands
- **API Gateway**: nginx + load balancing
- **MQTT Broker**: Mosquitto configuration
- **Deployment**: Docker Compose and Kubernetes configs
- **Monitoring**: Prometheus metrics, structured logging, distributed tracing
- **Security**: JWT authentication, RBAC authorization
- **Performance**: Caching, database optimization, message queues

## Protocol Comparison

| Feature | MCP | LLM-IoT Protocol |
|---------|-----|------------------|
| Primary Use Case | Static tools/resources | Dynamic IoT networks |
| Device Discovery | Manual configuration | Automatic discovery |
| State Management | Stateless | Stateful telemetry tracking |
| Real-time Updates | Request-response only | WebSocket/MQTT streaming |
| Schema Versioning | Not specified | Semantic versioning |
| Hardware Abstraction | Not applicable | Comprehensive schemas |
| Time-series Data | Not supported | Built-in database integration |

## Quick Start Example

### Device Registration
```json
{
  "jsonrpc": "2.0",
  "method": "device.register",
  "params": {
    "device_type": "mcu",
    "static_schema": {
      "manufacturer": "Espressif",
      "model": "ESP32-S3",
      "memory": {
        "flash_total": 8388608,
        "ram_total": 524288
      },
      "communication_protocols": {
        "supported": ["I2C", "SPI", "WiFi", "BLE"]
      }
    }
  },
  "id": 1
}
```

### Sensor Discovery
```json
{
  "jsonrpc": "2.0",
  "method": "sensor.discover",
  "params": {
    "mcu_id": "550e8400-e29b-41d4-a716-446655440000",
    "static_schema": {
      "manufacturer": "Bosch",
      "model": "BMP580",
      "sensor_category": "environmental",
      "communication": {
        "primary_protocol": "I2C",
        "i2c": {"address": "0x47"}
      },
      "measurements": [
        {
          "measurement_type": "temperature",
          "unit": "celsius",
          "range_min": -40,
          "range_max": 85,
          "accuracy": 0.5
        }
      ]
    }
  },
  "id": 2
}
```

### Read Sensor Data
```json
{
  "jsonrpc": "2.0",
  "method": "sensor.read",
  "params": {
    "sensor_id": "550e8400-e29b-41d4-a716-446655440001"
  },
  "id": 3
}

// Response:
{
  "jsonrpc": "2.0",
  "result": {
    "sensor_id": "550e8400-e29b-41d4-a716-446655440001",
    "timestamp": "2025-09-30T10:35:00Z",
    "readings": [
      {
        "measurement_type": "temperature",
        "value": 23.5,
        "unit": "celsius",
        "quality": "good"
      }
    ]
  },
  "id": 3
}
```

### LLM Natural Language Command
```json
{
  "jsonrpc": "2.0",
  "method": "llm.execute_command",
  "params": {
    "command": "Read temperature from all BMP580 sensors in the greenhouse"
  },
  "id": 4
}
```

### Query Historical Data
```json
{
  "jsonrpc": "2.0",
  "method": "data.query",
  "params": {
    "device_ids": ["550e8400-e29b-41d4-a716-446655440001"],
    "measurement_types": ["temperature"],
    "time_range": {
      "start": "2025-09-30T00:00:00Z",
      "end": "2025-09-30T23:59:59Z"
    },
    "aggregation": "avg",
    "interval": "1h"
  },
  "id": 5
}
```

## Validated Hardware

This protocol specification has been validated against real-world sensor datasheets:

- **BMP580/581/585** (Adafruit): Temperature and pressure sensor with I2C/SPI, configurable oversampling, IIR filters, and power modes
- **AS5600** (Adafruit): Magnetic angle sensor with 12-bit resolution, I2C/analog/PWM outputs, and calibration support
- **Simple Soil Moisture Sensor** (Adafruit): Resistive analog sensor with gold-plated prongs and power management

All schema examples in the documentation are based on these real sensors.

## Performance Targets

| Metric | Target |
|--------|--------|
| Device registration | < 500ms |
| Command execution | < 1s (end-to-end) |
| Telemetry ingestion | 10,000 messages/sec |
| Query response | < 200ms |
| WebSocket latency | < 50ms |
| Concurrent devices | 100,000+ |

## Technology Stack

**Protocol Server:**
- Node.js or Python (FastAPI)
- JSON-RPC 2.0 library
- WebSocket library

**Databases:**
- PostgreSQL (device registry)
- TimescaleDB (time-series telemetry)
- Redis (caching and sessions)

**Message Broker:**
- Eclipse Mosquitto (MQTT)
- Redis Streams (high-throughput buffering)

**Infrastructure:**
- nginx (API gateway & load balancer)
- Docker / Kubernetes (deployment)
- Prometheus + Grafana (monitoring)
- Elasticsearch (logging)

**Security:**
- TLS 1.3 (transport encryption)
- JWT (authentication)
- OAuth 2.0 (LLM authentication)
- RBAC (authorization)

## References

- [Model Context Protocol (MCP)](https://modelcontextprotocol.io/) - Inspiration for tool-calling mechanism
- [JSON-RPC 2.0 Specification](https://www.jsonrpc.org/specification)
- [Adafruit Sensor Documentation](https://learn.adafruit.com/)
- [MQTT Protocol](https://mqtt.org/)
- [TimescaleDB Documentation](https://docs.timescale.com/)
---

**Last Updated:** 2025-10-1
