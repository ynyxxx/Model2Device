# LLM-IoT Protocol: Device Interface Specification

## 1. Overview

This document defines the standardized device interfaces (endpoints) that all LLM-IoT Protocol-compliant devices must implement. These interfaces provide uniform access to device capabilities regardless of hardware type or manufacturer.

## 2. MCU Device Interfaces

### 2.1 Device Management Endpoints

#### 2.1.1 Register Device

Register a new MCU with the protocol server.

**Method:** `device.register`

**Parameters:**
```json
{
  "device_type": "mcu",
  "static_schema": {
    "manufacturer": "string",
    "model": "string",
    "firmware_version": "string",
    "power": {...},
    "processor": {...},
    "memory": {...},
    "communication_protocols": {...},
    "pins": {...}
  }
}
```

**Returns:**
```json
{
  "device_id": "string (UUID)",
  "assigned_at": "ISO 8601 datetime",
  "heartbeat_interval": "number (seconds)",
  "telemetry_interval": "number (seconds)",
  "server_time": "ISO 8601 datetime"
}
```

**Example:**
```json
{
  "jsonrpc": "2.0",
  "method": "device.register",
  "params": {
    "device_type": "mcu",
    "static_schema": {
      "manufacturer": "Espressif",
      "model": "ESP32-S3",
      "firmware_version": "2.1.0",
      "processor": {
        "architecture": "Xtensa LX7",
        "cpu_frequency": 240,
        "cores": 2
      },
      "memory": {
        "flash_total": 8388608,
        "ram_total": 524288
      }
    }
  },
  "id": 1
}
```

#### 2.1.2 Heartbeat

Send periodic heartbeat to maintain connection status.

**Method:** `device.heartbeat`

**Parameters:**
```json
{
  "device_id": "string (UUID)",
  "timestamp": "ISO 8601 datetime",
  "quick_status": {
    "power_state": "enum",
    "connection_status": "enum",
    "error_count": "number"
  }
}
```

**Returns:**
```json
{
  "acknowledged": "boolean",
  "server_time": "ISO 8601 datetime",
  "pending_commands": "number"
}
```

#### 2.1.3 Deregister Device

Gracefully disconnect device from server.

**Method:** `device.deregister`

**Parameters:**
```json
{
  "device_id": "string (UUID)",
  "reason": "string (optional)"
}
```

**Returns:**
```json
{
  "status": "deregistered",
  "timestamp": "ISO 8601 datetime"
}
```

### 2.2 Program Management Endpoints

#### 2.2.1 Upload Program

Upload new firmware or application code to MCU.

**Method:** `mcu.upload_program`

**Parameters:**
```json
{
  "device_id": "string (UUID)",
  "program_type": "enum (firmware, application, script)",
  "language": "string (e.g., CircuitPython, MicroPython, Arduino)",
  "program_data": "string (base64 encoded)",
  "program_size": "number (bytes)",
  "checksum": "string (SHA-256)",
  "metadata": {
    "name": "string",
    "version": "string",
    "description": "string"
  }
}
```

**Returns:**
```json
{
  "upload_id": "string (UUID)",
  "status": "enum (uploading, verifying, flashing, completed, failed)",
  "progress_percent": "number (0-100)",
  "estimated_time_remaining": "number (seconds)"
}
```

#### 2.2.2 Get Upload Status

Check status of ongoing program upload.

**Method:** `mcu.get_upload_status`

**Parameters:**
```json
{
  "device_id": "string (UUID)",
  "upload_id": "string (UUID)"
}
```

**Returns:**
```json
{
  "status": "enum (uploading, verifying, flashing, completed, failed)",
  "progress_percent": "number (0-100)",
  "error": "string (if failed)"
}
```

#### 2.2.3 Execute Script

Execute a script on the MCU without permanent flashing.

**Method:** `mcu.execute_script`

**Parameters:**
```json
{
  "device_id": "string (UUID)",
  "language": "string (e.g., Python, Lua)",
  "script_content": "string",
  "timeout": "number (seconds)",
  "return_output": "boolean"
}
```

**Returns:**
```json
{
  "execution_id": "string (UUID)",
  "status": "enum (running, completed, timeout, error)",
  "output": "string (stdout/stderr if return_output=true)",
  "execution_time_ms": "number"
}
```

### 2.3 Configuration Endpoints

#### 2.3.1 Get Configuration

Retrieve current MCU configuration.

**Method:** `mcu.get_config`

**Parameters:**
```json
{
  "device_id": "string (UUID)",
  "config_keys": ["string array (optional, specific keys to retrieve)"]
}
```

**Returns:**
```json
{
  "configuration": {
    "network": {...},
    "protocols": {...},
    "power_management": {...},
    "custom": {...}
  }
}
```

#### 2.3.2 Set Configuration

Update MCU configuration parameters.

**Method:** `mcu.set_config`

**Parameters:**
```json
{
  "device_id": "string (UUID)",
  "configuration": {
    "network": {
      "wifi_ssid": "string",
      "wifi_password": "string"
    },
    "telemetry_interval": "number (seconds)",
    "power_mode": "enum (normal, low_power)"
  },
  "apply_immediately": "boolean"
}
```

**Returns:**
```json
{
  "status": "success",
  "applied_config": {...},
  "requires_reboot": "boolean"
}
```

#### 2.3.3 Reboot Device

Restart the MCU.

**Method:** `mcu.reboot`

**Parameters:**
```json
{
  "device_id": "string (UUID)",
  "mode": "enum (normal, safe_mode, bootloader)"
}
```

**Returns:**
```json
{
  "status": "rebooting",
  "estimated_downtime": "number (seconds)"
}
```

### 2.4 Status Query Endpoints

#### 2.4.1 Get Dynamic Status

Retrieve full dynamic status of MCU.

**Method:** `mcu.get_status`

**Parameters:**
```json
{
  "device_id": "string (UUID)",
  "include_connected_devices": "boolean"
}
```

**Returns:**
```json
{
  "device_id": "string (UUID)",
  "timestamp": "ISO 8601 datetime",
  "operational_status": {...},
  "resource_usage": {...},
  "connected_devices": {...},
  "network_status": {...},
  "errors": {...}
}
```

#### 2.4.2 Get Resource Usage

Retrieve current resource utilization.

**Method:** `mcu.get_resources`

**Parameters:**
```json
{
  "device_id": "string (UUID)"
}
```

**Returns:**
```json
{
  "memory": {
    "flash_used": "number",
    "flash_free": "number",
    "flash_usage_percent": "number",
    "ram_used": "number",
    "ram_free": "number",
    "ram_usage_percent": "number"
  },
  "cpu": {
    "utilization_percent": "number",
    "temperature": "number (optional)"
  }
}
```

#### 2.4.3 Get Connected Sensors

List all sensors connected to this MCU.

**Method:** `mcu.get_sensors`

**Parameters:**
```json
{
  "device_id": "string (UUID)"
}
```

**Returns:**
```json
{
  "sensor_count": "number",
  "sensors": [
    {
      "sensor_id": "string (UUID)",
      "model": "string",
      "connection_type": "string (e.g., I2C, SPI)",
      "status": "enum (operational, error, disconnected)"
    }
  ]
}
```

## 3. Sensor Device Interfaces

### 3.1 Sensor Discovery

#### 3.1.1 Discover Sensor

Register a newly detected sensor.

**Method:** `sensor.discover`

**Parameters:**
```json
{
  "mcu_id": "string (UUID of host MCU)",
  "static_schema": {
    "manufacturer": "string",
    "model": "string",
    "sensor_category": "enum",
    "communication": {...},
    "measurements": [...]
  }
}
```

**Returns:**
```json
{
  "sensor_id": "string (UUID)",
  "assigned_at": "ISO 8601 datetime",
  "polling_interval": "number (seconds)"
}
```

#### 3.1.2 Scan Bus

Scan I2C/SPI bus for connected sensors.

**Method:** `mcu.scan_bus`

**Parameters:**
```json
{
  "device_id": "string (MCU UUID)",
  "bus_type": "enum (I2C, SPI)",
  "bus_number": "number (optional, default 0)"
}
```

**Returns:**
```json
{
  "bus_type": "string",
  "devices_found": "number",
  "devices": [
    {
      "address": "string (hex for I2C)",
      "device_type": "string (if identifiable)"
    }
  ]
}
```

### 3.2 Sensor Data Endpoints

#### 3.2.1 Read Sensor

Read current sensor values.

**Method:** `sensor.read`

**Parameters:**
```json
{
  "sensor_id": "string (UUID)",
  "measurement_types": ["string array (optional, specific measurements)"]
}
```

**Returns:**
```json
{
  "sensor_id": "string (UUID)",
  "timestamp": "ISO 8601 datetime",
  "readings": [
    {
      "measurement_type": "string",
      "value": "number",
      "unit": "string",
      "quality": "enum (good, fair, poor, invalid)"
    }
  ]
}
```

**Example:**
```json
{
  "jsonrpc": "2.0",
  "method": "sensor.read",
  "params": {
    "sensor_id": "550e8400-e29b-41d4-a716-446655440001"
  },
  "id": 10
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
      },
      {
        "measurement_type": "pressure",
        "value": 101325,
        "unit": "pascal",
        "quality": "good"
      }
    ]
  },
  "id": 10
}
```

#### 3.2.2 Read Raw Sensor Data

Read raw ADC or register values.

**Method:** `sensor.read_raw`

**Parameters:**
```json
{
  "sensor_id": "string (UUID)",
  "register_address": "string (hex, optional)"
}
```

**Returns:**
```json
{
  "raw_value": "number or string (hex)",
  "timestamp": "ISO 8601 datetime"
}
```

#### 3.2.3 Stream Sensor Data

Start continuous sensor data streaming.

**Method:** `sensor.start_stream`

**Parameters:**
```json
{
  "sensor_id": "string (UUID)",
  "interval": "number (seconds)",
  "duration": "number (seconds, optional)",
  "measurement_types": ["string array (optional)"]
}
```

**Returns:**
```json
{
  "stream_id": "string (UUID)",
  "status": "streaming",
  "websocket_url": "string (optional)"
}
```

#### 3.2.4 Stop Stream

Stop sensor data streaming.

**Method:** `sensor.stop_stream`

**Parameters:**
```json
{
  "sensor_id": "string (UUID)",
  "stream_id": "string (UUID)"
}
```

**Returns:**
```json
{
  "status": "stopped",
  "total_readings": "number",
  "duration": "number (seconds)"
}
```

### 3.3 Sensor Configuration Endpoints

#### 3.3.1 Configure Sensor

Apply configuration settings to sensor.

**Method:** `sensor.configure`

**Parameters:**
```json
{
  "sensor_id": "string (UUID)",
  "configuration": {
    "oversampling": "string (e.g., 16X)",
    "output_data_rate": "number (Hz)",
    "power_mode": "enum (normal, low_power)",
    "filter": "string (optional)",
    "interrupt_enabled": "boolean"
  }
}
```

**Returns:**
```json
{
  "status": "success",
  "applied_config": {...},
  "effective_sampling_rate": "number (Hz)"
}
```

#### 3.3.2 Calibrate Sensor

Initiate sensor calibration procedure.

**Method:** `sensor.calibrate`

**Parameters:**
```json
{
  "sensor_id": "string (UUID)",
  "calibration_type": "enum (zero, span, full)",
  "reference_value": "number (optional)"
}
```

**Returns:**
```json
{
  "calibration_id": "string (UUID)",
  "status": "enum (calibrating, completed, failed)",
  "calibration_data": {...},
  "timestamp": "ISO 8601 datetime"
}
```

#### 3.3.3 Reset Sensor

Reset sensor to default configuration.

**Method:** `sensor.reset`

**Parameters:**
```json
{
  "sensor_id": "string (UUID)",
  "reset_type": "enum (soft, hard)"
}
```

**Returns:**
```json
{
  "status": "reset_complete",
  "initialization_time_ms": "number"
}
```

### 3.4 Sensor Status Endpoints

#### 3.4.1 Get Sensor Status

Retrieve full sensor dynamic status.

**Method:** `sensor.get_status`

**Parameters:**
```json
{
  "sensor_id": "string (UUID)"
}
```

**Returns:**
```json
{
  "sensor_id": "string (UUID)",
  "timestamp": "ISO 8601 datetime",
  "operational_status": {...},
  "live_readings": [...],
  "configuration_state": {...},
  "statistics": {...},
  "health": {...}
}
```

#### 3.4.2 Get Sensor Health

Check sensor health and diagnostics.

**Method:** `sensor.get_health`

**Parameters:**
```json
{
  "sensor_id": "string (UUID)"
}
```

**Returns:**
```json
{
  "health_score": "number (0-100)",
  "calibration_status": "enum",
  "last_calibration": "ISO 8601 datetime",
  "error_count": "number",
  "diagnostics": {
    "connection_quality": "enum (excellent, good, fair, poor)",
    "reading_consistency": "number (0-100)",
    "noise_level": "number"
  }
}
```

## 4. Data Query Endpoints

### 4.1 Historical Data

#### 4.1.1 Query Time-Series Data

Query historical sensor data from database.

**Method:** `data.query`

**Parameters:**
```json
{
  "device_ids": ["string array (UUIDs)"],
  "measurement_types": ["string array"],
  "time_range": {
    "start": "ISO 8601 datetime",
    "end": "ISO 8601 datetime"
  },
  "aggregation": "enum (none, avg, min, max, sum, count)",
  "interval": "string (e.g., 1m, 5m, 1h, 1d)",
  "limit": "number (optional, max records)"
}
```

**Returns:**
```json
{
  "query_id": "string (UUID)",
  "total_records": "number",
  "data": [
    {
      "timestamp": "ISO 8601 datetime",
      "device_id": "string (UUID)",
      "measurement_type": "string",
      "value": "number",
      "aggregation_method": "string (if applicable)"
    }
  ]
}
```

#### 4.1.2 Export Data

Export sensor data in various formats.

**Method:** `data.export`

**Parameters:**
```json
{
  "query": {...},
  "format": "enum (json, csv, parquet, influxdb_line)",
  "compression": "enum (none, gzip, zip)"
}
```

**Returns:**
```json
{
  "export_id": "string (UUID)",
  "status": "enum (processing, ready, failed)",
  "download_url": "string (when ready)",
  "file_size": "number (bytes)",
  "expires_at": "ISO 8601 datetime"
}
```

### 4.2 Real-Time Subscriptions

#### 4.2.1 Subscribe to Device

Subscribe to real-time updates from device(s).

**Method:** `subscribe`

**Parameters:**
```json
{
  "channels": [
    "device.{device_id}.telemetry",
    "device.{device_id}.status",
    "device.*.alerts"
  ],
  "filter": {
    "measurement_types": ["string array (optional)"],
    "min_value": "number (optional)",
    "max_value": "number (optional)"
  }
}
```

**Returns:**
```json
{
  "subscription_id": "string (UUID)",
  "channels": ["string array (confirmed subscriptions)"],
  "websocket_url": "string (optional)"
}
```

#### 4.2.2 Unsubscribe

Unsubscribe from device updates.

**Method:** `unsubscribe`

**Parameters:**
```json
{
  "subscription_id": "string (UUID)"
}
```

**Returns:**
```json
{
  "status": "unsubscribed"
}
```

## 5. Batch Operations

### 5.1 Batch Read

Read multiple sensors simultaneously.

**Method:** `sensor.batch_read`

**Parameters:**
```json
{
  "sensor_ids": ["string array (UUIDs)"]
}
```

**Returns:**
```json
{
  "batch_id": "string (UUID)",
  "results": [
    {
      "sensor_id": "string (UUID)",
      "status": "enum (success, failed)",
      "readings": [...],
      "error": "string (if failed)"
    }
  ]
}
```

### 5.2 Batch Configure

Configure multiple sensors with same settings.

**Method:** `sensor.batch_configure`

**Parameters:**
```json
{
  "sensor_ids": ["string array (UUIDs)"],
  "configuration": {...}
}
```

**Returns:**
```json
{
  "batch_id": "string (UUID)",
  "results": [
    {
      "sensor_id": "string (UUID)",
      "status": "enum (success, failed)",
      "error": "string (if failed)"
    }
  ]
}
```

## 6. LLM Integration Endpoints

### 6.1 Discover All Devices

Get comprehensive list of all registered devices.

**Method:** `llm.discover_devices`

**Parameters:**
```json
{
  "device_type": "enum (all, mcu, sensor, actuator)",
  "status_filter": "enum (all, online, offline)",
  "include_schema": "boolean"
}
```

**Returns:**
```json
{
  "total_devices": "number",
  "devices": [
    {
      "device_id": "string (UUID)",
      "device_type": "string",
      "model": "string",
      "status": "enum",
      "capabilities": ["string array"],
      "static_schema": {...}
    }
  ]
}
```

### 6.2 Execute Natural Language Command

Execute LLM-interpreted command.

**Method:** `llm.execute_command`

**Parameters:**
```json
{
  "command": "string (natural language)",
  "context": {
    "user_id": "string",
    "session_id": "string",
    "target_devices": ["string array (optional)"]
  },
  "dry_run": "boolean (simulate without executing)"
}
```

**Returns:**
```json
{
  "command_id": "string (UUID)",
  "interpreted_actions": [
    {
      "device_id": "string (UUID)",
      "method": "string",
      "params": {...}
    }
  ],
  "execution_plan": "string (human-readable)",
  "status": "enum (planned, executing, completed, failed)",
  "results": [...]
}
```

**Example:**
```json
{
  "jsonrpc": "2.0",
  "method": "llm.execute_command",
  "params": {
    "command": "Read temperature from all BMP580 sensors in the greenhouse",
    "dry_run": false
  },
  "id": 20
}

// Response:
{
  "jsonrpc": "2.0",
  "result": {
    "command_id": "c0ffee00-1234-5678-9abc-def012345678",
    "interpreted_actions": [
      {
        "device_id": "550e8400-e29b-41d4-a716-446655440001",
        "method": "sensor.read",
        "params": {"measurement_types": ["temperature"]}
      }
    ],
    "execution_plan": "Reading temperature from 1 BMP580 sensor(s)",
    "status": "completed",
    "results": [
      {
        "device_id": "550e8400-e29b-41d4-a716-446655440001",
        "temperature": 23.5,
        "unit": "celsius"
      }
    ]
  },
  "id": 20
}
```

## 7. Error Handling

All endpoints follow standard JSON-RPC 2.0 error responses. See protocol_architecture.md for error codes.

**Example Error Response:**
```json
{
  "jsonrpc": "2.0",
  "error": {
    "code": -32000,
    "message": "Device not found",
    "data": {
      "device_id": "invalid-uuid",
      "suggestion": "Use llm.discover_devices to list available devices"
    }
  },
  "id": 15
}
```

## 8. Rate Limiting

| Endpoint Category        | Rate Limit (per device)  |
|--------------------------|--------------------------|
| Heartbeat                | 1 per second             |
| Sensor read              | 10 per second            |
| Configuration changes    | 5 per minute             |
| Program upload           | 1 per 5 minutes          |
| Historical queries       | 20 per minute            |
| Stream subscriptions     | 100 concurrent           |

## 9. Implementation Checklist

**Minimum MCU Implementation:**
- ✓ device.register
- ✓ device.heartbeat
- ✓ mcu.get_status
- ✓ mcu.scan_bus
- ✓ mcu.execute_script

**Minimum Sensor Implementation:**
- ✓ sensor.discover
- ✓ sensor.read
- ✓ sensor.get_status

**Advanced Features (Optional):**
- program upload
- streaming
- batch operations
- LLM command interpretation