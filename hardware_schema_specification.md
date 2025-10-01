# LLM-IoT Protocol: Hardware Schema Specification

## 1. MCU Hardware Schema

### 1.1 Static Attributes

```json
{
  "device_type": "mcu",
  "device_id": "string (auto-assigned UUID)",
  "manufacturer": "string",
  "model": "string",
  "firmware_version": "string",

  "power": {
    "input_voltage_min": "number (volts)",
    "input_voltage_max": "number (volts)",
    "operating_voltage": "number (volts)",
    "power_consumption_idle": "number (mA)",
    "power_consumption_active": "number (mA)"
  },

  "processor": {
    "architecture": "string (e.g., ARM Cortex-M4, ESP32)",
    "cpu_frequency": "number (MHz)",
    "cores": "number"
  },

  "memory": {
    "flash_total": "number (bytes)",
    "ram_total": "number (bytes)",
    "eeprom_total": "number (bytes, optional)"
  },

  "communication_protocols": {
    "supported": ["string array (e.g., I2C, SPI, UART, USB, WiFi, BLE)"],
    "i2c": {
      "available": "boolean",
      "max_speed": "number (Hz)",
      "pins": {
        "sda": "number or string",
        "scl": "number or string"
      }
    },
    "spi": {
      "available": "boolean",
      "max_speed": "number (Hz)",
      "pins": {
        "miso": "number or string",
        "mosi": "number or string",
        "sck": "number or string",
        "cs": "number or string"
      }
    },
    "uart": {
      "available": "boolean",
      "baud_rates": ["number array"],
      "pins": {
        "tx": "number or string",
        "rx": "number or string"
      }
    },
    "wireless": {
      "wifi": {
        "available": "boolean",
        "standards": ["string array (e.g., 802.11b/g/n)"]
      },
      "bluetooth": {
        "available": "boolean",
        "version": "string (e.g., BLE 5.0)"
      }
    }
  },

  "pins": {
    "total_gpio": "number",
    "analog_input": "number",
    "pwm_output": "number",
    "digital_io": "number",
    "pin_definitions": [
      {
        "pin_number": "number or string",
        "pin_name": "string",
        "capabilities": ["string array (e.g., GPIO, ADC, PWM, I2C_SDA)"],
        "voltage_tolerance": "number (volts)"
      }
    ]
  },

  "programming": {
    "supported_languages": ["string array (e.g., CircuitPython, MicroPython, Arduino, C++)"],
    "upload_protocols": ["string array (e.g., USB, UART, OTA)"],
    "bootloader": "string"
  },

  "connectors": {
    "stemma_qt": "boolean",
    "qwiic": "boolean",
    "grove": "boolean",
    "custom": ["string array"]
  },

  "metadata": {
    "datasheet_url": "string (URL)",
    "schema_version": "string (semver)",
    "registration_timestamp": "ISO 8601 datetime",
    "last_updated": "ISO 8601 datetime"
  }
}
```

### 1.2 Dynamic Attributes

```json
{
  "device_id": "string (references static schema)",
  "timestamp": "ISO 8601 datetime",

  "operational_status": {
    "power_state": "enum (on, off, sleep, deep_sleep)",
    "uptime": "number (seconds)",
    "connection_status": "enum (connected, disconnected, error)",
    "last_heartbeat": "ISO 8601 datetime"
  },

  "resource_usage": {
    "memory": {
      "flash_used": "number (bytes)",
      "flash_free": "number (bytes)",
      "flash_usage_percent": "number (0-100)",
      "ram_used": "number (bytes)",
      "ram_free": "number (bytes)",
      "ram_usage_percent": "number (0-100)"
    },
    "cpu": {
      "utilization_percent": "number (0-100)",
      "temperature": "number (celsius, optional)"
    }
  },

  "connected_devices": {
    "sensor_count": "number",
    "sensor_ids": ["string array (UUIDs)"],
    "actuator_count": "number",
    "actuator_ids": ["string array (UUIDs)"],
    "device_status_summary": [
      {
        "device_id": "string (UUID)",
        "device_type": "string",
        "status": "enum (operational, error, disconnected)"
      }
    ]
  },

  "network_status": {
    "wifi": {
      "connected": "boolean",
      "ssid": "string",
      "signal_strength": "number (dBm)",
      "ip_address": "string"
    },
    "bluetooth": {
      "active": "boolean",
      "paired_devices": "number"
    }
  },

  "errors": {
    "error_count": "number",
    "last_error": {
      "timestamp": "ISO 8601 datetime",
      "error_code": "string",
      "error_message": "string",
      "severity": "enum (info, warning, error, critical)"
    }
  }
}
```

## 2. Sensor Hardware Schema

### 2.1 Static Attributes

```json
{
  "device_type": "sensor",
  "device_id": "string (auto-assigned UUID)",
  "manufacturer": "string",
  "model": "string",
  "sensor_category": "enum (environmental, motion, position, proximity, biometric, other)",

  "power": {
    "input_voltage_min": "number (volts)",
    "input_voltage_max": "number (volts)",
    "operating_voltage": "number (volts)",
    "current_consumption_idle": "number (mA)",
    "current_consumption_active": "number (mA)",
    "power_management": {
      "supports_sleep": "boolean",
      "supports_power_gating": "boolean"
    }
  },

  "communication": {
    "primary_protocol": "enum (I2C, SPI, ANALOG, UART, PWM)",
    "i2c": {
      "available": "boolean",
      "address": "string (hex, e.g., 0x47)",
      "address_configurable": "boolean",
      "max_speed": "number (Hz)"
    },
    "spi": {
      "available": "boolean",
      "max_speed": "number (Hz)",
      "mode": "number (0-3)"
    },
    "analog": {
      "available": "boolean",
      "output_type": "enum (voltage, current)",
      "voltage_range_min": "number (volts)",
      "voltage_range_max": "number (volts)",
      "resolution_bits": "number"
    },
    "pwm": {
      "available": "boolean",
      "frequency": "number (Hz)",
      "duty_cycle_range": "object {min: number, max: number}"
    }
  },

  "measurements": [
    {
      "measurement_type": "string (e.g., temperature, pressure, humidity, angle, moisture)",
      "unit": "string (e.g., celsius, pascal, percent, degrees)",
      "range_min": "number",
      "range_max": "number",
      "resolution": "number",
      "accuracy": "number (±)",
      "precision": "number",
      "sampling_rate_max": "number (Hz)",
      "data_format": "string (e.g., int16, float32, uint12)"
    }
  ],

  "pins": {
    "pin_count": "number",
    "pin_definitions": [
      {
        "pin_number": "number or string",
        "pin_name": "string (e.g., VIN, GND, SCL, SDA, OUT)",
        "pin_type": "enum (power, ground, data, clock, interrupt, control)",
        "required": "boolean"
      }
    ]
  },

  "configuration_options": {
    "oversampling": {
      "available": "boolean",
      "options": ["string array (e.g., 1X, 2X, 4X, 8X, 16X, 32X, 64X, 128X)"]
    },
    "filters": {
      "available": "boolean",
      "types": ["string array (e.g., IIR, FIR)"]
    },
    "output_data_rate": {
      "configurable": "boolean",
      "rates": ["number array (Hz)"]
    },
    "power_modes": {
      "available": "boolean",
      "modes": ["string array (e.g., normal, low_power, ultra_low_power, forced)"]
    },
    "interrupts": {
      "available": "boolean",
      "trigger_types": ["string array (e.g., threshold, data_ready)"]
    },
    "calibration": {
      "required": "boolean",
      "auto_calibration": "boolean",
      "calibration_procedure": "string (description or URL)"
    }
  },

  "connectors": {
    "stemma_qt": "boolean",
    "qwiic": "boolean",
    "jst_sh": "boolean",
    "grove": "boolean",
    "custom": ["string array"]
  },

  "physical": {
    "sensing_method": "string (e.g., MEMS, resistive, capacitive, magnetic)",
    "sensing_element": "string (e.g., diaphragm, prongs, Hall effect)",
    "mounting_requirements": "string",
    "environmental_protection": "string (e.g., IP rating)"
  },

  "metadata": {
    "datasheet_url": "string (URL)",
    "library_support": {
      "circuitpython": "boolean",
      "micropython": "boolean",
      "arduino": "boolean",
      "python": "boolean"
    },
    "schema_version": "string (semver)",
    "registration_timestamp": "ISO 8601 datetime",
    "last_updated": "ISO 8601 datetime"
  }
}
```

### 2.2 Dynamic Attributes

```json
{
  "device_id": "string (references static schema)",
  "timestamp": "ISO 8601 datetime",

  "operational_status": {
    "power_state": "enum (on, off, sleep, low_power)",
    "connection_status": "enum (connected, disconnected, error)",
    "last_heartbeat": "ISO 8601 datetime",
    "initialization_status": "enum (initialized, calibrating, ready, error)"
  },

  "live_readings": [
    {
      "measurement_type": "string (matches static schema)",
      "value": "number",
      "unit": "string",
      "timestamp": "ISO 8601 datetime",
      "quality": "enum (good, fair, poor, invalid)",
      "confidence": "number (0-100, optional)"
    }
  ],

  "configuration_state": {
    "current_oversampling": "string",
    "current_filter": "string",
    "current_output_rate": "number (Hz)",
    "current_power_mode": "string",
    "interrupt_enabled": "boolean",
    "custom_settings": "object (key-value pairs)"
  },

  "statistics": {
    "readings_count": "number",
    "last_reading_timestamp": "ISO 8601 datetime",
    "average_reading_interval": "number (seconds)",
    "missed_readings": "number"
  },

  "health": {
    "calibration_status": "enum (calibrated, needs_calibration, calibrating)",
    "last_calibration": "ISO 8601 datetime",
    "error_count": "number",
    "last_error": {
      "timestamp": "ISO 8601 datetime",
      "error_code": "string",
      "error_message": "string",
      "severity": "enum (info, warning, error, critical)"
    }
  },

  "host_mcu": {
    "mcu_id": "string (UUID of connected MCU)",
    "connection_type": "string (e.g., I2C, SPI, ANALOG)",
    "physical_port": "string (e.g., I2C_0, SPI_1, A0)"
  }
}
```

## 3. Schema Validation Examples

### 3.1 BMP580 Temperature/Pressure Sensor (from datasheet)

**Static Schema Instance:**
```json
{
  "device_type": "sensor",
  "device_id": "550e8400-e29b-41d4-a716-446655440001",
  "manufacturer": "Bosch",
  "model": "BMP580",
  "sensor_category": "environmental",
  "power": {
    "input_voltage_min": 3.0,
    "input_voltage_max": 5.0,
    "operating_voltage": 3.3,
    "current_consumption_idle": 0.8,
    "current_consumption_active": 2.5
  },
  "communication": {
    "primary_protocol": "I2C",
    "i2c": {
      "available": true,
      "address": "0x47",
      "address_configurable": false,
      "max_speed": 3400000
    },
    "spi": {
      "available": true,
      "max_speed": 10000000,
      "mode": 0
    }
  },
  "measurements": [
    {
      "measurement_type": "temperature",
      "unit": "celsius",
      "range_min": -40,
      "range_max": 85,
      "resolution": 0.01,
      "accuracy": 0.5,
      "precision": 0.01,
      "sampling_rate_max": 240,
      "data_format": "float32"
    },
    {
      "measurement_type": "pressure",
      "unit": "pascal",
      "range_min": 30000,
      "range_max": 125000,
      "resolution": 0.01,
      "accuracy": 50,
      "precision": 0.01,
      "sampling_rate_max": 240,
      "data_format": "float32"
    }
  ],
  "pins": {
    "pin_count": 8,
    "pin_definitions": [
      {"pin_number": 1, "pin_name": "VIN", "pin_type": "power", "required": true},
      {"pin_number": 2, "pin_name": "3Vo", "pin_type": "power", "required": false},
      {"pin_number": 3, "pin_name": "GND", "pin_type": "ground", "required": true},
      {"pin_number": 4, "pin_name": "SCL", "pin_type": "clock", "required": true},
      {"pin_number": 5, "pin_name": "SDA", "pin_type": "data", "required": true},
      {"pin_number": 6, "pin_name": "SDO", "pin_type": "data", "required": false},
      {"pin_number": 7, "pin_name": "CS", "pin_type": "control", "required": false},
      {"pin_number": 8, "pin_name": "INT", "pin_type": "interrupt", "required": false}
    ]
  },
  "configuration_options": {
    "oversampling": {
      "available": true,
      "options": ["1X", "2X", "4X", "8X", "16X", "32X", "64X", "128X"]
    },
    "filters": {
      "available": true,
      "types": ["IIR"]
    },
    "output_data_rate": {
      "configurable": true,
      "rates": [0.125, 0.25, 0.5, 1, 2, 5, 10, 20, 40, 80, 160, 240]
    },
    "power_modes": {
      "available": true,
      "modes": ["normal", "low_power", "forced"]
    }
  },
  "connectors": {
    "stemma_qt": true,
    "qwiic": true
  },
  "metadata": {
    "datasheet_url": "https://www.adafruit.com/product/5808",
    "library_support": {
      "circuitpython": true,
      "micropython": true,
      "arduino": true,
      "python": true
    }
  }
}
```

### 3.2 AS5600 Magnetic Angle Sensor (from datasheet)

**Static Schema Instance:**
```json
{
  "device_type": "sensor",
  "device_id": "550e8400-e29b-41d4-a716-446655440002",
  "manufacturer": "AMS",
  "model": "AS5600",
  "sensor_category": "position",
  "power": {
    "input_voltage_min": 3.0,
    "input_voltage_max": 5.0,
    "operating_voltage": 3.3
  },
  "communication": {
    "primary_protocol": "I2C",
    "i2c": {
      "available": true,
      "address": "0x36",
      "address_configurable": false,
      "max_speed": 1000000
    },
    "analog": {
      "available": true,
      "output_type": "voltage",
      "voltage_range_min": 0,
      "voltage_range_max": 3.3,
      "resolution_bits": 12
    },
    "pwm": {
      "available": true,
      "frequency": 1000,
      "duty_cycle_range": {"min": 0, "max": 100}
    }
  },
  "measurements": [
    {
      "measurement_type": "angle",
      "unit": "degrees",
      "range_min": 0,
      "range_max": 359,
      "resolution": 0.087,
      "accuracy": 0.4,
      "precision": 0.1,
      "sampling_rate_max": 1000,
      "data_format": "uint12"
    }
  ],
  "pins": {
    "pin_count": 7,
    "pin_definitions": [
      {"pin_number": 1, "pin_name": "VIN", "pin_type": "power", "required": true},
      {"pin_number": 2, "pin_name": "3Vo", "pin_type": "power", "required": false},
      {"pin_number": 3, "pin_name": "GND", "pin_type": "ground", "required": true},
      {"pin_number": 4, "pin_name": "SCL", "pin_type": "clock", "required": true},
      {"pin_number": 5, "pin_name": "SDA", "pin_type": "data", "required": true},
      {"pin_number": 6, "pin_name": "OUT", "pin_type": "data", "required": false},
      {"pin_number": 7, "pin_name": "DIR", "pin_type": "control", "required": false}
    ]
  },
  "configuration_options": {
    "calibration": {
      "required": false,
      "auto_calibration": true,
      "calibration_procedure": "Use PGO pin to store zero position and max angle"
    }
  },
  "connectors": {
    "stemma_qt": true,
    "qwiic": true
  },
  "physical": {
    "sensing_method": "magnetic",
    "sensing_element": "Hall effect",
    "mounting_requirements": "Magnet must be 3mm or less from sensor surface"
  }
}
```

### 3.3 Simple Soil Moisture Sensor (from datasheet)

**Static Schema Instance:**
```json
{
  "device_type": "sensor",
  "device_id": "550e8400-e29b-41d4-a716-446655440003",
  "manufacturer": "Adafruit",
  "model": "4026",
  "sensor_category": "environmental",
  "power": {
    "input_voltage_min": 3.0,
    "input_voltage_max": 5.0,
    "operating_voltage": 3.3,
    "power_management": {
      "supports_power_gating": true
    }
  },
  "communication": {
    "primary_protocol": "ANALOG",
    "analog": {
      "available": true,
      "output_type": "voltage",
      "voltage_range_min": 0,
      "voltage_range_max": 3.3,
      "resolution_bits": 10
    },
    "i2c": {
      "available": true,
      "address": "0x36",
      "address_configurable": false
    }
  },
  "measurements": [
    {
      "measurement_type": "moisture",
      "unit": "percent",
      "range_min": 0,
      "range_max": 100,
      "data_format": "uint16"
    }
  ],
  "pins": {
    "pin_count": 3,
    "pin_definitions": [
      {"pin_number": 1, "pin_name": "VIN", "pin_type": "power", "required": true},
      {"pin_number": 2, "pin_name": "GND", "pin_type": "ground", "required": true},
      {"pin_number": 3, "pin_name": "OUT", "pin_type": "data", "required": true}
    ]
  },
  "connectors": {
    "jst_sh": true
  },
  "physical": {
    "sensing_method": "resistive",
    "sensing_element": "prongs",
    "environmental_protection": "Gold-plated prongs reduce oxidation"
  }
}
```

## 4. Schema Versioning

The schema follows semantic versioning (semver):
- **Major version**: Breaking changes to schema structure
- **Minor version**: Backward-compatible additions
- **Patch version**: Backward-compatible bug fixes

Current schema version: **1.0.0**

## 5. Implementation Notes

1. **Device Discovery**: When a new device connects, the server queries its static attributes and assigns a UUID
2. **Dynamic Updates**: Dynamic attributes are polled/pushed at configurable intervals (default: 1-60 seconds)
3. **Database Storage**: Static schemas stored in versioned table; dynamic attributes in time-series table
4. **Validation**: All schema instances validated against JSON Schema definitions before storage
5. **Extensibility**: Custom fields allowed in "custom_settings" or "metadata" sections