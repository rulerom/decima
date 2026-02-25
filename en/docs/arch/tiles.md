# Tile Model v0.2

> **Tile = minimal programmable entity of Decima-8**

---

## 📊 Tile Structure

### Baked State

| Parameter | Type | Range | Description |
|-----------|------|-------|-------------|
| **thr_lo16** | i16 | -32768..+32767 | Lower fuse threshold |
| **thr_hi16** | i16 | -32768..+32767 | Upper fuse threshold |
| **decay16** | u16 | 0..32767 | Decay to zero |
| **domain_id4** | u8 | 0..15 | Reset group |
| **priority8** | u8 | 0..255 | Collision priority |
| **pattern_id16** | u16 | 0..32767 | Pattern ID |
| **routing_flags16** | u16 | 10 bits | Activation directions |
| **W[8][8]** | SignedWeight5 | mag4+sign1 | Weight matrix 8×8 |
| **reset_on_fire_mask16** | u16 | 16 bits | Auto-reset domains on fire |

### Runtime

| Parameter | Type | Description |
|-----------|------|-------------|
| **thr_cur16** | i16 | Current accumulator (-32768..+32767) |
| **locked** | 0/1 | Fuse latched |
| **drive_vec[8]** | u8[8] | Output values (0..15) |

> **Note:** Decay is always applied (if decay16 > 0), even on locked tiles.

---

## 🔄 ACTIVE Closure (Activation Graph)

A tile is **ACTIVE** if it's "in the live chain" and can read BUS16, compute, apply decay.

### Activation Rules

```
Seed: ACTIVE[t] = 1 if BUS_R == 1 (source/root of chain)

Propagate: ACTIVE[t] = 1 if exists p ∈ Parents(t) such that:
           ACTIVE[p] == 1 AND locked_before[p] == 1
```

This is computed as **monotonic closure until stabilization** (least fixed point).

### Branch Collapse

If `ACTIVE[t] == 0`, tile is considered "dead zone":

```
thr_cur16 := 0
locked := 0
drive_vec := {0, 0, 0, 0, 0, 0, 0, 0}
```

Weights, decay, drive are not applied in this tick.

---

## 📥 Tile Input (VSB_INGRESS)

If `ACTIVE[t] == 1`, tile reads **only VSB_INGRESS16** (all 8 lanes):

```
for i in 0..7:
  in16[t][i] = clamp15(VSB_INGRESS16[i])
  IN_CLIP[t][i] = (VSB_INGRESS16[i] > 15)
```

> **Important:** BUS16 is not summed with VSB. The only role of BUS16 in READ phase is semantic: tiles with BUS_R flag become activation graph sources (ACTIVE seed).

### Relay (2 ticks)

```
Tick N:   Ancestor fuses → drives bus in PHASE_WRITE
Tick N+1: Descendant activates via BUS_R → reads VSB_INGRESS → computes
```

---

## 🔒 FUSE-LOCK Mechanism

### Decay-to-Zero + Fuse-by-Range

**Decay is always applied** (if decay16 > 0), even on locked tiles.

If `locked_before == 0`:

```python
# 1. Compute row-pipeline
delta_raw = Σ row16_signed[r]  # range [-6720..+6720]

# 2. Update accumulator
thr_tmp = thr_cur16 + delta_raw

# 3. Decay pulls to 0, doesn't jump over
if decay16 > 0:
  if thr_tmp > 0:
    thr_tmp = max(thr_tmp - decay16, 0)
  elif thr_tmp < 0:
    thr_tmp = min(thr_tmp + decay16, 0)

thr_cur16 = clamp_range(thr_tmp, -32768, 32767)

# 4. Fuse by range
range_active = (thr_lo16 < thr_hi16)
in_range = range_active AND (thr_lo16 <= thr_cur16) AND (thr_cur16 <= thr_hi16)

has_signal = (delta_raw != 0)
entered_by_decay = (decay16 > 0) AND (in_range == true) AND (in_range_before_decay == false)

locked_after = (BAKE_APPLIED == 1) AND in_range AND (has_signal OR entered_by_decay)
```

If `locked_before == 1`:

```python
locked_after := 1
# Weights not applied (passthrough)

# Decay is always applied (if decay16 > 0)
if decay16 > 0:
  if thr_cur16 > 0:
    thr_cur16 = max(thr_cur16 - decay16, 0)
  elif thr_cur16 < 0:
    thr_cur16 = min(thr_cur16 + decay16, 0)
```

### Locked Passthrough

If `locked_after == 1`, tile acts as "copper bridge":

- Weight matrix W is **not applied**
- `drive_vec[i] = in16[i]` for all i=0..7 (passthrough)
- **Decay is applied** (if decay16 > 0, pulls thr_cur16 to 0)

### Latched State

If `locked_before == 1`:

- `locked_after := 1`
- Weights not applied (passthrough)
- **Decay is always applied** (if decay16 > 0)

```python
# Decay pulls to 0 even on locked tiles
if decay16 > 0:
  if thr_cur16 > 0:
    thr_cur16 = max(thr_cur16 - decay16, 0)
  elif thr_cur16 < 0:
    thr_cur16 = min(thr_cur16 + decay16, 0)
```

---

## 🧮 RowOut Pipeline (PHASE_READ)

For each row r=0..7:

### Signed Multiplication

```
row_raw_signed[r] = Σ_{i=0..7} (in16[i] * Wmag[r][i] * sign)
# Range: [-840..+840]
```

### For Lines/Drive (no negatives)

```
row16_out[r] = clamp15((max(row_raw_signed[r], 0) + 7) / 8)
# Range: 0..15
```

### For Accumulator (signed)

```
row16_signed[r] = row_raw_signed[r]
# Range: [-840..+840]
```

---

## 🚗 Drive Selection (WRITE)

At end of READ:

```
if locked_after == 1:
  drive_vec[i] = in16[i]  # passthrough
else:
  drive_vec[i] = row16_out[i]  # computed from weights
```

---

## 🎯 Tile Events

| Event | Condition |
|-------|-----------|
| **LOCK_TRANSITION(t)** | locked_before==0 && locked_after==1 |
| **FIRE(t)** | LOCK_TRANSITION(t) |

### Invariant v0.2

```
locked == 1 ⇒ thr_lo16 <= thr_cur16 <= thr_hi16
```

---

## 📐 SignedWeight5

Weight: mag4 ∈ [0..15], sign1 ∈ {0,1} (1="+", 0="−")

```
mul_signed_raw(a, mag4, sign1) = (sign1 ? +1 : -1) * (a * mag4)
# a ∈ [0..15] → range [-225..+225]
```

---

## 🧩 State Example

```json
{
  "baked": {
    "thr_lo16": 100,
    "thr_hi16": 200,
    "decay16": 5,
    "domain_id4": 0,
    "priority8": 128,
    "pattern_id16": 42,
    "routing_flags16": 0x0301,  // N + BUS_R
    "reset_on_fire_mask16": 0x0001
  },
  "runtime": {
    "thr_cur16": 150,
    "locked": 1
  }
}
```

---

**Bake the Future. Build the Substrate.** 🛠️⚡️
