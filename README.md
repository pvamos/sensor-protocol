# 🛰️ IoT Binary Message Protocol Specification

## 🧭 Overview
This document defines a compact, extensible binary protocol used by IoT end devices to transmit sensor data. The format supports robust error correction using BCH(1023,1001), optional fields like GPS and timestamp, and multi-frame messages for large sensor sets.

## 🧩 Protocol Version
- **Version:** 0x01

---

## 📦 Frame Types
### 1. Primary Frame
Used for all messages. Includes full header.

### 2. Continuation Frame
Used when sensor data does not fit in a single frame. Contains only sensor blocks and minimal header.

---

## 🔢 Frame Structure
### Common to All Frames
| Field        | Size (Bytes) | Description                                |
|-------------|--------------|--------------------------------------------|
| Magic Bytes  | 3            | ASCII "ESN" (0x45 0x53 0x4E)               |
| Version      | 1            | Protocol version (0x01)                   |
| Message ID   | 2            | 16-bit message sequence ID                |
| Flags        | 1            | Bit 0: Continuation Flag                  |

### Primary Frame Additional Fields
| Field            | Size (Bytes) | Description                                    |
|------------------|--------------|------------------------------------------------|
| Location ID      | 5            | Unique 40-bit deployment ID                   |
| MCU Type         | 1            | Device type (e.g., 1 = ESP32, 2 = STM32)       |
| MCU Serial       | 4            | Unique device serial (from MAC/UID)           |
| Firmware Version | 2            | Major.Minor                                   |
| Sensor Count     | 1            | Number of sensor blocks in this frame         |

### Continuation Frame Additional Fields
| Field        | Size (Bytes) | Description                                 |
|-------------|--------------|---------------------------------------------|
| Sensor Count | 1            | Number of sensor blocks in this frame      |

### Sensor Block Format
| Field        | Size        | Description                                                                 |
|-------------|------------|-----------------------------------------------------------------------------|
| Sensor Type  | 2 bytes     | Sensor type ID (see registry below)                                        |
| Value Count  | 1 byte      | Number of float32 values (includes serial if real sensor)                   |
| Values       | N * 4 bytes | float32 array. First = serial (if real), subsequent = measured or meta data |

### ECC (Error Correction Code)
| Field | Size (Bytes) | Description                                  |
|------|--------------|----------------------------------------------|
| ECC  | 3            | 22-bit BCH(1023,1001) for 125-byte payload   |

---

## 🧮 Value Ranges and Precision
All values are encoded as IEEE 754 **float32** (4 bytes per value).

| Value Type         | Range                        | Precision                                                        |
|--------------------|------------------------------|------------------------------------------------------------------|
| Temperature (°C)   | -50 to +150                  | ~0.001°C                                                         |
| Humidity (%RH)     | 0 to 100                     | ~0.001%                                                          |
| Pressure (hPa)     | 300 to 1100                  | ~0.01 hPa                                                        |
| GPS Latitude/Long. | ±90 / ±180                   | ~±0.000001 (~1 cm)                                              |
| GPS Altitude (m)   | -500 to +10,000              | ~0.001 m                                                         |
| Timestamp (ns)     | 0 to ~year 2262              | Nanosecond precision using float64 (split into 2 float32 values) |

Float32 offers about **6–7 significant digits** of precision, which is sufficient for environmental sensor data, high-resolution GPS, and time measurements.

---

## 🧬 Error Correction with BCH(1023,1001)

The protocol uses **BCH(1023,1001)** forward error correction for reliability.

### Properties
- **Data length:** 1001 bits (~125 bytes)
- **Code length:** 1023 bits (~128 bytes)
- **ECC length:** 22 bits (~3 bytes)
- **Error correction capability:**
  - Detects up to 4-bit errors
  - Corrects up to **2-bit errors** per frame

### Encoding and Decoding
- The 125-byte payload (header + sensor data) is zero-padded to 1001 bits.
- A 22-bit ECC is calculated and stored as the final 3 bytes of the frame.
- On decode, if 1–2 bits are flipped due to noise/interference, the message can be automatically corrected.
- Frames with more than 2 errors are discarded or logged.

### Why BCH(1023,1001)?
- Minimal overhead (~2.4%)
- Strong correction capability for low-bandwidth environments
- Efficient software/hardware implementations exist for ESP32 and STM32

Each frame is protected independently, ensuring that multi-frame messages can be partially reconstructed or corrected.

### Detailed BCH Algorithm

A typical implementation flow for **BCH(1023,1001)** is:
1. **Message Preparation**:  Collect up to 125 bytes (1000 bits) of payload + 1 padding bit to make 1001.
2. **Galois Field Setup**:   The code is defined over GF(2^m), typically m=10 for length 1023.
3. **Polynomial Representation**: The generator polynomial G(x) is chosen to correct up to 2 bits.
4. **Encoding**:
   - Represent the payload bits as a polynomial M(x).
   - Multiply M(x) by x^(n−k), then divide by G(x). The remainder is the ECC.
   - Append the ECC (22 bits) to form the final codeword C(x).
5. **Transmission**: Send the 125-byte data + 3-byte ECC over MQTT.
6. **Decoding**:
   - Receive codeword C'(x). If no errors, C'(x) ≈ C(x).
   - Compute syndrome S(x) = C'(x) mod G(x).
   - If S(x)=0, no errors. If not zero, attempt up to 2-bit error correction.
   - Corrected codeword yields the original message M(x).

This process is typically wrapped in library calls for hardware-accelerated or bitwise-optimized code on ESP32/STM32. The user application only needs to pass the 125 bytes for encoding/decoding.

---

## ⚡ Transmitting Short Frames without Padding

By default, each frame is conceptually 125 bytes of data (padded if needed) plus 3 bytes of ECC. However, for small messages, you can **omit sending the zero-padding** to reduce bandwidth:

1. **ECC Calculation**: You still pad the payload up to 125 bytes internally, then calculate the 22-bit BCH ECC.
2. **Transmission**: Send only the actual data bytes plus 3 ECC bytes (no padding bytes).
   - For instance, if your payload is 40 bytes, you send 40 + 3 = 43 bytes.
3. **Reception**: The receiver reconstructs the 125-byte buffer by appending zero-padding at the end (up to 125 bytes), then verifies/corrects using the ECC.

To do this correctly:
- The receiver must know exactly how many bytes of **real** payload were sent before the ECC.
- You can store a "data length" field in the header or deduce it from sensor blocks.
- Internally, both sides treat it as a 125-byte block with trailing zeroes.

This optimization maintains BCH(1023,1001) protection while lowering the transmitted size for short frames.

---

## 🏷️ Sensor Type Registry

| Sensor Type | Description                | Notes                                                 |
|------------:|----------------------------|-------------------------------------------------------|
| 0           | Battery Voltage            | Virtual/metadata                                      |
| 1           | Timestamp                  | Virtual/metadata                                      |
| 2           | GPS Position (3D)          | Virtual/metadata                                      |
| 3           | BME280                     | Real: Temperature, Humidity, Pressure                |
| 4           | SHT41                      | Real: Temperature, Humidity                          |
| 5           | AHT20                      | Real: Temperature, Humidity                          |
| 6           | TMP117                     | Real: Temperature only                               |

### 🆕 Adding Sensor Types
- **Real sensor types**: assign the next available ID ≥ 7
- **Virtual/metadata sensor types**: assign a unique ID and document its layout
- All sensor types must be globally unique and documented

> Sensor type IDs are 2 bytes (uint16) and allow up to 65536 distinct sensors or metadata types.

---

## 📡 Example Virtual Sensors

### Battery Voltage Block (Sensor Type: 0)
- Value Count: 1
- Values: float32 (Battery voltage in Volts)

Example:
- Voltage: 3.76 V

```hex
00 00 01
40 70 66 66   # 3.76 (approx)
```

### Timestamp Block (Sensor Type: 1)
- Value Count: 2
- Values: float32 (high and low parts of UNIX timestamp in nanoseconds)

Example:
- Timestamp: 1700000000123456789 ns (2023-11-14T00:13:20Z)
- Float64 split into hi/lo float32s:

```hex
00 01 02
5E 6B D3 00   # hi part
49 96 02 2B   # lo part
```

### GPS Coordinates Block (Sensor Type: 2)
- Value Count: 3
- Values: float32 (latitude, longitude, altitude)

Example:
- Latitude: 48.1351
- Longitude: 11.5820
- Altitude: 519.0 m

```hex
00 02 03
42 20 AE D0   # lat = 48.1351
41 5A 60 3D   # lon = 11.5820
44 03 00 00   # alt = 519.0
```

---

## 🌡️ Example Sensor Blocks
### BME280 (Sensor Type: 3)
- Value Count: 4
- Values:
  - Serial: 0x10000001
  - Temp: 22.50 °C
  - Humidity: 45.3 %RH
  - Pressure: 1012.6 hPa

```hex
00 03 04
10 00 00 01
41 B4 00 00   # temp
42 34 66 66   # humidity
44 7E 66 66   # pressure
```

### SHT41 (Sensor Type: 4)
- Value Count: 3
- Values:
  - Serial: 0x10000002
  - Temp: 21.8 °C
  - Humidity: 48.5 %RH

```hex
00 04 03
10 00 00 02
41 AE 66 66   # temp
42 38 00 00   # humidity
```

### AHT20 (Sensor Type: 5)
- Value Count: 3
- Values:
  - Serial: 0x10000004
  - Temp: 22.1 °C
  - Humidity: 46.7 %RH

```hex
00 05 03
10 00 00 04
41 B0 CC CD   # temp
42 3A CC CD   # humidity
```

### TMP117 (Sensor Type: 6)
- Value Count: 2
- Values:
  - Serial: 0x10000003
  - Temp: 23.25 °C

```hex
00 06 02
10 00 00 03
41 BA 00 00   # temp
```

---

## 🔁 Multi-Frame Behavior
- If `Flags & 0x01` in a frame is set, one or more continuation frames must follow.
- All frames must share the same `Message ID`.
- Each frame has its own `Sensor Count`, sensor blocks, and ECC.
- Frames are processed sequentially and assembled into a single message.
- **Typically, fill the primary frame up to 125 bytes of payload before starting a continuation frame.**

---

## 📊 Max Sensors Per Frame
Depends on sensor type:

- 1-value real sensor block: ~11 bytes → up to 11 sensors/frame
- 3-value real sensor block: ~19 bytes → up to 6 sensors/frame
- GPS block: 15 bytes → fits easily
- Timestamp block: 11 bytes

Maximum payload per frame: **125 bytes**, followed by **3-byte ECC**.

---

## 🌐 Total Theoretical Device Capacity
- **Location ID:** 2^40 = 1.1 trillion locations
- **MCU Serial:** 2^32 per location
- **Sensor Type IDs:** 2^16
- **Timestamp Range:** 64-bit nanoseconds (covers until year 2262)

---

## ⚙️ Padding and Alignment
- If payload < 125 bytes, zero-padding is applied before ECC generation.
- ECC is always calculated over exactly 125 bytes.

---

## 📜 Encoding Notes
- All multibyte integers are **big-endian**.
- All floating-point values are IEEE 754 **float32**.
- GPS and timestamps must use virtual sensor blocks.
- Timestamp is represented as two float32 values encoding a float64 UNIX timestamp.

---

## 🚀 Future Extensions
- Add sensor value type flags (e.g., float64, int32)
- Optional compression in continuation frames
- Message chaining with Message ID + Segment Index

---

## 🧪 Example Messages

### 🔹 Example 1 – Single Frame: BME280 + SHT41 (No GPS or Timestamp)

**Sensors:**
- BME280 (Sensor Type: 3)
- SHT41 (Sensor Type: 4)

**Frame Header:**
```
45 53 4E          # Magic "ESN"
01                # Version
00 01             # Message ID = 1
00                # Flags (no continuation)
01 02 03 04 05    # Location ID
01                # MCU Type = ESP32
DE AD BE EF       # MCU Serial
00 01             # Firmware v0.1
02                # Sensor Count = 2
```

**Sensor Blocks:**
```
00 03 04          # BME280: type=3, count=4
10 00 00 01       # Serial = 0x10000001
41 B4 00 00       # Temp = 22.5
42 34 66 66       # Hum = 45.3
44 7E 66 66       # Press = 1012.6

00 04 03          # SHT41: type=4, count=3
10 00 00 02       # Serial = 0x10000002
41 AE 66 66       # Temp = 21.8
42 38 00 00       # Hum = 48.5
```

**ECC:**
```
XX XX XX          # 3-byte BCH ECC
```

---

### 🔸 Example 2 – Two Frames: Battery Voltage, Timestamp, GPS, BME280, SHT41, AHT20, TMP117, + Additional

**Frame 1 (Primary)**
```
45 53 4E          # Magic "ESN"
01                # Version
00 02             # Message ID = 2
01                # Flags (continuation follows)
01 02 03 04 05    # Location ID
02                # MCU Type = STM32
12 34 56 78       # MCU Serial
00 02             # Firmware v0.2
08                # Sensor Count = 8
```

**Sensor Blocks (fills primary frame up until 125 bytes)**
```
00 00 01          # Battery Voltage: type=0, count=1
40 70 66 66       # ~3.76 V

00 01 02          # Timestamp: type=1, count=2
5E 6B D3 00       # hi part
49 96 02 2B       # lo part

00 02 03          # GPS: type=2, count=3
42 20 AE D0       # lat=48.1351
41 5A 60 3D       # lon=11.5820
44 03 00 00       # alt=519.0

00 03 04          # BME280: type=3, count=4
10 00 00 01       # Serial=0x10000001
41 B4 00 00       # 22.5
42 34 66 66       # 45.3
44 7E 66 66       # 1012.6

00 04 03          # SHT41: type=4, count=3
10 00 00 02       # Serial=0x10000002
41 AE 66 66       # 21.8
42 38 00 00       # 48.5

00 05 03          # AHT20: type=5, count=3
10 00 00 04       # Serial=0x10000004
41 B0 CC CD       # 22.1
42 3A CC CD       # 46.7

00 06 02          # TMP117: type=6, count=2
10 00 00 03       # Serial=0x10000003
41 BA 00 00       # 23.25
```

**ECC:**
```
XX XX XX
```

---

**Frame 2 (Continuation)**
```
45 53 4E          # Magic "ESN"
01                # Version
00 02             # Message ID = 2
00                # Flags=0 (no more continuation)
02                # Sensor Count=2
```

**Sensor Blocks:**
```
00 07 04          # Placeholder (type=7?), count=4
10 00 00 07       # Serial=0x10000007
40 00 00 00       # val1=2.0
40 40 00 00       # val2=3.0
40 80 00 00       # val3=4.0

00 08 02          # Another placeholder (type=8?), count=2
10 00 00 08       # Serial=0x10000008
40 00 00 00       # val1=2.0

00 09 04          # Another placeholder (type=9?), count=4
10 00 00 09       # Serial=0x10000009
40 10 00 00       # val1=2.25
40 60 00 00       # val2=3.5
40 90 00 00       # val3=4.5
```

**ECC:**
```
XX XX XX
```

📌 Here, the first frame includes **8 sensor blocks**, aiming for ~125 bytes of payload, and the second frame carries 2 more placeholder blocks.

---

> These examples demonstrate how data can be encoded in a compact, extensible binary structure, with one nearly-full primary frame and a second continuation frame, all protected by BCH error correction.

