# MEMS2J Interface — Protocol and Data Formats
All the findings about the protocol and the used data formats. Check my sources at the end of the document for attribution.

## Protocol Overview

| Parameter | Value |
|---|---|
| Protocol | [KWP-2000 (ISO 14230-4)](https://en.wikipedia.org/wiki/Keyword_Protocol_2000)|
| Physical layer | K-Line (ISO 9141), Single-Wire Half-Duplex |
| Baud rate | **10400 baud** |
| Data format | 8 data bits, No Parity, 1 Stop bit (8N1) |
| K-Line voltage | 0V / 12–14.4V |
| Echo | Every transmitted byte is mirrored on RX (must be discarded) |

## Packet Format

### Structure

```raw
┌──────────┬────────────────────────┬──────────┐
│  LENGTH  │     DATA BYTES         │ CHECKSUM │
│  1 Byte  │     N Bytes            │  1 Byte  │
└──────────┴────────────────────────┴──────────┘
```

- **LENGTH**: Number of data bytes (excluding LENGTH byte and CHECKSUM)
- **CHECKSUM**: `(LENGTH + DATA[0] + DATA[1] + ... + DATA[N-1]) & 0xFF`

### Examples

| Command | Data bytes | Full packet |
|---|---|---|
| Ping | `0x3E` | `[0x01][0x3E][0x3F]` |
| StartDiag | `0x10 0xA0` | `[0x02][0x10][0xA0][0xB2]` |
| RequestSeed | `0x27 0x01` | `[0x02][0x27][0x01][0x2A]` |
| DataRequest 0x09 | `0x21 0x09` | `[0x02][0x21][0x09][0x2C]` |


## Connection Sequence

### Phase 1: Fast-Init (K-Line Wake-Up)

```raw
K-Line:  ──────────────____________________──────────────────────
                       ↑ 25ms LOW          ↑ 25ms HIGH   ↑ wait 50ms
                     (Break)           (non-break)
```

TX pin must be controlled as GPIO (not UART) during Fast-Init, then switched back to UART before sending.

### Phase 2: Init Command (RAW — no packet wrapper)

```raw
Send (5 bytes, raw):  0x81  0x13  0xF7  0x81  0x0C
                       ←──── Echo back ─────────────→ (discard)
ECU responds:          0xC1  0xD5  0x8F
```

### Phase 3: Start Diagnostic Session (Service 0x10)

```raw
Send:   [0x02][0x10][0xA0][0xB2]
ECU:    [0x01][0x50][0x51]
```

### Phase 4: Security Access — Seed/Key (Service 0x27)

```raw
Send:   [0x02][0x27][0x01][0x2A]             ← RequestSeed
ECU:    [0x04][0x67][0x01][SH][SL][CS]       ← Seed (2 bytes)

→ Key = generateKey(seed)
  Special case: if seed == 0x0000, key = 0x0000 (no auth required)

Send:   [0x04][0x27][0x02][KH][KL][CS]       ← SendKey
ECU:    [0x02][0x67][0x02][CS]               ← Key accepted
```

### Phase 5: Heartbeat / Ping (Service 0x3E)

```raw
Send:   [0x01][0x3E][0x3F]
ECU:    [0x01][0x7E][0x7F]
```

Send every ~2 seconds when no other commands are active.

## Seed-to-Key Algorithm
```go
func generateKey(seed int) int {
    key := 0
    loops := 1

    if bit(15, seed) > 0 { loops += 8 }
    if bit(7,  seed) > 0 { loops += 4 }
    if bit(4,  seed) > 0 { loops += 2 }
    if bit(0,  seed) > 0 { loops += 1 }

    for loops > 0 {
        key = seed >> 1
        if bit(13, seed) > 0 && bit(3, seed) > 0 {
            key &= 0xFFFE   // clear bit 0
        } else {
            key |= 0x0001   // set bit 0
        }
        xors := bit(9, seed) ^ bit(8, seed) ^ bit(2, seed) ^ bit(1, seed)
        if xors > 0 {
            key |= 0x8000   // set bit 15
        }
        seed = key
        loops--
    }
    return key
}
```
## Service Table

| Service | Code | Description | Request → Response |
|---|---|---|---|
| StartDiagnosticSession | `0x10` | General diagnostic session | `[0x10, 0xA0]` → `[0x50]` |
| StartDiagnosticSession | `0x10` | Boot Loader / programming session | `[0x10, 0x80]` → Reset |
| SecurityAccess | `0x27` | Request seed | `[0x27, 0x01]` → `[0x67, 0x01, SH, SL]` |
| SecurityAccess | `0x27` | Send key | `[0x27, 0x02, KH, KL]` → `[0x67, 0x02]` |
| ReadDataByLocalId | `0x21` | Live data frame | `[0x21, ID]` → `[0x61, ID, data...]` |
| ReadDataByLocalId | `0x21` | Fault codes | `[0x21, 0x19]` → `[0x61, 0x19, faults...]` |
| StartRoutineByLocalId | `0x31` | Clear fault codes | `[0x31, 0xCB, 0x00×15]` → `[0x71, 0xCB]` |
| TesterPresent | `0x3E` | Heartbeat / Ping | `[0x3E]` → `[0x7E]` |

### Negative Response

```raw
[LEN][0x7F][SERVICE_CODE][ERROR_CODE][CS]
```

| Error code | Meaning |
|---|---|
| `0x10` | generalReject / subFunctionNotSupported |
| `0x22` | conditionsNotCorrect |
| `0x35` | invalidKey |
| `0x36` | exceededNumberOfAttempts |

### Clear Fault Codes (full packet)

```raw
Send:  [0x11][0x31][0xCB][0x00][0x00][0x00][0x00][0x00]
       [0x00][0x00][0x00][0x00][0x00][0x00][0x00][0x00]
       [0x00][0x00][CS]
ECU:   [0x02][0x71][0xCB][CS]
```
## Timing Requirements

| Parameter | Value |
|---|---|
| Fast-Init LOW (break) | 25 ms |
| Fast-Init HIGH (non-break) | 25 ms |
| Wait after Fast-Init | 50 ms |
| Additional wait after Wake-Up | 50 ms |
| Inter-packet delay | 25 ms |
| Heartbeat interval | max. ~2 seconds |
| Response timeout | 1000 ms |

## Echo Handling

K-Line is single-wire half-duplex: every transmitted byte is echoed back on RX and must be discarded before reading the ECU response.

```raw
1. Send packet
2. Read and discard echo (same bytes as sent)
3. Read ECU response
```
## Live Data — Service 0x21 Frame Table

### Request / Response Format

```raw
Request:  [0x02][0x21][ID][CS]
Response: [LEN][0x61][ID][byte2][byte3]...[CS]
```

Byte indices below are 0-based within the full response (including `0x61` and frame ID).

### Frame Table

| ID | Field | Bytes | Calculation | Unit | Notes |
|---|---|---|---|---|---|
| `0x00` | *(ignored)* | — | — | — | |
| `0x01` | Coolant temperature | [2:3] uint16 BE | `(val − 2732) / 10` | °C | Kelvin×10 |
| `0x02` | Engine oil temperature | [2:3] uint16 BE | `(val − 2732) / 10` | °C | Kelvin×10 |
| `0x03` | Intake air temperature | [2:3] uint16 BE | `(val − 2732) / 10` | °C | Kelvin×10 |
| `0x05` | Fuel temperature | [2:3] uint16 BE | raw | — | encoding unclear |
| `0x06` | *(ignored)* | — | — | — | |
| `0x07` | MAP sensor | [2:3] uint16 BE | `val / 100` | kPa | |
| `0x08` | Throttle position (TPS) | [2:3] uint16 BE | `val / 100` | degrees | |
| `0x09` | Engine speed | [2:3] uint16 BE | raw | RPM | |
| `0x0A` | Fuelling feedback | [2:3] uint16 BE | `val / 100` | % | |
| | Lambda voltage (O2) | [4:5] uint16 BE | raw | mV | |
| | Estimated AFR | [4:5] | `(val/1000 × 2) + 10` | λ | estimate |
| `0x0B` | Coil 1 charge time | [2] | `val / 1000` | ms | |
| | Coil 2 charge time | [3] | `val / 1000` | ms | |
| `0x0C` | Injector 1 pulse width | [2] | raw | — | |
| | Injector 2 pulse width | [3] | raw | — | |
| | Injector 3 pulse width | [4] | raw | — | |
| | Injector 4 pulse width | [5] | raw | — | |
| `0x0D` | Vehicle speed | [2] | raw | km/h | |
| `0x0F` | Throttle switch | [2] bit 0 | `val & 0x01` | bool | |
| | Ignition | [2] bit 1 | `(val >> 1) & 0x01` | bool | |
| | AC button | [2] bit 3 | `(val >> 3) & 0x01` | bool | |
| `0x10` | Battery voltage | [4:5] uint16 BE | `val / 1000` | V | |
| `0x11` | Primary trigger sync | [2] bit 0 | `1 − (val & 0x01)` | bool | inverted: 0=OK |
| | Secondary trigger sync | [2] bit 1 | `1 − ((val>>1) & 0x01)` | bool | inverted: 0=OK |
| `0x12` | Idle air valve position | [2:3] uint16 BE | `val / 2` | steps | |
| `0x13` | Closed-loop status | [2] bit 0 | `val & 0x01` | bool | |
| `0x19` | Fault codes | — | see §9 | — | |
| `0x21` | RPM error | [2:3] uint16 BE | signed: if >32768 subtract 65535 | RPM | |
| `0x25` | Camshaft % (VVC) | [2:3] uint16 BE | raw | % | **Mini MPI: refused** |
| `0x3A` | Idle ignition offset | [2:3] uint16 BE | `val / 10` | degrees | |
| | Idle adjuster RPM | [4:5] uint16 BE | raw | RPM | |

### Temperature Encoding

MEMS2J uses **Kelvin × 10**

```raw
°C = (ECU_value − 2732) / 10

2982 → 25.0 °C
3600 → 86.8 °C
2582 → −15.0 °C
```

### Mini MPI Notes

| Frame | Status | Reason |
|---|---|---|
| `0x25` | **Refused by ECU** | No VVC on Mini MPI |
| `0x0B` | 2 coils only | Direct ignition, one coil per cylinder pair |
| `0x0C` | 4 injectors | Multi Point Injection |

## Fault Codes — Frame 0x19

Byte indices are 0-based within the response payload (byte 0 = `0x61`).

### Undervoltage Faults

| Buffer byte | Bit | Fault |
|---|---|---|
| [4] | 6 | Ambient air temperature |
| [4] | 5 | Supply voltage |
| [4] | 4 | Engine oil temperature |
| [4] | 2 | Coolant temperature |
| [4] | 0 | System |
| [5] | 7 | Battery |
| [5] | 4 | Lambda Bank 1 |
| [5] | 2 | Throttle potentiometer |
| [5] | 1 | Intake air temperature |
| [5] | 0 | MAP sensor |

### Overvoltage Faults

| Buffer byte | Bit | Fault |
|---|---|---|
| [8] | 6 | Ambient air temperature |
| [8] | 5 | Supply voltage |
| [8] | 4 | Oil temperature |
| [8] | 2 | Coolant temperature |
| [8] | 0 | System |
| [9] | 7 | Battery |
| [9] | 4 | Lambda Bank 1 |
| [9] | 2 | Throttle potentiometer |
| [9] | 1 | Intake air temperature |
| [9] | 0 | MAP sensor |

### Stored / Present Faults

| Buffer byte | Bit | Fault |
|---|---|---|
| [12] | 6 | Ambient temperature sensor |
| [12] | 5 | Supply voltage |
| [12] | 4 | Oil temperature |
| [12] | 2 | Coolant temperature |
| [13] | 7 | Battery voltage |
| [13] | 4 | Lambda Bank 1 |
| [13] | 2 | Throttle potentiometer |
| [13] | 1 | Intake air temperature |
| [13] | 0 | MAP sensor |
| [23] | 3 | MAP sensor (2) |
| [23] | 2 | Oil temperature (2) |
| [23] | 1 | Intake air temperature (2) |
| [23] | 0 | Coolant temperature (2) |
| [26] | 0 | Vehicle speed sensor |
| [26] | 1 | AT communication |
| [26] | 4 | Bank 1 fuelling feedback |
| [26] | 5 | Bank 2 fuelling feedback |

### Historic Faults

| Buffer byte | Bit | Fault |
|---|---|---|
| [25] | 3 | MAP sensor |
| [25] | 2 | Oil temperature |
| [25] | 1 | Intake air temperature |
| [25] | 0 | Coolant temperature |
| [28] | 0 | Vehicle speed sensor |
| [28] | 1 | AT communication |
| [28] | 4 | Fuelling feedback |

## Sources

| Source | Content |
|---|---|
| [rover-mems-agent — ecu-2j.go](https://github.com/james-portman/rover-mems-agent/blob/master/ecu-2j.go) | Connection sequence, command definitions, timing constants |
| [rover-mems-agent — ecu-2j-parse.go](https://github.com/james-portman/rover-mems-agent/blob/master/ecu-2j-parse.go) | Complete frame parser: byte offsets, formulas, Mini MPI exclusions |
| [rover-mems-agent — ecu-auth.go](https://github.com/james-portman/rover-mems-agent/blob/master/ecu-auth.go) | Seed-to-key algorithm |
| [rover-mems-agent — ecu-2j-faults.go](https://github.com/james-portman/rover-mems-agent/blob/master/ecu-2j-faults.go) | Fault code byte positions and bitmasks |
| [andrewrevill.co.uk — MEMSMapperEU2](https://andrewrevill.co.uk/MEMSMapperEU2.htm) | MEMS2J protocol documentation, service table |
