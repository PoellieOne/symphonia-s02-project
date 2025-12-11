# EVENT24 Protocol Documentation

## Symphonia S02 Core-0 → Host Communication

**Version:** 1.1  
**Last Updated:** 2025-11-28  
**Firmware:** S02-Core0-Sensing v1.1.0

---

## 1. Overview

The EVENT24 packet is the primary data structure for communicating magnetic field transition events from the ESP32 Core-0 firmware to the host (Python/Core-1 Awareness layer).

```
┌─────────────┐      UART/Binary       ┌─────────────┐
│   ESP32     │  ─────────────────────▶│    Host     │
│   Core-0    │      EVENT24 packets   │   Python    │
│  (Sensing)  │                        │  (Core-1)   │
└─────────────┘                        └─────────────┘
```

---

## 2. Frame Structure

All packets use a common framing protocol:

```
┌──────┬──────────┬─────┬─────────────────┬───────┐
│ SYNC │ TYPE|VER │ LEN │     PAYLOAD     │ CRC16 │
├──────┼──────────┼─────┼─────────────────┼───────┤
│ 0xA5 │  1 byte  │ 1B  │   LEN bytes     │ 2B LE │
└──────┴──────────┴─────┴─────────────────┴───────┘
```

| Field | Size | Description |
|-------|------|-------------|
| SYNC | 1 byte | Always `0xA5` |
| TYPE\|VER | 1 byte | Upper nibble = packet type, Lower nibble = version |
| LEN | 1 byte | Payload length in bytes |
| PAYLOAD | LEN bytes | Packet-specific data |
| CRC16 | 2 bytes | CRC16-CCITT(FALSE) over TYPE\|VER + LEN + PAYLOAD |

### Packet Types

| Type | Value | Payload Size | Description |
|------|-------|--------------|-------------|
| PKT_EVENT16 | 0x0 | 11 bytes | Basic event (deprecated) |
| PKT_EVENT24 | 0x1 | 19 bytes | Extended event (primary) |
| PKT_SUMMARY16 | 0x2 | 11 bytes | Legacy summary |
| PKT_SUMMARY24 | 0x3 | 19 bytes | Legacy summary |
| PKT_FILTER_STATS | 0x4 | 19 bytes | Filter layer statistics |
| PKT_LINK_STATS | 0x5 | 19 bytes | Transport layer statistics |

---

## 3. EVENT24 Payload Structure

**Total size: 19 bytes**

```
Offset  Size  Field           Format      Description
──────────────────────────────────────────────────────────
 0      2     dt_us           uint16 LE   Delta time since last event (µs)
 2      4     t_abs_us        uint32 LE   Absolute timestamp (µs)
 6      1     flags0          uint8       Sensor/quality flags
 7      1     flags1          uint8       Pool/direction flags
 8      2     dvdt_q15        int16 LE    dV/dt magnitude (Q15 scaled)
10      1     mono_q8         uint8       Monotonicity score (0-255)
11      1     snr_q8          uint8       Signal-to-noise ratio (0-255)
12      1     fit_err_q8      uint8       Fit error score (0-255)
13      2     rpm_hint_q      uint16 LE   RPM hint (reserved, currently 0)
15      1     score_q8        uint8       Overall quality score (0-255)
16      1     seq             uint8       Sequence number (0-255, wraps)
17      2     reserved        uint16      Reserved for future use
```

### 3.1 Byte Layout Diagram

```
Byte:  0   1   2   3   4   5   6   7   8   9  10  11  12  13  14  15  16  17  18
     ├───┴───┼───┴───┴───┴───┼───┼───┼───┴───┼───┼───┼───┼───┴───┼───┼───┼───┴───┤
     │ dt_us │   t_abs_us    │f0 │f1 │dvdt_q15│mon│snr│fit│rpm_hnt│scr│seq│reservd│
     └───────┴───────────────┴───┴───┴───────┴───┴───┴───┴───────┴───┴───┴───────┘
```

---

## 4. Flags Decoding

### 4.1 flags0 (Byte 6)

```
Bit:   7      6  5     4        3       2  1  0
     ┌────┬───────┬────────┬────────┬─────────────┐
     │PAIR│QLEVEL │POLARITY│ SENSOR │  reserved   │
     └────┴───────┴────────┴────────┴─────────────┘
```

| Bits | Field | Values |
|------|-------|--------|
| 7 | pair | 0 = single event, 1 = paired with other sensor |
| 6:5 | qlevel | 0 = rejected, 1 = weak, 2 = normal, 3 = strong |
| 4 | polarity | 0 = negative (toward SOUTH), 1 = positive (toward NORTH) |
| 3 | sensor | 0 = Sensor A, 1 = Sensor B |
| 2:0 | reserved | Always 0 |

### 4.2 flags1 (Byte 7)

```
Bit:   7  6     5  4     3  2     1  0
     ┌───────┬───────┬───────┬───────┐
     │FROM_P │ TO_P  │DIR_HNT│EDGE_K │
     └───────┴───────┴───────┴───────┘
```

| Bits | Field | Values |
|------|-------|--------|
| 7:6 | from_pool | Pool before transition (see below) |
| 5:4 | to_pool | Pool after transition (see below) |
| 3:2 | dir_hint | Direction hint (see below) |
| 1:0 | edge_kind | Edge classification (see below) |

#### Pool Values (2 bits)

| Value | Name | Description |
|-------|------|-------------|
| 0 | POOL_NEU | Neutral zone (between thresholds) |
| 1 | POOL_N | North pole (above high threshold) |
| 2 | POOL_S | South pole (below low threshold) |
| 3 | POOL_UNK | Unknown (reserved) |

#### Direction Hint Values (2 bits)

| Value | Name | Description |
|-------|------|-------------|
| 0 | DIR_NONE | No direction determined (V1.0 default) |
| 1 | DIR_CW | Clockwise rotation detected |
| 2 | DIR_CCW | Counter-clockwise rotation detected |
| 3 | reserved | Reserved |

#### Edge Kind Values (2 bits)

| Value | Name | Description |
|-------|------|-------------|
| 0 | EDGE_KIND_ZC | Baseline zero-crossing |
| 1 | EDGE_KIND_POOL_HARD | Direct N↔S pool transition |
| 2 | EDGE_KIND_POOL_NEUT | Pool ↔ Neutral transition |
| 3 | reserved | Reserved |

---

## 5. Quality Metrics

### 5.1 dvdt_q15 (dV/dt)

Rate of voltage change at the transition point.

```
Physical dV/dt (LSB/µs) = dvdt_q15 × 100
```

- **Higher values** = faster, sharper transition = better quality
- **Typical range:** 500 - 5000 (raw), 5 - 50 (in Q15 packet)
- **Threshold:** `dvdt_min_q15` (default: 15)

### 5.2 mono_q8 (Monotonicity)

Measures how consistently the signal moves in one direction during transition.

```
mono_q8 = 0   → chaotic, bouncing signal
mono_q8 = 255 → perfectly monotonic transition
```

- **Calculation:** Fraction of sample-pairs that follow expected direction
- **Threshold:** `mono_min_q8` (default: 105 ≈ 41%)

### 5.3 snr_q8 (Signal-to-Noise Ratio)

Signal strength relative to baseline noise.

```
snr_q8 = min(SNR × 2.0, 255)
SNR = |signal - baseline| / noise_stddev
```

- **Threshold:** `snr_min_q8` (default: 10 ≈ SNR of 5)

### 5.4 fit_err_q8 (Fit Error)

**V1.0 (default):** Simple noise scaling
```c
fit_err_q8 = min(noise × 5.0, 255)
```

**V1.1 (improved, optional):** Linear regression residual
```c
// Least-squares fit: y = mx + b
// fit_err = mean absolute residual × 2.55
```

- **Lower values** = cleaner, more linear transition
- **Threshold:** `fit_err_max_q8` (default: 80)

### 5.5 score_q8 (Overall Quality)

Weighted combination of all metrics:

```c
score = w_dvdt × norm(dvdt) 
      + w_mono × mono_q8 
      + w_snr  × snr_q8 
      - w_fit  × fit_err_q8

score_q8 = clamp(score >> 8, 0, 255)
```

Default weights: `w_dvdt=2, w_mono=3, w_snr=2, w_fit=1`

---

## 6. Quality Levels (qlevel)

Events are classified into quality levels based on thresholds:

| Level | Name | Criteria |
|-------|------|----------|
| 0 | Rejected | Failed any gate (dvdt, mono, snr, fit_err) |
| 1 | Weak | Passed gates, below normal thresholds |
| 2 | Normal | dvdt ≥ min, mono ≥ min, snr ≥ min |
| 3 | Strong | dvdt ≥ 1.5×min, mono ≥ 200, snr ≥ min+24 |

---

## 7. Pair Detection & Direction

### 7.1 Pair Detection

Two events are considered a "pair" if:
- Different sensors (A and B)
- Time difference ≤ `PAIR_WINDOW_US` (default: 50,000 µs = 50ms)

```
Sensor A event ──┐
                 ├── dt ≤ 50ms? ──▶ PAIR
Sensor B event ──┘
```

### 7.2 Direction Hint (V1.1)

When `CFG_DIR.enabled = true`:

```
Hardware geometry: Sensors A and B are ~100° apart

CW rotation:  Magnet passes A first, then B
              A_then_B → dir_hint = DIR_CW

CCW rotation: Magnet passes B first, then A  
              B_then_A → dir_hint = DIR_CCW
```

**Consistency check** (if `require_consistent_transition = true`):
Both sensors must show the same pool transition (e.g., both NEU→NORTH)

---

## 8. Python Decoding Example

```python
import struct

def parse_event24(payload: bytes) -> dict:
    """Parse 19-byte EVENT24 payload"""
    
    dt_us, t_abs_us, flags0, flags1, dvdt_q15, \
    mono_q8, snr_q8, fit_err_q8, rpm_hint_q, \
    score_q8, seq = struct.unpack('<HIBBBHBBBHBB', payload[:17])
    
    # Decode flags0
    pair     = (flags0 >> 7) & 0x1
    qlevel   = (flags0 >> 5) & 0x3
    polarity = (flags0 >> 4) & 0x1
    sensor   = (flags0 >> 3) & 0x1
    
    # Decode flags1
    from_pool = (flags1 >> 6) & 0x3
    to_pool   = (flags1 >> 4) & 0x3
    dir_hint  = (flags1 >> 2) & 0x3
    edge_kind = (flags1 >> 0) & 0x3
    
    return {
        "dt_us": dt_us,
        "t_abs_us": t_abs_us,
        "sensor": "A" if sensor == 0 else "B",
        "pair": bool(pair),
        "qlevel": qlevel,
        "polarity": "+" if polarity else "-",
        "from_pool": ["NEU", "NORTH", "SOUTH", "UNK"][from_pool],
        "to_pool": ["NEU", "NORTH", "SOUTH", "UNK"][to_pool],
        "dir_hint": ["NONE", "CW", "CCW", "?"][dir_hint],
        "edge_kind": ["ZC", "POOL_HARD", "POOL_NEUT", "?"][edge_kind],
        "dvdt_q15": dvdt_q15,
        "mono_q8": mono_q8,
        "snr_q8": snr_q8,
        "fit_err_q8": fit_err_q8,
        "score_q8": score_q8,
        "seq": seq,
    }
```

---

## 9. Typical Event Patterns

### 9.1 Clean Rotation (CW)

```
seq  sensor  from→to      dir   qlevel  pair
───────────────────────────────────────────────
 0   A       NEU→NORTH    NONE    3      0
 1   B       NEU→NORTH    CW      3      1   ← paired with #0
 2   A       NORTH→NEU    NONE    2      0
 3   B       NORTH→NEU    CW      2      1
 4   A       NEU→SOUTH    NONE    3      0
 5   B       NEU→SOUTH    CW      3      1
 ...
```

### 9.2 Nudge (no rotation)

```
seq  sensor  from→to      dir   qlevel  pair
───────────────────────────────────────────────
 0   A       NEU→NORTH    NONE    2      0   ← single, no pair
 1   A       NORTH→NEU    NONE    2      0   ← back to neutral
```

### 9.3 Noisy/Bouncing

```
seq  sensor  from→to      dir   qlevel  pair
───────────────────────────────────────────────
 0   A       NEU→NORTH    NONE    1      0   ← weak quality
 1   A       NORTH→NEU    NONE    0      0   ← rejected (not emitted in production)
 2   A       NEU→NORTH    NONE    1      0   ← bouncing
```

---

## 10. Configuration Reference

Key parameters affecting EVENT24 output:

| Parameter | Default | Description |
|-----------|---------|-------------|
| `CFG_SENS_A.low_th` | 1610 | Sensor A SOUTH threshold |
| `CFG_SENS_A.high_th` | 2540 | Sensor A NORTH threshold |
| `CFG_SENS_B.low_th` | 740 | Sensor B SOUTH threshold |
| `CFG_SENS_B.high_th` | 1290 | Sensor B NORTH threshold |
| `CFG_FILT.dvdt_min_q15` | 15 | Minimum dV/dt for acceptance |
| `CFG_FILT.mono_min_q8` | 105 | Minimum monotonicity |
| `CFG_FILT.snr_min_q8` | 10 | Minimum SNR |
| `CFG_FILT.fit_err_max_q8` | 80 | Maximum fit error |
| `CFG_PAIR.window_us` | 50000 | Pair detection window (µs) |
| `CFG_DIR.enabled` | false | Enable direction hint (V1.1) |

---

## 11. Version History

| Version | Date | Changes |
|---------|------|---------|
| 1.0 | 2025-10 | Initial EVENT24 format |
| 1.1 | 2025-11 | Added dir_hint, improved fit_error (optional) |

---

## 12. See Also

- `core0_config.h` - All configuration parameters
- `binlink.py` - Python frame parser
- `capture_core0.py` - Capture and logging script
- `core0_analyzer.py` - Event analysis tools
