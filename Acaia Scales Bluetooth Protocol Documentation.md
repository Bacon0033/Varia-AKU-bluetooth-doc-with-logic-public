# Acaia Scales Bluetooth Protocol Documentation
 
**Author: Tyler Swensen**

**Target Scales:** Acaia Pearl, Pearl S, Lunar, Lunar 2021, Pyxis, Cinco

---

This is everything I discovered about the Acaia scales' Bluetooth protocol through manual packet sniffing and/or documentation written by Acaia. Hopefully, this saves you the hours I spent figuring it out!

---

### Device Discovery

The Acaia scales advertise with names starting with specific prefixes. When scanning for BLE devices, filter for devices whose name starts with:
- `ACAIA`
- `LUNAR`
- `PYXIS`
- `PROCH`
- `PEARL`
- `CINCO`

### Connection

There are two connection modes depending on the scale model:

**Standard Scales (Pearl, Lunar, older models):**
```
Service UUID:       1820
Char UUID:          2A80  (Notify & Write - bidirectional)
```

**Pyxis-Style Scales (Pyxis, newer Lunar 2021):**
```
Service UUID:       49535343-FE7D-4AE5-8FA9-9FAFD205E455
TX Char UUID:       49535343-8841-43F4-A8D4-ECBE34729BB3  (Write - send commands)
RX Char UUID:       49535343-1E4D-4BD9-BA61-23C647249616  (Notify - receive data)
```

---

## Bluetooth Service & Characteristics

### Standard Scale Service UUID

```
00001820-0000-1000-8000-00805f9b34fb
```

Short form: `1820`

### Pyxis-Style Service UUID

```
49535343-FE7D-4AE5-8FA9-9FAFD205E455
```

### Characteristics

**Standard Scales:**

| Characteristic | UUID | Direction | Description |
|---------------|------|-----------|-------------|
| Command/Weight | `00002a80-0000-1000-8000-00805f9b34fb` | Notify + Write | Bidirectional - streams data & accepts commands |

**Pyxis-Style Scales:**

| Characteristic | UUID | Direction | Description |
|---------------|------|-----------|-------------|
| TX (Commands) | `49535343-8841-43F4-A8D4-ECBE34729BB3` | Write | Send commands to scale |
| RX (Notifications) | `49535343-1E4D-4BD9-BA61-23C647249616` | Notify (Subscribe) | Receive weight data & events |

---

## Packet Format

All Acaia packets use a common structure with magic header bytes.

### Magic Header

```
MAGIC1 = 0xEF (239 decimal)
MAGIC2 = 0xDD (221 decimal)
```

### General Packet Structure

```
Byte:    [0]    [1]    [2]     [3]      [4..n]    [n+1]    [n+2]
         ──────────────────────────────────────────────────────────
         0xEF   0xDD   TYPE    LEN      PAYLOAD   CKSUM1   CKSUM2

Where:
  - 0xEF, 0xDD = Magic header bytes
  - TYPE = Message type code
  - LEN = Payload length
  - PAYLOAD = Variable length data
  - CKSUM1 = Sum of even-indexed payload bytes (& 0xFF)
  - CKSUM2 = Sum of odd-indexed payload bytes (& 0xFF)
```

### Message Types

| Type Code | Name | Description |
|-----------|------|-------------|
| 0 | Heartbeat | Keep-alive signal |
| 4 | Tare | Zero the scale |
| 6 | Get Settings | Request scale settings |
| 8 | Settings Response | Scale returns settings |
| 11 | Identity | Handshake/authentication |
| 12 | Event Data | Weight updates, timer events, button presses |
| 13 | Timer Control | Start/stop/reset timer |

---

## Weight Data Packet Format

Weight data arrives in **Event Data (type 12)** messages. The inner message type is at byte 4.

### Scale Message Types (inside Event Data)

| Code | Message Type |
|------|--------------|
| 5 | Weight |
| 7 | Timer |
| 8 | Tare/Start/Stop/Reset button events |
| 11 | Heartbeat response |

### Weight Payload Format

For weight messages (type 5), the payload structure:

| Byte | Name | Description |
|------|------|-------------|
| 0 | Weight Low | Low byte of weight value |
| 1 | Weight High | High byte of weight value |
| 2-3 | Reserved | (varies) |
| 4 | Unit/Precision | Decimal precision indicator |
| 5 | Sign/Flags | Bit 1 = negative weight flag |

### Weight Parsing Algorithm

```
Raw Value = (payload[1] << 8) | payload[0]

Precision divisor based on payload[4]:
  1 = divide by 10
  2 = divide by 100
  3 = divide by 1000
  4 = divide by 10000

Sign: if (payload[5] & 0x02) != 0, weight is negative
```

---

## Command Payloads

All commands follow the packet format: `[0xEF, 0xDD, TYPE, ...PAYLOAD, CKSUM1, CKSUM2]`

### Checksum Calculation

```python
def calculate_checksums(payload: list[int]) -> tuple[int, int]:
    cksum1 = 0
    cksum2 = 0
    for i, val in enumerate(payload):
        if i % 2 == 0:
            cksum1 += val
        else:
            cksum2 += val
    return (cksum1 & 0xFF, cksum2 & 0xFF)
```

### Identity Command (Required for Connection)

Must be sent after connecting to authenticate with the scale.

**Standard Scales:**
```
Type: 11
Payload: [0x2D, 0x2D, 0x2D, 0x2D, 0x2D, 0x2D, 0x2D, 0x2D, 0x2D, 0x2D, 0x2D, 0x2D, 0x2D, 0x2D, 0x2D]
         (15 bytes of 0x2D, which is ASCII '-')
```

**Pyxis-Style Scales:**
```
Type: 11
Payload: [0x30, 0x31, 0x32, 0x33, 0x34, 0x35, 0x36, 0x37, 0x38, 0x39, 0x30, 0x31, 0x32, 0x33, 0x34]
         (ASCII "012345678901234")
```

### Notification Request (Required for Weight Updates)

Must be sent after identity to start receiving weight notifications.

```
Type: 12
Payload: [
    0x09,  // length + 1
    0x00,  // weight notification
    0x01,  // weight argument
    0x01,  // battery notification
    0x02,  // battery argument
    0x02,  // timer notification
    0x05,  // timer argument (heartbeats between timer messages)
    0x03,  // key notification
    0x04   // setting notification
]
```

### Available Commands

| Command | Type | Payload | Full Encoded Bytes |
|---------|------|---------|-------------------|
| **Heartbeat** | 0 | `[0x02, 0x00]` | `EF DD 00 02 00 02 00` |
| **Tare** | 4 | `[0x00]` | `EF DD 04 00 00 00` |
| **Timer Start** | 13 | `[0x00, 0x00]` | `EF DD 0D 00 00 00 00` |
| **Timer Stop** | 13 | `[0x00, 0x02]` | `EF DD 0D 00 02 00 02` |
| **Timer Reset** | 13 | `[0x00, 0x01]` | `EF DD 0D 00 01 00 01` |

### Encoding Function

```python
MAGIC1 = 0xEF
MAGIC2 = 0xDD

def encode(msg_type: int, payload: list[int]) -> bytes:
    cksum1 = 0
    cksum2 = 0
    
    result = bytearray([MAGIC1, MAGIC2, msg_type])
    
    for i, val in enumerate(payload):
        result.append(val & 0xFF)
        if i % 2 == 0:
            cksum1 += val
        else:
            cksum2 += val
    
    result.append(cksum1 & 0xFF)
    result.append(cksum2 & 0xFF)
    
    return bytes(result)

# Commands
CMD_HEARTBEAT = encode(0, [0x02, 0x00])
CMD_TARE = encode(4, [0x00])
CMD_TIMER_START = encode(13, [0x00, 0x00])
CMD_TIMER_STOP = encode(13, [0x00, 0x02])
CMD_TIMER_RESET = encode(13, [0x00, 0x01])
```

---

## Connection Sequence

1. Scan for device with name starting with ACAIA/LUNAR/PYXIS/PROCH/PEARL/CINCO
2. Connect to the device
3. Discover services and characteristics
4. (Android only) Request MTU of 247
5. Subscribe to notifications on the appropriate characteristic
6. Send Identity command
7. Wait ~100ms
8. Send Notification Request command
9. Start heartbeat timer (send heartbeat every ~1000ms to keep connection alive)

---

## Settings Response

When the scale responds with settings (type 8), the payload contains:

| Byte | Description |
|------|-------------|
| 1 | Battery level (& 0x7F for value) |
| 2 | Units: 2 = grams, 5 = ounces |
| 4 | Auto-off setting (multiply by 5 for minutes) |
| 6 | Beep: 1 = on, 0 = off |

---

## Timer Data Format

Timer values are encoded as:

```
Minutes = payload[0]
Seconds = payload[1]
Deciseconds = payload[2]

Total milliseconds = ((minutes * 60) + seconds + (deciseconds / 10)) * 1000
```

---

## Button Events

Tare/Start/Stop/Reset events (message type 8 inside Event Data) are identified by the first two bytes of the payload:

| Bytes [0, 1] | Event | Additional Data |
|--------------|-------|-----------------|
| `0x00, 0x05` | Tare | Weight follows |
| `0x08, 0x05` | Timer Start | Weight follows |
| `0x0A, 0x07` | Timer Stop | Time + weight follows |
| `0x09, 0x07` | Timer Reset | Time + weight follows |

---

