# ESP32 VE.Direct Proxy

ESP32-based serial-to-WiFi bridge for Victron Energy VE.Direct protocol devices (solar charge controllers, battery monitors, inverters).

## Features

- Supports 3 VE.Direct devices simultaneously (UART0, UART1, UART2)
- Reads VE.Direct protocol from serial ports (19200 baud)
- Exposes real-time data via HTTP REST API
- Publishes data to MQTT broker (TLS/SSL support)
- Pushes metrics to VictoriaMetrics/Prometheus
- OTA updates via Arduino IDE or espota.py
- Continuous data parsing and caching for all devices
- JSON response format
- Compatible with Victron MPPT solar controllers, BMV battery monitors, and other VE.Direct devices

## Hardware Setup

### Wiring

Connect up to 3 Victron devices to ESP32:

```
Device 1 (UART1):
Victron VE.Direct Port    ESP32
------------------------  -----
GND (Pin 1)          ->   GND
RX  (Pin 2)          ->   GPIO 26 (TX1) D26
TX  (Pin 3)          ->   GPIO 25 (RX1) D25

Device 2 (UART2):
Victron VE.Direct Port    ESP32
------------------------  -----
GND (Pin 1)          ->   GND
RX  (Pin 2)          ->   GPIO 17 (TX2) D17
TX  (Pin 3)          ->   GPIO 16 (RX2) D16

Device 3 (UART0):
Victron VE.Direct Port    ESP32
------------------------  -----
GND (Pin 1)          ->   GND
RX  (Pin 2)          ->   GPIO 14 (TX0) D14
TX  (Pin 3)          ->   GPIO 27 (RX0) D27
```

**Note:** All pins are available on standard ESP32-DevKit boards.

**Note:** VE.Direct uses 5V logic but ESP32 uses 3.3V. Most Victron devices are 3.3V tolerant, but verify your device specifications. Use a level shifter if needed.

### VE.Direct Cable Pinout

Standard VE.Direct cable (JST-PH 4-pin):

1. GND (Black)
2. RX (Blue)
3. TX (Yellow)
4. +5V (Red) - Not used

## Software Setup

### 1. Install Arduino CLI

```bash
# Download and install
curl -fsSL https://raw.githubusercontent.com/arduino/arduino-cli/master/install.sh | sh

# Or on macOS with Homebrew
brew install arduino-cli

# Initialize and install ESP32 platform
arduino-cli config init
arduino-cli config add board_manager.additional_urls https://raw.githubusercontent.com/espressif/arduino-esp32/gh-pages/package_esp32_index.json
arduino-cli core update-index
arduino-cli core install esp32:esp32

# Install required libraries
arduino-cli lib install ArduinoJson
arduino-cli lib install PubSubClient
```

### 2. Configure Settings

Edit `.env` file:

```bash
VERSION=0.0.1
WIFI_SSID=YOUR_WIFI_SSID
WIFI_PASSWORD=YOUR_WIFI_PASSWORD
MQTT_SERVER=mqtt.server
MQTT_PORT=8883
MQTT_USERNAME=username
MQTT_PASSWORD=YOUR_MQTT_PASSWORD
MQTT_TOPIC_PREFIX=username/victron
API_KEY=your-secret-api-key-here
OTA_PASSWORD=your-ota-password-here
VM_SERVER=10.2.1.7
VM_PORT=8428
VM_PATH=/api/v1/import
```

### 3. Build and Upload

```bash
# Build firmware
make build

# Upload via USB (connect ESP32 first)
make upload

# Or upload to specific device
make upload-113  # Upload to 10.2.1.113
make upload-115  # Upload to 10.2.1.115
```

### 4. Get IP Address

Monitor serial output:

```bash
arduino-cli monitor -p /dev/ttyUSB0 -c baudrate=115200
```

## OTA Updates

After initial USB upload, you can update firmware over WiFi:

```bash
# OTA upload to specific device
make ota-113  # Upload to 10.2.1.113
make ota-115  # Upload to 10.2.1.115
```

**Note:** If OTA fails with "Connection reset by peer", reboot the device first:

```bash
curl -X POST http://10.2.1.113/api/reboot -H "X-API-Key: your-secret-api-key-here"
sleep 5
make ota-113
```

**Note:** For reliability, use USB upload instead of OTA.

### OTA Rollback Protection

Firmware uses dual OTA partitions with automatic rollback:

- **Partition Scheme**: "Minimal SPIFFS (1.9MB APP with OTA/190KB SPIFFS)"
- **Two app partitions**: ota_0 and ota_1 (~1.3MB each)
- **Auto-rollback**: If new firmware crashes on boot, ESP32 automatically reverts to previous working version
- **Validation**: Firmware marks itself as valid after successful boot

This prevents bricking the device during failed OTA updates.

## API Usage

### Get Individual Device Status

**GET** `/api/victron/device1` (or device2, device3)

**Response:**

```json
{
  "status": "success",
  "device": "device1",
  "timestamp": 1234567890,
  "data_age_ms": 150,
  "data": {
    "V": 12850,
    "I": -120,
    "VPV": 18500,
    "PPV": 15,
    "CS": 3,
    "MPPT": 2,
    "ERR": 0,
    "LOAD": "ON",
    "IL": 0,
    "H19": 88,
    "H20": 6,
    "H21": 28,
    "H22": 43,
    "H23": 211,
    "HSDS": 16,
    "PID": "0xA060",
    "FW": 164,
    "SER#": "HQ2239YTQ62"
  }
}
```

### Get All Devices Status

**GET** `/api/victron/all`

**Response:**

```json
{
  "status": "success",
  "timestamp": 1234567890,
  "devices": [
    {
      "name": "device1",
      "data_available": true,
      "data_age_ms": 150,
      "data": {
        "V": 12850,
        "I": -120,
        "VPV": 18500
      }
    },
    {
      "name": "device2",
      "data_available": true,
      "data_age_ms": 200,
      "data": {
        "V": 13100,
        "I": 500
      }
    },
    {
      "name": "device3",
      "data_available": false
    }
  ]
}
```

### Reboot Device

**POST** `/api/reboot`

Headers:

```
X-API-Key: your-secret-api-key-here
```

**Response:**

```json
{
  "status": "success",
  "message": "Rebooting..."
}
```

### Health Check

**GET** `/api/health`

**Response:**

```json
{
  "status": "ok",
  "service": "vedirect-proxy",
  "uptime_ms": 123456,
  "num_devices": 3,
  "devices": [
    {"name": "device1", "data_available": true},
    {"name": "device2", "data_available": true},
    {"name": "device3", "data_available": false}
  ]
}
```

## VE.Direct Data Fields

Common fields (varies by device):

- **V** - Battery voltage (mV)
- **I** - Battery current (mA)
- **VPV** - Panel voltage (mV)
- **PPV** - Panel power (W)
- **CS** - State of operation (0=Off, 2=Fault, 3=Bulk, 4=Absorption, 5=Float)
- **MPPT** - Tracker operation mode (0=Off, 1=Limited, 2=Active)
- **ERR** - Error code
- **LOAD** - Load output state (ON/OFF)
- **IL** - Load current (mA)
- **H19** - Yield total (0.01 kWh)
- **H20** - Yield today (0.01 kWh)
- **H21** - Maximum power today (W)
- **H22** - Yield yesterday (0.01 kWh)
- **H23** - Maximum power yesterday (W)
- **HSDS** - Day sequence number
- **PID** - Product ID
- **FW** - Firmware version
- **SER#** - Serial number

See [VE.Direct Protocol documentation](https://www.victronenergy.com/upload/documents/VE.Direct-Protocol-3.33.pdf) for complete field reference.

## MQTT Topics

When MQTT is enabled, data is published to:

- `user/victron/{SER#}` - Device data by serial number

Example: `user/victron/HQ20333V8IF`

Message format (JSON):

```json
{
  "V": 12850,
  "I": -120,
  "VPV": 18500,
  "PPV": 15,
  "CS": 3,
  "MPPT": 2,
  "PID": "0xA060",
  "SER#": "HQ20333V8IF",
  "SER": "HQ20333V8IF"
}
```

## VictoriaMetrics/Prometheus Metrics

When VictoriaMetrics is enabled, metrics are pushed to `http://vm_server:vm_port/api/v1/import`:

**Metrics exported:**

- `victron_load_current` - Load current (mA)
- `victron_yield_total` - Total yield (0.01 kWh)
- `victron_yield_today` - Today's yield (0.01 kWh)
- `victron_maximum_power_today` - Max power today (W)
- `victron_yield_yesterday` - Yesterday's yield (0.01 kWh)
- `victron_maximum_power_yesterday` - Max power yesterday (W)
- `victron_day_sequence_number` - Day sequence
- `victron_battery_voltage` - Battery voltage (mV)
- `victron_battery_current` - Battery current (mA)
- `victron_battery_watt` - Battery power (W)
- `victron_panel_voltage` - Panel voltage (mV)
- `victron_panel_power` - Panel power (W)
- `victron_tracker_operation_mode` - MPPT tracker mode
- `victron_state_of_operation` - Charge state
- `victron_error_code` - Error code
- `victron_load_output_state` - Load output (0/1)

**Labels:**

- `job="victron_esp32"`
- `pid` - Product ID
- `ser` - Serial number
- `firmware` - Firmware version
- `device` - Device name (device1/device2/device3)

## Build Commands

```bash
make build        # Build firmware
make upload       # Upload via USB
make upload-113   # Upload to .113 via USB
make upload-115   # Upload to .115 via USB
make ota-113      # OTA upload to .113
make ota-115      # OTA upload to .115
make version      # Show version
make clean        # Clean build files
```

## Testing

### Using curl

```bash
# Get device 1 status
curl http://ESP32_IP/api/victron/device1

# Get device 2 status
curl http://ESP32_IP/api/victron/device2

# Get all devices status
curl http://ESP32_IP/api/victron/all

# Health check
curl http://ESP32_IP/api/health

# Reboot device
curl -X POST http://ESP32_IP/api/reboot -H "X-API-Key: your-secret-api-key-here"
```

### Using Python

```python
import requests

# Get all devices
response = requests.get('http://ESP32_IP/api/victron/all')
data = response.json()

for device in data['devices']:
    if device['data_available']:
        name = device['name']
        voltage = int(device['data']['V']) / 1000  # mV to volts
        current = int(device['data']['I']) / 1000  # mA to amps
        print(f"{name}: {voltage}V, {current}A")

# Get single device
response = requests.get('http://ESP32_IP/api/victron/device1')
data = response.json()
voltage = int(data['data']['V']) / 1000
print(f"Device 1: {voltage}V")
```

## Troubleshooting

**No data available:**

- Verify VE.Direct cable connections (GND, RX, TX)
- Check serial pins are correct:
  - Device 1: GPIO 25(RX), 26(TX)
  - Device 2: GPIO 16(RX), 17(TX)
  - Device 3: GPIO 27(RX), 14(TX)
- Ensure Victron device is powered on
- Verify baud rate is 19200 (standard for VE.Direct)

**USB Permission Error (Linux):**

```bash
sudo chmod 666 /dev/ttyUSB0
# Or permanent fix:
sudo usermod -a -G dialout $USER
```

**Sketch Too Large Error:**
Partition scheme is set to "Minimal SPIFFS (1.9MB APP with OTA/190KB SPIFFS)" for OTA rollback support

**WiFi connection fails:**

- Verify SSID and password
- ESP32 only supports 2.4GHz WiFi (not 5GHz)
- Check WiFi signal strength

**Incorrect data values:**

- Verify RX/TX are not swapped
- Check for proper ground connection
- Ensure 3.3V/5V logic level compatibility

## Protocol Reference

VE.Direct is a text-based protocol:

- 19200 baud, 8N1
- Tab-delimited key-value pairs
- Terminated with checksum
- Updates every 1 second

Example frame:

```
\r\n
V\t12850\r\n
I\t-120\r\n
VPV\t18500\r\n
PPV\t15\r\n
CS\t3\r\n
Checksum\t<byte>\r\n
```

## Compatible Devices

- MPPT Solar Charge Controllers (75/10, 75/15, 100/15, 100/20, etc.)
- BMV Battery Monitors (BMV-700, BMV-702, BMV-712)
- Phoenix Inverters with VE.Direct port
- Other Victron devices with VE.Direct interface

## References

- [VE.Direct Protocol Specification](https://www.victronenergy.com/upload/documents/VE.Direct-Protocol-3.33.pdf)
- [Victron Energy Documentation](https://www.victronenergy.com/live/)
