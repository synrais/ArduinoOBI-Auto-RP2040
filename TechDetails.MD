## Firmware State Machine

```
WAIT_BATTERY  ──►  SCAN_NOW  ──►  IDLE
     ▲                │              │
     │                │ scan failed  │ battery removed
     └────────────────┘◄─────────────┘
```

| State          | What is happening                                                    |
|----------------|----------------------------------------------------------------------|
| `WAIT_BATTERY` | Polling the bus every 800 ms for a presence pulse                   |
| `SCAN_NOW`     | Battery detected (or `s` pressed) — full scan about to execute      |
| `IDLE`         | Scan complete, polling every 800 ms to detect removal               |

Every 800 ms the firmware pulses `ENABLE_PIN` HIGH for 50 ms and fires a 1-Wire reset. If the battery answers with a presence pulse the state advances to `SCAN_NOW`. If the battery stops answering while in `IDLE` the state returns to `WAIT_BATTERY` and "Battery removed" is printed.

---

## Scan Sequence (Detail)

#### 1 — Defensive test-mode exit
`exit_testmode()` is called unconditionally (`0xCC 0xD9 0xFF 0xFF`) to clear any lingering firmware state from a previous interrupted scan.

#### 2 — Model read (bounded retry)
`cmd_cc(0xDC 0x0C)` reads up to 7 ASCII characters of the model string. Retries up to 15 times (~3 s) to allow the BMS to finish waking up. Aborts if nothing valid is returned.

#### 3 — Welcome flash
NeoPixel turns green, battery LEDs are briefly flashed via 1-Wire. Test mode is entered and then explicitly exited before step 4.

#### 4 — ROM ID read
`cmd_33(0xAA 0x00)` — 8-byte ROM ID captured for type detection.

#### 5 — Basic info frame
`cmd_cc(0xAA 0x00)` — 32-byte nybble-packed frame with type, cell count, capacity, flags, failure code, cycle count, checksums.

#### 6 — Battery type detection

| Priority | Type | Detection method |
|----------|------|-----------------|
| 1 | 5 | ROM ID byte 3 < 100 (F0513 chip) |
| 2 | 6 | Basic-info byte 17 == 30 (BL36xx) |
| 3 | 0 | `cmd_cc(0xDC 0x0B)` last byte == 0x06 |
| 4 | 2 | Test mode → `cmd_cc(0xDC 0x0A)` last byte == 0x06 → exit test mode |
| 5 | 3 | `cmd_cc(0xD4 0x2C 0x00 0x02)` last byte == 0x06 |
| 6 | 0 | Default fallback |

#### 7 — Voltages

- **Types 0 / 2 / 3** — `cmd_cc(0xD7 0x00 0x00 0x0C)` — pack + up to 5 cell voltages as 16-bit LE millivolts
- **Type 5** — five commands `0x31`–`0x35`, one per cell; pack = sum
- **Type 6** — `cmd_cc(0x10 0x21)` enters voltage-read mode, then `cmd_cc(0xD4)` returns 20 bytes; conversion: `V = (6000 − raw/10) / 1000`

#### 8 — Temperature

- **Types 0 / 2 / 3** — `cmd_cc(0xD7 0x0E 0x00 0x02)` → 16-bit LE tenths of Kelvin → °C
- **Type 5** — `cmd_cc(0x52)` — same format
- **Type 6** — `cmd_cc(0xD2)` — single byte, formula: `(-40x + 9323) / 100`

#### 9 — Health, counters, charge level

| Battery type | Health source |
|--------------|--------------|
| 5 / 6 | Calculated from cycle count, overload, and overdischarge fields in basic info |
| 0 / 2 / 3 | Dedicated health command; also reads OD event count, overload counters, coulomb counter |
| Other | Damage rating field from basic-info nybble 46 |

#### 10 — Lock state and auto-unlock

See [Auto-Unlock](#auto-unlock) above for user-facing description. Technically: up to 5 attempts of power-cycle → test mode → `0x33 0xDA 0x04` error-reset → exit test mode → re-parse basic info frame → verify all three primary checksums. Progressive retry delay: `200 ms × attempt number`.

#### 11 — Scan complete
`disable_bus()` called, ENABLE_PIN goes LOW, transitions to `IDLE`.

---

## Battery Type Raw Ranges

| Raw value | Decoded        |
|-----------|----------------|
| 0 – 12    | 4-cell BL14xx  |
| 13 – 29   | 5-cell BL18xx  |
| 30+       | 10-cell BL36xx |

---

## Why Checksums Fail

The most common cause is the battery being left in test mode by a previous interrupted scan. A correctly timed charger handshake and explicit test-mode exit resolves this. The auto-unlock sequence handles it automatically.

---

## OneWire Library

This project uses a locally patched version of the OneWire library (`lib/OneWire/`).

Bit-level timing has been widened from the standard 1-Wire spec to match Makita's BMS timing requirements (reset pulse 750 µs, write-1 recovery 120 µs, write-0 low 100 µs). These values are the same across all supported boards.

| Platform | GPIO implementation |
|----------|-------------------|
| AVR (Uno / Nano) | Direct port register access |
| ESP32-C3 | Direct register access via ESP32 GPIO hardware block |
| RP2040 | Direct Single-cycle IO (SIO) register access at `0xD0000000`; `delayMicroseconds` mapped to `busy_wait_us` from Pico SDK |
