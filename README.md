# 🛰️ IoT Binary Message Protocol Specification

## 🧭 Overview

This document defines a compact, extensible binary protocol used by IoT end devices to transmit sensor data.
The format supports robust error correction using BCH(1023,1003), optional fields like GPS and timestamp, and multi-frame messages for large sensor sets.

## 🧩 Protocol Version

**Version:** 0x01

---

## 📦 Frame Types

The maximum frame size is **125 bytes** of data plus **3 bytes** of ECC.
Primary frames and continuation frames are transmitted together, typically in the same TCP/MQTT package, frames exist only to fit the ECC block size.

### 1. Primary Frame

Used for all messages. Includes the **full header**.

### 2. Continuation Frame

Used when sensor data does not fit in a single frame. Contains only sensor blocks and a minimal header.

> If the primary frame's `Flags` indicates more frames are needed (`Flags & 0x01 = 1`), then one or more continuation frames follow.

---

## 🚩 Flags

Boolean flags are bits of the **Flags** field in frame header. ( 1 = True / Enabled and 0 = False / Disabled)

| Bit | Mask | Usage                                                                                       |
|-----|------|---------------------------------------------------------------------------------------------|
| 0   | 0x01 | Continuation bit: additional frame follows the current frame                                |
| 1   | 0x02 | Enable **MCU Type** field in Primary Frame Header, no effect in Continuation Frame          |
| 2   | 0x04 | Enable **MCU Serial** field in Primary Frame Header, no effect in Continuation Frame        |
| 3   | 0x08 | Enable **Firmware Version** field in Primary Frame Header, no effect in Continuation Frame  |
| 4   | 0x10 | Enable **Serial** field for Sensor Block(s) in the same Frame                               |
| 5   | 0x20 | Reserved                                                                                    |
| 6   | 0x40 | Reserved                                                                                    |
| 7   | 0x80 | Reserved                                                                                    |

---

## 📦 Primary Frame Header Structure

| Field               | Size (Bytes) | Description                                                         |
|---------------------|--------------|---------------------------------------------------------------------|
| **Magic Bytes**     | 2            | ASCII "SN" (0x53 0x4E)                                              |
| **Version**         | 1            | Protocol version (0x01)                                             |
| **Flags**           | 1            | Bit 0 = Continuation Flag, other bits: see Flags description below  |
| **Message ID**      | 4            | 32-bit message sequence ID (rolls over)                             |
| **Location ID**     | 6            | Unique 48-bit deployment ID                                         |
| **Sensor Count**    | 1            | Number of sensor blocks in this frame                               |
| **MCU Type**        | 1            | **optional** Device type (e.g. 1 = ESP32, 2 = STM32)                |
| **MCU Serial**      | 12           | **optional** Unique device serial (from MAC/UID)                    |
| **Firmware Version**| 2            | **optional** Major.Minor firmware version                           |

---

## 📦 Continuation Frame Header Structure

If the **primary frame** sets the `Continuation bit` (bit 0 of the `Flags` byte) to 1 (true), one or more continuation frames must follow. Each continuation frame has the minimal header:

| Field            | Size (Bytes) | Description                                                   |
|------------------|--------------|---------------------------------------------------------------|
| **Flags**        | 1            | Bit 0 = Continuation Flag; 1 if another continuation, else 0  |
| **Sensor Count** | 1            | Number of sensor blocks in this continuation frame            |

> **Magic Bytes**, **Version**, and **Message ID** are **not** included in continuation frames.
> After the header, the specified number of sensor blocks follow.
> Each continuation frame ends with a **3-byte ECC** after the sensor blocks.

---

## 🔢 Sensor Block Format

After the header comes at least one sensor block.

| Field           | Size        | Description                                                                  |
|-----------------|-------------|------------------------------------------------------------------------------|
| **Sensor Type** | 2 bytes     | Sensor type ID (see registry below)                                          |
| **Value Count** | 1 byte      | Number of float32 values (includes serial if real sensor)                    |
| **Serial**      | 8 bytes     | **optional** Sensor unique serial number / ID, (header flag enables it)      |
| **Values**      | N * 4 bytes | float32 array. First = serial (if real), subsequent = measured or meta data  |

> The optional **Serial** field contains the sensor's unique serial number / ID in 8 bytes.
> Padded with zeroes if the ID is shorter than 8 byte, all zeroes if no ID is applicable for the sensor.

---

## 🧩 ECC (Error Correction Code)

| Field   | Size (Bytes) | Description                                                                   |
|---------|--------------|-------------------------------------------------------------------------------|
| **ECC** | 3            | 22-bit **BCH(1023,1003)** for 125-byte payload (corrects up to 2-bit errors)  |

---

## ⚡ Transmitting Short Frames without Padding

By default, each frame is conceptually 125 bytes of data (padded if needed) plus 3 bytes of ECC.
However, for messages smaller than 125 bytes, you can **omit sending the zero-padding** to reduce bandwidth:

1. **ECC Calculation**: You still pad the payload up to 125 bytes internally, then calculate the 22-bit BCH ECC.
2. **Transmission**: Send only the actual data bytes plus 3 ECC bytes (no padding bytes).
   - For instance, if your payload is 40 bytes, you send 40 + 3 = 43 bytes.
3. **Reception**: The receiver reconstructs the 125-byte buffer by appending zero-padding at the end (up to 125 bytes), then verifies/corrects using the ECC.

To do this:
- The receiver must know exactly how many bytes of **real** payload were sent before the ECC. (The last 3 bytes is the ECC, the rest is the real payload.)
- You can also deduce real payload "data length" from `Sensor Count` field and optional value `Flags` bits in the header, and the `Values` fields in sensor blocks.
- Internally, both sides treat it as a 125-byte block with trailing zeroes.

---

## 🔁 Multi-Frame Behavior

- If `Flags & 0x01` in a frame header is set, one or more continuation frames must follow.
- All frames share the same `Message ID` in the primary frame.
- Each frame has its own `Flags`, `Sensor Count`, sensor blocks, and ECC.
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

- **Location ID:** 2^48 = ~281 trillion locations
- **Message ID** 2^32 = 4 294 967 296 (rolls over)
- **MCU Serial:** 2^96 (UID / left zero padded MAC address)
- **Sensor Type IDs:** 2^16 = 65 536
- **Timestamp Range:** 64-bit nanoseconds (covers until year 2262)

---

## ⚙️ Padding and Alignment

- If payload < 125 bytes, zero-padding is applied before ECC generation.
- ECC is always calculated over exactly 125 bytes.

---

## 📜 Encoding Notes

- All multibyte integers are **big-endian**.
- All floating-point values are IEEE 754 **float32**.
- Battery voltage, timestamps and GPS use virtual sensor blocks.
- Timestamp is represented as two float32 values encoding a float64 UNIX timestamp.

---

## 🧮 Value Ranges and Precision

All values are encoded as IEEE 754 **float32** (4 bytes each).

| Value Type         | Range             | Precision                                                         |
|--------------------|-------------------|-------------------------------------------------------------------|
| Temperature (°C)   | -50 to +150       | ~0.001°C                                                          |
| Humidity (%RH)     | 0 to 100          | ~0.001%                                                           |
| Pressure (hPa)     | 300 to 1100       | ~0.01 hPa                                                         |
| GPS Latitude/Long. | ±90 / ±180        | ~±0.000001 (~1 cm)                                                |
| GPS Altitude (m)   | -500 to +10,000   | ~0.001 m                                                          |
| Timestamp (ns)     | 0 to ~year 2262   | Nanosecond precision using float64 (split into 2 float32 values)  |

> Float32 offers about **6–7 significant digits** of precision, which is sufficient for environmental sensor data, high-resolution GPS, and time measurements.

---

## 🏷️ Sensor Type Registry

| Sensor Type | Description        | Values | Notes                                  |
|-------------|--------------------|--------|----------------------------------------|
| 0           | Battery Voltage    | 1      | Virtual/metadata                       |
| 1           | Timestamp          | 2      | Virtual/metadata                       |
| 2           | GPS Position (3D)  | 3      | Virtual/metadata                       |
| 3           | BME280             | 3      | Real: Temperature, Humidity, Pressure  |
| 4           | SHT41              | 2      | Real: Temperature, Humidity            |
| 5           | AHT20              | 2      | Real: Temperature, Humidity            |
| 6           | TMP117             | 1      | Real: Temperature only                 |

> The `Values` value does not contain the optional Sensor Serial / ID field.
> If the `Serial` flag (flag bit 4, 0x10 mask) is true (1), then additional 8 bytes of Serial/ID is inserted between Continuation Frame Headers and Sensor Blocks.

### 🆕 Adding Sensor Types

- **Real sensor types**: assign the next available ID ≥ 7
- **Virtual/metadata sensor types**: assign a unique ID and document its layout
- All sensor types must be globally unique and documented

> Sensor type IDs are 2 bytes (uint16) and allow up to 65536 distinct sensors or metadata types.

---

## 📡 Example Virtual Sensors

Virtual sensors do not have unique serial number or ID.

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

- Value Count: 3 (optional Serial not counted)
- Values:
  - Serial: 0x0000000010000001
    - (No official ID, possible to generate "ID" by hashing "unique" device specific 42 bytes calibration data)
  - Temp: 22.50 °C
  - Humidity: 45.3 %RH
  - Pressure: 1012.6 hPa

```hex
00 03 03
00 00 00 00   # serial (optional)
10 00 00 01   # serial (optional)
41 B4 00 00   # temperature
42 34 66 66   # humidity
44 7E 66 66   # pressure
```

### SHT41 (Sensor Type: 4)

- Value Count: 2 (optional Serial not counted)
- Values:
  - Serial: 0x0000000010000002
  - Temp: 21.8 °C
  - Humidity: 48.5 %RH

```hex
00 04 02
00 00 00 00   # serial (optional)
10 00 00 02   # serial (optional)
41 AE 66 66   # temperature
42 38 00 00   # humidity
```

### AHT20 (Sensor Type: 5)

- Value Count: 2 (optional Serial not counted)
- Values:
  - Serial: 0x0000000000000000 (no serial / ID implemented)
  - Temp: 22.1 °C
  - Humidity: 46.7 %RH

```hex
00 05 03
00 00 00 00   # serial (optional)
00 00 00 00   # serial (optional)
41 B0 CC CD   # temperature
42 3A CC CD   # humidity
```

### TMP117 (Sensor Type: 6)

- Value Count: 1 (optional Serial not counted)
- Values:
  - Serial: 0x0000000010000003
  - Temp: 23.25 °C

```hex
00 06 02
00 00 00 00   # serial (optional)
10 00 00 03   # serial (optional)
41 BA 00 00   # temperature
```

---

## 🧬 Error Correction with BCH(1023,1003)

The protocol uses **BCH(1023,1003)** forward error correction to ensure reliable delivery of binary sensor data over potentially noisy or lossy links.

---

### Properties

| Property             | Value                                                  |
|----------------------|--------------------------------------------------------|
| Codeword length `n` | 1023 bits (127.875 bytes)                               |
| Message length `k`  | 1003 bits (125.375 bytes)                               |
| ECC bits            | 20 bits (~2.5 bytes, aligned to 3 bytes)                |
| Error correction    | Up to **2-bit correction**, detects up to 4-bit errors  |
| Field size          | GF(2<sup>10</sup>)                                      |
| Overhead            | ~1.95%                                                  |

---

### Encoding & Decoding Flow

```text
 ┌─────────────────────────────┐
 │   1003-bit Message Payload  │
 └─────────────────────────────┘
                 │
                 ▼
      Pad to 125.375 bytes if needed
                 │
                 ▼
     ┌──────────────────────────────┐
     │  BCH Encode (t=2, m=10)      │
     └──────────────────────────────┘
                 │
                 ▼
┌──────────────────────────────────────────┐
│ 1023-bit Codeword = Message + 20-bit ECC │
└──────────────────────────────────────────┘
                 │
                 ▼
         Transmit over MQTT
                 │
                 ▼
     ┌──────────────────────────────┐
     │  BCH Decode + Error Correct  │
     └──────────────────────────────┘
                 │
                 ▼
       Recovered 1003-bit Message
```

---

### Encoding and Decoding

- Payloads up to **125 bytes** (1000 bits) are padded to **1003 bits**.
- The **20-bit ECC** is computed using BCH encoding and stored as the final **3 bytes** of the frame.
- On reception:
  - If **1–2 bits** are flipped due to noise, they are corrected.
  - If **3–4 bits** are flipped, errors are detected but uncorrectable.
  - If more than 4 errors: frame is **discarded or logged**.

---

### Why BCH(1023,1003)?

- ✅ **Minimal overhead** (~2%)
- ✅ **Strong protection** in noisy environments
- ✅ Efficient **software implementations** on ESP32 / STM32
- ✅ **Supported by Linux kernel** (`lib/bch.c`) and various embedded toolkits

> Each frame is protected independently, enabling partial recovery from lost or corrupted frames.

---

### BCH Implementation in the Linux kernel `lib/bch.c`

A universal Bose-Chaudhuri-Hocquenghem (BCH) solution is included in the Linux kernel code, published under GNU General Public License version 2 at:

https://github.com/torvalds/linux/blob/master/lib/bch.c

---

### Detailed BCH Algorithm

A typical implementation flow for **BCH(1023,1003)** is:

1. **Message Preparation**
   - Gather up to **125 bytes** of payload.
   - Add up to 3 bits of **zero padding** to reach 1003 bits.

2. **Galois Field Setup**
   - Use **GF(2<sup>10</sup>)**, where `m = 10`, `n = 2^m - 1 = 1023`.

3. **Polynomial Representation**
   - Generator polynomial `G(x)` is chosen to correct up to **2-bit errors**.

4. **Encoding**
   - Represent message as `M(x)`.
   - Multiply by `x^(n-k)`, divide by `G(x)`, take remainder as **ECC**.
   - Append ECC to message to get full codeword `C(x)`.

5. **Transmission**
   - Send `C(x)` (message + 3-byte ECC) over the wire (e.g., via MQTT).

6. **Decoding**
   - Receive possibly corrupted `C'(x)`.
   - Compute **syndromes** to detect errors.
   - If syndromes = 0 → no error.
   - If syndromes ≠ 0 → attempt to locate and correct up to 2 errors.
   - Return corrected message `M(x)`.

> Libraries (or hardware accelerators) encapsulate these steps. The application just provides and receives message data.

---

## 🚀 Future Protocol Extension Possibilities

- Add sensor value type flags (e.g., float64, int32)
- Optional compression of payload
- Message chaining with Message ID + Segment Index

---

## 🧪 Example Messages

### 📦 Example 1 – Single Frame: BME280 + SHT41 (Minimal length message, no optional fields/values)

#### Description

Optional fields/values omitted:
- In Primary Frame Header
  - MCU Type
  - MCU Serial
  - Firmware Version
- Optional Virtual Sensor Values
  - Battery Voltage
  - GPS
  - Timestamp
- Sensor serial number / ID

**Sensors:**
- BME280 (Sensor Type: 3)
- SHT41 (Sensor Type: 4)

#### Example Data

**Frame Header:**
```
53 4E              # Magic "SN"
01                 # Version
00 00 00 01        # Message ID = 1
00                 # Flags (no continuation or optional fields/values)
01 02 03 04 05 06  # Location ID
02                 # Sensor Count = 2
```

Frame Header size: 15 bytes

**Sensor Blocks:**
```
00 03 03          # BME280: type=3, count=3
41 B4 00 00       # Temp = 22.5
42 34 66 66       # Hum = 45.3
44 7E 66 66       # Press = 1012.6

00 04 02          # SHT41: type=4, count=2
41 AE 66 66       # Temp = 21.8
42 38 00 00       # Hum = 48.5
```

Sensor Blocks size: 26 bytes

**ECC:**
```
XX XX XX          # 3-byte BCH ECC
```

ECC size: 3 Bytes

Whole message (and frame) size: 15 + 26 + 3 = 44 bytes

---

### 📦 Example 2 – Two Frames: Battery Voltage, Timestamp, GPS, BME280, SHT41, AHT20, TMP117, + Additional

📌 Here, the first frame includes **6 sensor blocks**, aiming for up to 125 bytes of payload, and the second frame carries 3 more sensor blocks.

#### Description

Optional fields/values included:
- In Primary Frame Header
  - MCU Type
  - MCU Serial
  - Firmware Version
- Optional Virtual Sensor Values
  - Battery Voltage
  - GPS
  - Timestamp
- Sensor serial number / ID

**Sensors:**
- BME280 (Sensor Type: 3)
- SHT41 (Sensor Type: 4)
- AHT20 (Sensor Type: 5)
- TMP117 (Sensor Type: 6)
- placeholder (Sensor Type: 7)
- placeholder (Sensor Type: 8)
- placeholder (Sensor Type: 9)

#### Example Data

**Frame 1 (Primary)**
```
53 4E               # Magic "SN"
01                  # Version
00 00 00 02         # Message ID = 2
1F                  # Flags ('0001 1111' continuation follows, all optional fields/values enabled)
01 02 03 04 05 06   # Location ID
02                  # MCU Type = STM32
12 34 56 78 90 12   # MCU Serial
34 56 78 90 12 34   # MCU Serial
00 02               # Firmware v0.2
06                  # Sensor Count = 6
```

Frame 1 Header size: 30 bytes

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

00 03 03          # BME280: type=3, count=3
00 00 00 00       # serial (optional)
10 00 00 01       # serial (optional)
41 B4 00 00       # 22.5
42 34 66 66       # 45.3
44 7E 66 66       # 1012.6

00 04 02          # SHT41: type=4, count=2
00 00 00 00       # serial (optional)
10 00 00 02       # serial (optional)
41 AE 66 66       # 21.8
42 38 00 00       # 48.5

00 05 02          # AHT20: type=5, count=2
00 00 00 00       # serial (optional)
00 00 00 00       # serial (optional)
41 B0 CC CD       # 22.1
42 3A CC CD       # 46.7
```

Frame 1 Sensor Blocks size: 92 bytes

**ECC:**
```
XX XX XX          # 3-byte BCH ECC
```

Frame 1 ECC size: 3 Bytes

Whole Frame 1 size: 30 + 92 + 3 = 125 bytes

---

**Frame 2 (Continuation)**
```
10                # Flags=0 ('0001 0000' no continuation, all optional fields/values enabled)
02                # Sensor Count=4
```

Frame 2 Header size: 2 bytes

**Sensor Blocks:**
```
00 06 01          # TMP117: type=6, count=1
00 00 00 00       # serial (optional)
10 00 00 03       # serial (optional)
41 BA 00 00       # 23.25

00 07 03          # Placeholder (type=7?), count=3
00 00 00 00       # serial (optional)
10 00 00 07       # serial (optional)
40 00 00 00       # val1=2.0
40 40 00 00       # val2=3.0
40 80 00 00       # val3=4.0

00 08 01          # Another placeholder (type=8?), count=1
00 00 00 00       # serial (optional)
10 00 00 08       # serial (optional)
40 00 00 00       # val1=2.0

00 09 03          # Another placeholder (type=9?), count=3
00 00 00 00       # serial (optional)
10 00 00 09       # serial (optional)
40 10 00 00       # val1=2.25
40 60 00 00       # val2=3.5
40 90 00 00       # val3=4.5
```

Frame 1 Sensor Blocks size: 76 bytes

**ECC:**
```
XX XX XX          # 3-byte BCH ECC
```

Frame 2 ECC size: 3 Bytes

Whole Frame 2 size: 2 + 76 + 3 = 81 bytes

Whole message size: 125 + 81 = 206 bytes
