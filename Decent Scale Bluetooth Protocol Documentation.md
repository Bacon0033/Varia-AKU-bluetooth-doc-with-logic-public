# Decent Scale Bluetooth Protocol Documentation
 
**Author: Tyler Swensen**

**Target Scale:** Decent Scale

---

This is everything I discovered about the Decent Scale's Bluetooth protocol through **manual packet sniffing and/or documentation written by Decent**. Hopefully, this saves you the hours I spent figuring it out!

---

### Device Discovery

The Decent Scale advertises with a name starting with "Decent". When scanning for BLE devices, filter for:
- `decent` (case-insensitive, matches start of name)

### Connection

```
Service UUID:       FFF0
Read Char UUID:     FFF4  (Notify & subscribe for weight updates)
Write Char UUID:    36F5  (Write - send commands)
```

**Note:** Requires heartbeat/keep-alive every 2 seconds to maintain connection.

---

## Bluetooth Service & Characteristics

### Service UUID

```
0000fff0-0000-1000-8000-00805f9b34fb
```

Short form: `FFF0`

### Characteristics

| Characteristic | UUID | Direction | Description |
|---------------|------|-----------|-------------|
| Weight Notifications | `0000fff4-0000-1000-8000-00805f9b34fb` | Notify (Subscribe) | Streams weight data & button events |
| Commands | `000036f5-0000-1000-8000-00805f9b34fb` | Write | Send tare, timer, LED commands |

---

## Weight Data Packet Format

The scale sends **7-byte or 10-byte packets** via notifications on `FFF4` (depends on firmware API version):
- API < 1.3: 7-byte packets
- API >= 1.3: 10-byte packets

### Byte Layout (Weight Message)

```
Byte:    [0]    [1]      [2]     [3]     [4-6 or 4-9]
         ───────────────────────────────────────────────
         0x03   type     hi      low     reserved/checksum
```

### Message Types (Byte 1)

| Value | Type | Description |
|-------|------|-------------|
| `0xCE` | Stable Weight | Weight reading is stable |
| `0xCA` | Unstable Weight | Weight reading is still settling |
| `0xAA` | Button Event | Physical button was pressed |

### Weight Parsing

For weight messages (`0xCE` or `0xCA`):
- Bytes 2-3: Signed 16-bit big-endian weight value
- Divide by 10 to get grams

### Button Events (Byte 1 = 0xAA)

| Byte 2 | Event |
|--------|-------|
| `0x01` | Tare button pressed |
| `0x02` | Timer button pressed |

### Weight Parsing Algorithm

```python
def parse_weight(data: bytes) -> tuple[float, bool] | None:
    # returns (weight in grams, is_stable), or None if invalid packet.
    if len(data) < 7:  # validate minimum length
        return None
    
    if data[1] not in (0xCE, 0xCA):  # not a weight update
        return None
    
    is_stable = data[1] == 0xCE
    
    # extract weight as signed 16-bit big-endian
    weight_raw = int.from_bytes(data[2:4], byteorder='big', signed=True)
    
    weight_grams = weight_raw / 10.0
    
    return (weight_grams, is_stable)
```

---

## Command Payloads

All commands are 7 bytes written to `36F5` characteristic.

### Command Format

```
Byte:    [0]    [1]    [2]    [3]    [4]    [5]    [6]
         ─────────────────────────────────────────────────
         0x03   CMD    arg1   arg2   arg3   arg4   XOR

Where:
  - 0x03 = Header byte
  - CMD  = Command code
  - arg1-4 = Command arguments
  - XOR = Checksum (XOR of bytes 0-5)
```

### XOR Checksum Calculation

```python
def calculate_xor(data: bytes) -> int:
    return data[0] ^ data[1] ^ data[2] ^ data[3] ^ data[4] ^ data[5]
```

### Available Commands

| Command | Bytes (before XOR) | Description |
|---------|-------------------|-------------|
| **Tare** | `03 0F FD [counter] 00 01 [XOR]` | Zero the scale (counter increments 0-255) |
| **Timer Start** | `03 0B 03 00 00 00 [XOR]` | Start the on-scale timer |
| **Timer Stop** | `03 0B 00 00 00 00 [XOR]` | Stop/pause the timer |
| **Timer Reset** | `03 0B 02 00 00 00 [XOR]` | Reset timer to 00:00 |
| **LED On (Both)** | `03 0A 01 01 00 00 [XOR]` | Turn on weight & timer LEDs |
| **LED Off (Both)** | `03 0A 00 00 00 00 [XOR]` | Turn off weight & timer LEDs |
| **Keep Alive** | `03 0A 03 FF FF 00 0A` | Heartbeat (send every 2 seconds) |

### Command Codes

| Code | Function |
|------|----------|
| `0x0A` | LED control / Keep alive |
| `0x0B` | Timer control |
| `0x0F` | Tare |

### Python e.g.

```python
def get_xor(data: bytes) -> int:
    return data[0] ^ data[1] ^ data[2] ^ data[3] ^ data[4] ^ data[5]

# Tare command with counter
tare_counter = 0

def build_tare_command() -> bytes:
    global tare_counter
    data = bytes([0x03, 0x0F, 0xFD, tare_counter, 0x00, 0x01])
    cmd = data + bytes([get_xor(data)])
    tare_counter = (tare_counter + 1) % 256
    return cmd

# Timer commands
def build_timer_command(action: str) -> bytes:
    if action == 'start':
        arg = 0x03
    elif action == 'reset':
        arg = 0x02
    else:  # stop
        arg = 0x00
    data = bytes([0x03, 0x0B, arg, 0x00, 0x00, 0x00])
    return data + bytes([get_xor(data)])

# LED control
def build_led_command(weight_on: bool, timer_on: bool) -> bytes:
    data = bytes([0x03, 0x0A, int(weight_on), int(timer_on), 0x00, 0x00])
    return data + bytes([get_xor(data)])

# Keep alive (send every 2 seconds)
CMD_KEEP_ALIVE = bytes([0x03, 0x0A, 0x03, 0xFF, 0xFF, 0x00, 0x0A])

async def tare_scale(client):
    cmd = build_tare_command()
    await client.write_gatt_char(WRITE_UUID, cmd)
    await asyncio.sleep(0.2)
    cmd = build_tare_command()  # Send twice for reliability
    await client.write_gatt_char(WRITE_UUID, cmd)

async def start_timer(client):
    cmd = build_timer_command('start')
    await client.write_gatt_char(WRITE_UUID, cmd)
    
async def stop_timer(client):
    cmd = build_timer_command('stop')
    await client.write_gatt_char(WRITE_UUID, cmd)

async def reset_timer(client):
    cmd = build_timer_command('reset')
    await client.write_gatt_char(WRITE_UUID, cmd)
```

### TypeScript e.g.

```typescript
function getXOR(data: Uint8Array): number {
  return data[0] ^ data[1] ^ data[2] ^ data[3] ^ data[4] ^ data[5];
}

let tareCounter = 0;

function buildTareCommand(): Uint8Array {
  const data = new Uint8Array([0x03, 0x0F, 0xFD, tareCounter, 0x00, 0x01]);
  const cmd = new Uint8Array(7);
  cmd.set(data);
  cmd[6] = getXOR(data);
  tareCounter = (tareCounter + 1) % 256;
  return cmd;
}

function buildTimerCommand(action: 'start' | 'stop' | 'reset'): Uint8Array {
  let arg: number;
  if (action === 'start') arg = 0x03;
  else if (action === 'reset') arg = 0x02;
  else arg = 0x00;
  
  const data = new Uint8Array([0x03, 0x0B, arg, 0x00, 0x00, 0x00]);
  const cmd = new Uint8Array(7);
  cmd.set(data);
  cmd[6] = getXOR(data);
  return cmd;
}

function buildLedCommand(weightOn: boolean, timerOn: boolean): Uint8Array {
  const data = new Uint8Array([0x03, 0x0A, weightOn ? 1 : 0, timerOn ? 1 : 0, 0x00, 0x00]);
  const cmd = new Uint8Array(7);
  cmd.set(data);
  cmd[6] = getXOR(data);
  return cmd;
}

const CMD_KEEP_ALIVE = new Uint8Array([0x03, 0x0A, 0x03, 0xFF, 0xFF, 0x00, 0x0A]);

async function sendCommand(characteristic: BluetoothRemoteGATTCharacteristic, cmd: Uint8Array) {
  await characteristic.writeValue(cmd);
}
```

---

## Connection Sequence

1. Scan for device with name starting with "Decent"
2. Connect to the device
3. Discover services and characteristics
4. Send LED On command to initialize display
5. Subscribe to notifications on `FFF4`
6. Start heartbeat timer - send keep-alive every 2 seconds

**Important:** Commands should be sent twice with ~200ms delay for reliability.

---

