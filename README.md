# ESP32 GPS NTP Server Stratum 1

A GPS-disciplined Network Time Protocol (NTP) server implementation for ESP32 microcontrollers. The server derives time directly from a GNSS receiver and uses a hardware Pulse Per Second (PPS) signal for sub-millisecond timestamping, qualifying it as a **Stratum 1** time source as defined by [RFC 5905](https://datatracker.ietf.org/doc/html/rfc5905).

Two firmware variants are provided:

| Variant | Target | Connectivity |
|---|---|---|
| `esp32-stratum1-ntp` | ESP32 (classic) | Wi-Fi |
| `esp32p4-stratum1-ntp` | ESP32-P4 | Ethernet (preferred) + Wi-Fi fallback |

---

## Table of Contents

- [How It Works](#how-it-works)
- [Features](#features)
- [Hardware Requirements](#hardware-requirements)
- [Wiring](#wiring)
- [Software Requirements](#software-requirements)
- [Configuration](#configuration)
- [Flashing](#flashing)
- [Services](#services)
- [NTP Packet Structure](#ntp-packet-structure)
- [Verifying Operation](#verifying-operation)
- [Project Structure](#project-structure)
- [License](#license)

---

## How It Works

```
GPS Receiver (UART, 9600 baud)
        │
        ├─── NMEA sentences ──► TinyGPSPlus parser ──► UTC date/time (1-second resolution)
        │
        └─── PPS signal ──► GPIO interrupt (IRAM_ATTR) ──► microsecond timestamp of each second edge
                                                                │
                                                                ▼
                                              NTP timestamp = GPS epoch seconds
                                                            + micros() elapsed since last PPS edge
                                                            + fractional second (32-bit NTP format)
```

The GPS module continuously outputs NMEA sentences over UART. The `TinyGPSPlus` library parses these to extract UTC date and time, which are converted to Unix epoch seconds and then to NTP epoch (offset by 2,208,988,800 seconds, the number of seconds between 1 January 1900 and 1 January 1970).

Crucially, NMEA time has only one-second resolution. The PPS signal from the GPS module provides a hardware pulse precisely aligned to each UTC second transition. A GPIO interrupt handler (placed in IRAM for deterministic latency) captures the `micros()` counter at each rising PPS edge. All subsequent NTP timestamps are computed as:

```
NTP seconds   = GPS epoch seconds (from NMEA)
              + floor((micros() − lastPPS) / 1,000,000)

NTP fraction  = ((micros() − lastPPS) mod 1,000,000) × 2³²
              / 1,000,000
```

This approach produces timestamps with microsecond-class resolution, bounded by the stability of the ESP32 crystal oscillator between PPS edges.

---

## Features

- **Stratum 1 NTP server** responds to NTP v4 client requests (UDP port 123)
- **PPS-disciplined timestamping** hardware interrupt captures each GPS second edge for high-resolution fractional seconds
- **Raw NMEA TCP stream** serves raw NMEA sentences on TCP port 2947 (compatible with `gpsd` clients such as `gpsmon` or Chrony's `SHM` refclock)
- **Web status interface** auto-refreshing dashboard (port 80) showing GPS fix, satellite count, PPS status, location, and network details
- **JSON status API** machine-readable endpoint at `GET /status`
- **ESP32-P4 variant additionally provides:**
  - Wired Ethernet as the primary network interface
  - Automatic Wi-Fi fallback when Ethernet is unavailable
  - Periodic GPS status logging to the serial console (every 10 seconds)
  - Automatic Wi-Fi reconnection attempt (every 30 seconds)

---

## Hardware Requirements

- **Microcontroller:** ESP32 (any variant with two available UART ports) or ESP32-P4
- **GNSS receiver:** Any module outputting NMEA 0183 sentences at 9600 baud with a 3.3 V-compatible UART and a hardware PPS output (e.g., u-blox NEO-6M, NEO-M8N, or equivalent)
- **Antenna:** Active or passive GPS antenna appropriate for your receiver module

---

## Wiring

### ESP32 (classic)

| Signal | ESP32 GPIO |
|---|---|
| GPS TX → ESP32 RX | GPIO 16 |
| GPS RX → ESP32 TX | GPIO 17 |
| GPS PPS | GPIO 4 |
| GPS VCC | 3.3 V |
| GPS GND | GND |

> The GPS TX→RX and RX→TX labels refer to the direction from the GPS module's perspective. Connect the GPS module's TX pin to the ESP32's RX pin (GPIO 16), and the GPS module's RX pin to the ESP32's TX pin (GPIO 17).

### ESP32-P4

| Signal | ESP32-P4 GPIO |
|---|---|
| GPS TX → ESP32 RX | GPIO 20 |
| GPS RX → ESP32 TX | GPIO 5 |
| GPS PPS | GPIO 21 |
| GPS VCC | 3.3 V |
| GPS GND | GND |

> Ethernet is handled by the on-chip MAC and requires a compatible PHY chip. Refer to your specific ESP32-P4 development board schematic for the Ethernet PHY connections.

---

## Software Requirements

- [Arduino IDE](https://www.arduino.cc/en/software) 2.x or later (or Arduino CLI)
- **ESP32 Arduino core** install via the Arduino Boards Manager:
  - For ESP32 classic: `esp32` by Espressif Systems
  - For ESP32-P4: `esp32` by Espressif Systems (version supporting P4)
- **TinyGPSPlus** library install via the Arduino Library Manager:
  - Search for `TinyGPSPlus` by Mikal Hart

---

## Configuration

Open the relevant `.ino` file and edit the configuration section near the top of the file.

### Wi-Fi credentials (both variants)

```cpp
String ssid = "your-network-name";
String password = "your-network-password";
```

### ESP32-P4: connectivity mode

The ESP32-P4 variant attempts Ethernet first and falls back to Wi-Fi automatically. No additional flag is required. If Ethernet is not present, the firmware proceeds directly to Wi-Fi association.

### Pin assignments

If your hardware uses different GPIO pins, update the `#define` constants:

```cpp
// ESP32 classic
#define GPS_RX_PIN  16
#define GPS_TX_PIN  17
#define PPS_PIN      4

// ESP32-P4
#define GPS_RX_PIN  20
#define GPS_TX_PIN   5
#define PPS_PIN     21
```

### GPS baud rate

```cpp
#define GPS_BAUD 9600   // Change if your module is configured differently
```

---

## Flashing

1. Open the Arduino IDE.
2. Select your board:
   - **Tools → Board → ESP32 Arduino → ESP32 Dev Module** (for classic ESP32)
   - **Tools → Board → ESP32 Arduino → ESP32-P4** (for P4)
3. Select the correct serial port under **Tools → Port**.
4. Open `src/esp32-stratum1-ntp/esp32-stratum1-ntp.ino` or `src/esp32p4-stratum1-ntp/esp32p4-stratum1-ntp.ino`.
5. Click **Upload**.
6. Open the Serial Monitor at **115200 baud** to observe the boot sequence and GPS status output.

---

## Services

Once the device is running and the GPS has a valid fix, the following services are available:

| Service | Protocol | Port | Description |
|---|---|---|---|
| NTP | UDP | 123 | Standard NTP v4 time service |
| NMEA stream | TCP | 2947 | Raw NMEA sentences (one client at a time) |
| Web dashboard | HTTP | 80 | Auto-refreshing status page |
| JSON API | HTTP | 80 | `GET /status` machine-readable status |

### JSON Status Response

`GET http://<device-ip>/status` returns a JSON object with the following fields:

```json
{
  "ip": "192.168.1.100",
  "ntpPort": 123,
  "tcpPort": 2947,
  "timeValid": true,
  "ppsActive": true,
  "satellites": 8,
  "lastPPS": 45,
  "latitude": 48.858800,
  "longitude": 2.294500,
  "altitude": 35.0,
  "tcpConnected": false
}
```

The ESP32-P4 variant additionally includes `connectionType`, `ethConnected`, and `wifiConnected` fields.

---

## NTP Packet Structure

The server constructs NTP v4 response packets per RFC 5905. Key fields:

| Byte(s) | Field | Value | Meaning |
|---|---|---|---|
| 0 | LI/VN/Mode | `0x24` (`0b00100100`) | LI=0 (no leap warning), VN=4, Mode=4 (server) |
| 1 | Stratum | `1` | Stratum 1 direct GPS reference |
| 2 | Poll | `6` | Minimum poll interval (2⁶ = 64 seconds) |
| 3 | Precision | `0xEC` | ~−20 (≈ 1 µs) |
| 16–23 | Reference Timestamp | GPS+PPS | Time of last clock update |
| 24–31 | Originate Timestamp | Echoed from client | Client's transmit time |
| 32–39 | Receive Timestamp | GPS+PPS | Time the request was received |
| 40–47 | Transmit Timestamp | GPS+PPS | Time the response was sent |

---

## Verifying Operation

### Query the NTP server from a Unix host

```bash
ntpdate -q <device-ip>
```

Or using `sntp`:

```bash
sntp <device-ip>
```

### Connect to the NMEA TCP stream

```bash
nc <device-ip> 2947
```

You should see a continuous stream of NMEA sentences such as `$GPRMC`, `$GPGGA`, and `$GPGSV`.

### Use with gpsd

The TCP NMEA stream on port 2947 is compatible with `gpsd`:

```bash
gpsd tcp://<device-ip>:2947
gpsmon
```

### Inspect with Chrony

Add the device as a NTP source in `/etc/chrony.conf`:

```
server <device-ip> iburst prefer
```

Then check synchronisation status:

```bash
chronyc sources -v
chronyc tracking
```

---

## Project Structure

```
esp32-gps-ntp/
├── src/
│   ├── esp32-stratum1-ntp/
│   │   └── esp32-stratum1-ntp.ino     # ESP32 (classic) firmware
│   └── esp32p4-stratum1-ntp/
│       └── esp32p4-stratum1-ntp.ino   # ESP32-P4 firmware (Ethernet + Wi-Fi)
├── LICENSE
└── README.md
```

---

## License

This project is licensed under the **Apache License 2.0**. See [LICENSE](LICENSE) for the full text.
