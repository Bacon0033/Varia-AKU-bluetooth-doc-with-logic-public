# Varia AKU Pro Bluetooth Scale Protocol Documentation
 
**Author: Tyler Swensen**

**Target Scale:** Varia AKU Pro (Should work with other Varia AKU scales)

---

This is everything I discovered about the Varia AKU Pro's Bluetooth protocol through manual packet sniffing, along with some algorithms I use to calculate my data. There's no official documentation from Varia, so I had to reverse-engineer all of this. Hopefully, this saves you the hours I spent figuring it out!

---

## Table of Contents

1. [Connection Overview](#device-discovery)
2. [Bluetooth Service & Characteristics](#bluetooth-service--characteristics)
3. [Weight Data Packet Format](#weight-data-packet-format)
4. [Command Payloads](#command-payloads)
5. [Flow Rate Calculation](#flow-rate-calculation)
6. [First Drip Detection](#first-drip-detection)
7. [Pre-Infusion Time](#pre-infusion-time)
8. [Brew Analysis & Scoring](#brew-analysis--scoring)
9. [Cup Placement Detection](#cup-placement-detection)
10. [Data Recording](#data-recording)

---

### Device Discovery

The Varia AKU Pro advertises with names containing "Varia" or "AKU". When scanning for BLE devices, filter for:
- `varia` (case-insensitive)
- `aku` (case-insensitive)
- `scale`

### Connection

```
Service UUID:       FFF0
Weight Char UUID:   FFF1  (Notify & subscribe for weight updates)
Command Char UUID:  FFF2  (Write - send commands)
```

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
| Weight Notifications | `0000fff1-0000-1000-8000-00805f9b34fb` | Notify (Subscribe) | Streams weight data at ~10Hz |
| Commands | `0000fff2-0000-1000-8000-00805f9b34fb` | Write (No Response) | Send tare, timer commands |

---

## Weight Data Packet Format

The scale sends **7-byte packets** via notifications on `FFF1`:

### Byte Layout

```
Byte:    [0]    [1]    [2]    [3]       [4]      [5]      [6]
         ───────────────────────────────────────────────────────
         0xFA   0x01   mode   sign+hi   mid      low      checksum
```

### Detailed

| Byte | Name | Value/Description |
|------|------|-------------------|
| 0 | Header | Always `0xFA` (250 decimal) - start of valid packet |
| 1 | Type | `0x01` = weight update |
| 2 | Mode/Flags | Mode indicator (varies, not fully decoded (also not sure if you can do stuff with the scales mode over BLE, unfortunately. If needed, I could look into it!)) |
| 3 | Sign + High | Bit 4 = sign (0=positive, 1=negative), Lower nibble = high 4 bits of weight |
| 4 | Mid | Middle byte of weight value |
| 5 | Low | Low byte of weight value |
| 6 | Checksum | Packet verification |

### Weight Parsing Algorithm (What I used to test)

```python
def parse_weight(data: bytes) -> float | None:
    # returns weight in grams, or None if invalid packet.
    if len(data) != 7:  # validate
        return None
    if data[1] != 0x01:  # not a weight update
        return None
    
    # extract sign from bit 4 of byte 3
    sign = 1 if (data[3] & 0x10) == 0 else -1
    
    # combine weight bytes
    weight_raw = ((data[3] & 0x0F) << 16) | (data[4] << 8) | data[5]
    
    weight_grams = sign * (weight_raw / 100.0)
    
    return weight_grams
```

### JavaScript/TypeScript Version (What I used in my pi application)

```typescript
function parseWeight(data: Uint8Array): number | null {
  if (data.length !== 7) return null;
  if (data[1] !== 0x01) return null;
  
  const sign = (data[3] & 0x10) === 0 ? 1 : -1;
  const weightRaw = ((data[3] & 0x0F) << 16) | (data[4] << 8) | data[5];
  
  return sign * (weightRaw / 100.0);
}
```

### e.g. packets

| Hex Bytes | Weight |
|-----------|--------|
| `FA 01 00 00 00 00 XX` | 0.00g |
| `FA 01 00 00 07 08 XX` | 18.00g |
| `FA 01 00 00 0E 10 XX` | 36.00g |
| `FA 01 00 10 00 64 XX` | -1.00g |

---

## Command Payloads

All commands are 5 bytes written to `FFF2` characteristic.

### Command Format

```
Byte:    [0]    [1]    [2]    [3]    [4]
         ─────────────────────────────────
         0xFA   CMD    0x01   0x01   CHK

Where:
  - 0xFA = Header
  - CMD  = Command code
  - 0x01, 0x01 = Fixed payload bytes
  - CHK  = Checksum
```

### Available Commands

| Command | Hex Bytes | Description |
|---------|-----------|-------------|
| **Tare** | `FA 82 01 01 82` | Zero the scale |
| **Timer Start** | `FA 88 01 01 88` | Start the on-scale timer |
| **Timer Stop** | `FA 89 01 01 89` | Stop/pause the timer |
| **Timer Reset** | `FA 8A 01 01 89` | Reset timer to 00:00 |

### Python e.g.

```python
# constants
CMD_TARE        = bytes.fromhex('FA82010182')
CMD_TIMER_START = bytes.fromhex('FA88010188')
CMD_TIMER_STOP  = bytes.fromhex('FA89010189')
CMD_TIMER_RESET = bytes.fromhex('FA8A010189')

async def tare_scale(client):
    await client.write_gatt_char(COMMAND_UUID, CMD_TARE, response=False)

async def start_timer(client):
    await client.write_gatt_char(COMMAND_UUID, CMD_TIMER_START, response=False)
    
async def stop_timer(client):
    await client.write_gatt_char(COMMAND_UUID, CMD_TIMER_STOP, response=False)

async def reset_timer(client):
    await client.write_gatt_char(COMMAND_UUID, CMD_TIMER_RESET, response=False)
```

### TypeScript e.g. (Web Bluetooth, also what I ended up using for my project)

```typescript
const CMD_TARE = new Uint8Array([0xFA, 0x82, 0x01, 0x01, 0x82]);
const CMD_TIMER_START = new Uint8Array([0xFA, 0x88, 0x01, 0x01, 0x88]);
const CMD_TIMER_STOP = new Uint8Array([0xFA, 0x89, 0x01, 0x01, 0x89]);
const CMD_TIMER_RESET = new Uint8Array([0xFA, 0x8A, 0x01, 0x01, 0x89]);

async function sendCommand(characteristic: BluetoothRemoteGATTCharacteristic, cmd: Uint8Array) {
  await characteristic.writeValueWithoutResponse(cmd);
}
```

---

## Flow Rate Calculation

### Formula

```
Flow Rate (g/s) = ΔWeight / ΔTime

Where:
  ΔWeight = current_weight - previous_weight
  ΔTime   = (current_timestamp - previous_timestamp) / 1000
```

### Implementation

```typescript
interface WeightReading {
  weight: number;
  timestamp: number;
}

class FlowRateCalculator {
  private lastReading: WeightReading | null = null;
  
  calculateFlowRate(weight: number, timestamp: number): number {
    if (!this.lastReading) {
      this.lastReading = { weight, timestamp };
      return 0;
    }
    
    const weightDelta = weight - this.lastReading.weight;
    const timeDelta = (timestamp - this.lastReading.timestamp) / 1000;
    
    // ignore if time gap is too small or too large
    if (timeDelta <= 0 || timeDelta > 2) {
      this.lastReading = { weight, timestamp };
      return 0;
    }
    
    // flow rate should be non-negative, this makes sure
    const flowRate = Math.max(0, weightDelta / timeDelta);
    
    this.lastReading = { weight, timestamp };
    return flowRate;
  }
}
```

### Typical Flow Rate Values (According to Google)

| Phase | Flow Rate (g/s) | Description |
|-------|-----------------|-------------|
| Pre-infusion | 0 - 0.1 | |
| First drops | 0.1 - 0.5 | starting to emerge |
| Main extraction | 1.0 - 2.5 | Steady flow |
| Channeling | > 3.0 | possible channeling |
| Choking | < 0.5 sustained | puck resistance too high |

---

## First Drip Detection

### Detection Algorithm

I used a multi-condition way to avoid false positives:

```typescript
/* detection parameters (you may need to tune this for your machine, as it is tuned for mine
 (Breville Barista Touch Impress), but will prob be ok unless someone's using a really weird machine lol) */
const FIRST_DRIP_MIN_FLOW = 0.08;       // minimum flow rate (g/s)
const FIRST_DRIP_SUSTAINED_MS = 800;    // flow must sustain durring this time
const FIRST_DRIP_MIN_WEIGHT = 0.3;      // minimum weight increase required
const CUP_PLACEMENT_THRESHOLD = 5.0;    // ignore sudden jumps
```

### Detection Logic

```typescript
class FirstDripDetector {
  private flowStartTime: number | null = null;
  private flowSamples: number[] = [];
  private weightBeforeFlow: number = 0;
  private detected: boolean = false;
  
  check(weight: number, flowRate: number, timestamp: number): {
    detected: boolean;
    time: number | null;
  } {
    if (this.detected) {
      return { detected: true, time: null };
    }
    
    // add flow sample
    this.flowSamples.push(flowRate);
    if (this.flowSamples.length > 5) {
      this.flowSamples.shift();
    }
    
    // calculate average flow
    const avgFlow = this.flowSamples.reduce((a, b) => a + b, 0) / this.flowSamples.length;
    
    if (avgFlow >= FIRST_DRIP_MIN_FLOW) {
      // flow
      if (this.flowStartTime === null) {
        this.flowStartTime = timestamp;
        this.weightBeforeFlow = weight;
      }
      
      // check if sustained long enough
      const flowDuration = timestamp - this.flowStartTime;
      const weightIncrease = weight - this.weightBeforeFlow;
      
      if (flowDuration >= FIRST_DRIP_SUSTAINED_MS && weightIncrease >= FIRST_DRIP_MIN_WEIGHT) {
        // confirmed
        this.detected = true;
        return { detected: true, time: timestamp };
      }
    } else {
      // flow stopped
      this.flowStartTime = null;
      this.flowSamples = [];
    }
    
    return { detected: false, time: null };
  }
  
  reset(): void {
    this.flowStartTime = null;
    this.flowSamples = [];
    this.weightBeforeFlow = 0;
    this.detected = false;
  }
}
```


## Pre-Infusion Time


### Calculation

```typescript
// brew starts
const brewStartTime = Date.now();

// first drip is detected
const firstDripTime = /* from FirstDripDetector */;

// pre-infusion
const preInfusionTime = (firstDripTime - brewStartTime) / 1000;
```


###  Why I track pre-infusion? 

I track it to help calculate the quality score at the end.

---

## Brew Analysis & Scoring

### Brew Ratio

espresso metric

```typescript
const dose = 18;       // coffee in
const yield_ = 36;     // liquid out
const ratio = yield_ / dose;  // = 2.0

//  format to look better "1:2.0"
console.log(`1:${ratio.toFixed(1)}`);
```

### Target Ratios (Google's definition)

| Style | Ratio |
|-------|-------|
| Ristretto | 1:1 - 1:1.5 |
| Normale | 1:1.5 - 1:2 |
| Lungo | 1:2.5 - 1:3+ |

### Automated Scoring Algorithm

scoring system based on hitting targets:

```typescript
interface BrewTargets {
  targetRatio: number;      // e.g., 2.0
  ratioTolerance: number;   // e.g., 0.2 (±10%)
  targetTime: number;       // e.g., 28 seconds
  timeTolerance: number;    // e.g., 5 seconds
}

function scoreBrew(
  dose: number,
  yield_: number,
  brewTime: number,
  targets: BrewTargets
): { score: number; breakdown: object } {
  const actualRatio = yield_ / dose;
  
  // ratio score
  const ratioDiff = Math.abs(actualRatio - targets.targetRatio);
  const ratioScore = Math.max(0, 50 - (ratioDiff / targets.ratioTolerance) * 25);
  
  // time score
  const timeDiff = Math.abs(brewTime - targets.targetTime);
  const timeScore = Math.max(0, 50 - (timeDiff / targets.timeTolerance) * 25);
  
  const totalScore = Math.round(ratioScore + timeScore);
  
  return {
    score: totalScore,
    breakdown: {
      ratioScore: Math.round(ratioScore),
      timeScore: Math.round(timeScore),
      actualRatio: actualRatio.toFixed(2),
      brewTime: brewTime.toFixed(1)
    }
  };
}
```

---

## Cup Placement Detection

Distinguish between coffee dripping and someone placing a cup on the scale.

### Logic

```typescript
const CUP_PLACEMENT_THRESHOLD = 5.0; // g

function isCupPlacement(weightDelta: number): boolean {
  return weightDelta > CUP_PLACEMENT_THRESHOLD;
}

function onWeightUpdate(newWeight: number) {
  const delta = newWeight - previousWeight;
  
  if (isCupPlacement(delta)) {
    console.log("cup placed");
    // don't treat this as flow
  } else {
    // normal weight change
    calculateFlowRate(newWeight);
    checkFirstDrip(newWeight);
  }
}
```

---

## Data Recording

### Data Point Structure

```typescript
interface BrewDataPoint {
  timestamp: number;   // ms since brew start
  weight: number;      // grams
  flowRate: number;    // g/s
}

interface BrewRecord {
  id: number;
  startTime: Date;
  endTime: Date;
  dose: number;
  yield: number;
  brewTime: number;           // seconds
  ratio: number;
  preInfusionTime: number | null;
  firstDripDetected: boolean;
  series: BrewDataPoint[];
  score: number | null;
}
```

### Recording During Brew

```typescript
class BrewRecorder {
  private startTime: number = 0;
  private data: BrewDataPoint[] = [];
  
  start(): void {
    this.startTime = Date.now();
    this.data = [];
  }
  
  record(weight: number, flowRate: number): void {
    this.data.push({
      timestamp: Date.now() - this.startTime,
      weight,
      flowRate
    });
  }
  
  stop(): { elapsed: number; data: BrewDataPoint[] } {
    return {
      elapsed: (Date.now() - this.startTime) / 1000,
      data: [...this.data]
    };
  }
}
```

---

## What You Can Detect & Analyze

Here's a summary of everything possible with this protocol:

| Feature | Description |
|---------|-------------|
| **Real-time Weight** | ~10Hz weight updates |
| **Flow Rate** | Calculated from weight deltas |
| **First Drip** | Detect when coffee starts flowing |
| **Pre-Infusion Time** | Time until first drip |
| **Brew Ratio** | Yield ÷ Dose |
| **Total Brew Time** | Start to stop |
| **Cup Placement** | Sudden weight jump |
| **Auto-Start** | Detect brew start from flow |
| **Time-Series Data** | Full weight/flow curve |

---

## Troubleshooting

### Issues I had

| Problem | Solution |
|---------|----------|
| Erratic flow rate | Add smoothing/averaging to reduce noise |
| Commands don't work | Ensure you're using write without response |

### Platform Notes

- **iOS/Android:** Use native BLE APIs or React Native BLE libraries

---

*This documentation was created through manual Bluetooth packet analysis. Varia has no official documentation for their BLE protocol. All findings are from reverse engineering.*
