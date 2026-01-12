# Bookoo Mini Scale Bluetooth Protocol Documentation
 
**Author: Tyler Swensen**

**Target Scale:** Bookoo Mini Scale (Should work with other Bookoo scales)

---

This is everything I discovered about the Bookoo Mini Scale's Bluetooth protocol through manual packet sniffing, along with some algorithms I use to calculate my data. There's no official documentation from Bookoo, so I had to reverse-engineer all of this. Hopefully, this saves you the hours I spent figuring it out!

---

### Device Discovery

The Bookoo Mini Scale advertises with a name containing "bookoo_sc". When scanning for BLE devices, filter for:
- `bookoo_sc` (case-insensitive)

### Connection

```
Service UUID:       0FFE
Weight Char UUID:   FF11  (Notify & subscribe for weight updates)
Command Char UUID:  FF12  (Write - send commands)
```

---

## Bluetooth Service & Characteristics

### Service UUID

```
00000ffe-0000-1000-8000-00805f9b34fb
```

Short form: `0FFE`

### Characteristics

| Characteristic | UUID | Direction | Description |
|---------------|------|-----------|-------------|
| Weight Notifications | `0000ff11-0000-1000-8000-00805f9b34fb` | Notify (Subscribe) | Streams weight data |
| Commands | `0000ff12-0000-1000-8000-00805f9b34fb` | Write | Send tare, timer commands |

---

## Weight Data Packet Format

The scale sends **20-byte packets** via notifications on `FF11`:

### Byte Layout

```
Byte:    [0-5]     [6]     [7]      [8]     [9]     [10-12]   [13]      [14-19]
         ─────────────────────────────────────────────────────────────────────────
         header    sign    hi       mid     low     reserved  battery   reserved
```

### Detailed

| Byte | Name | Value/Description |
|------|------|-------------------|
| 0-5 | Header/Reserved | Packet header data |
| 6 | Sign | `45` (0x2D, ASCII '-') = negative weight, otherwise positive |
| 7 | Weight High | High byte of weight value |
| 8 | Weight Mid | Middle byte of weight value |
| 9 | Weight Low | Low byte of weight value |
| 10-12 | Reserved | Other data |
| 13 | Battery | Battery level percentage |
| 14-19 | Reserved | Other data |

### Weight Parsing Algorithm

```python
def parse_weight(data: bytes) -> float | None:
    # returns weight in grams, or None if invalid packet.
    if len(data) != 20:  # validate
        return None
    
    # extract sign from byte 6 (0x2D = '-' = negative)
    sign = -1 if data[6] == 0x2D else 1
    
    # combine weight bytes (big-endian)
    weight_raw = (data[7] << 16) | (data[8] << 8) | data[9]
    
    weight_grams = sign * (weight_raw / 100.0)
    
    return weight_grams

def parse_battery(data: bytes) -> int | None:
    if len(data) != 20:
        return None
    return data[13]
```

### e.g. packets

| Bytes 6-9 | Weight |
|-----------|--------|
| `00 00 00 00` | 0.00g |
| `00 00 07 08` | 18.00g |
| `00 00 0E 10` | 36.00g |
| `2D 00 00 64` | -1.00g |

---

## Command Payloads

All commands are 6 bytes written to `FF12` characteristic.

### Command Format

```
Byte:    [0]    [1]    [2]    [3]    [4]    [5]
         ────────────────────────────────────────
         0x03   0x0A   CMD    0x00   0x00   CHK

Where:
  - 0x03 = Header byte 1
  - 0x0A = Header byte 2
  - CMD  = Command code
  - 0x00, 0x00 = Fixed payload bytes
  - CHK  = Checksum/verification byte
```

### Available Commands

| Command | Hex Bytes | Description |
|---------|-----------|-------------|
| **Tare** | `03 0A 01 00 00 08` | Zero the scale |
| **Timer Start** | `03 0A 04 00 00 0A` | Start the on-scale timer |
| **Timer Stop** | `03 0A 05 00 00 0D` | Stop/pause the timer |
| **Timer Reset** | `03 0A 06 00 00 0C` | Reset timer to 00:00 |
| **Tare + Auto Timer** | `03 0A 07 00 00 00` | Tare and enter auto-timer mode |

### Python e.g.

```python
# constants
CMD_TARE            = bytes([0x03, 0x0A, 0x01, 0x00, 0x00, 0x08])
CMD_TIMER_START     = bytes([0x03, 0x0A, 0x04, 0x00, 0x00, 0x0A])
CMD_TIMER_STOP      = bytes([0x03, 0x0A, 0x05, 0x00, 0x00, 0x0D])
CMD_TIMER_RESET     = bytes([0x03, 0x0A, 0x06, 0x00, 0x00, 0x0C])
CMD_TARE_AUTO_TIMER = bytes([0x03, 0x0A, 0x07, 0x00, 0x00, 0x00])

async def tare_scale(client):
    await client.write_gatt_char(COMMAND_UUID, CMD_TARE)

async def start_timer(client):
    await client.write_gatt_char(COMMAND_UUID, CMD_TIMER_START)
    
async def stop_timer(client):
    await client.write_gatt_char(COMMAND_UUID, CMD_TIMER_STOP)

async def reset_timer(client):
    await client.write_gatt_char(COMMAND_UUID, CMD_TIMER_RESET)

async def tare_and_auto_timer(client):
    await client.write_gatt_char(COMMAND_UUID, CMD_TARE_AUTO_TIMER)
```

### TypeScript e.g.

```typescript
const CMD_TARE = new Uint8Array([0x03, 0x0A, 0x01, 0x00, 0x00, 0x08]);
const CMD_TIMER_START = new Uint8Array([0x03, 0x0A, 0x04, 0x00, 0x00, 0x0A]);
const CMD_TIMER_STOP = new Uint8Array([0x03, 0x0A, 0x05, 0x00, 0x00, 0x0D]);
const CMD_TIMER_RESET = new Uint8Array([0x03, 0x0A, 0x06, 0x00, 0x00, 0x0C]);
const CMD_TARE_AUTO_TIMER = new Uint8Array([0x03, 0x0A, 0x07, 0x00, 0x00, 0x00]);

async function sendCommand(characteristic: BluetoothRemoteGATTCharacteristic, cmd: Uint8Array) {
  await characteristic.writeValue(cmd);
}
```

---
