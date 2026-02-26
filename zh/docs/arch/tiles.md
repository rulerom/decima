# 图块模型 v0.2

> **图块 = DECIMA-8 的最小可编程实体**

---

## 📊 图块结构

### 烘焙状态 (Baked State)

| 参数 | 类型 | 范围 | 描述 |
|------|------|------|------|
| **thr_lo16** | i16 | -32768..+32767 | 熔丝下阈值 |
| **thr_hi16** | i16 | -32768..+32767 | 熔丝上阈值 |
| **decay16** | u16 | 0..32767 | 衰减到零 |
| **domain_id4** | u8 | 0..15 | 重置组 |
| **priority8** | u8 | 0..255 | 碰撞优先级 |
| **pattern_id16** | u16 | 0..32767 | 模式 ID |
| **routing_flags16** | u16 | 10 比特 | 激活方向 |
| **W[8][8]** | SignedWeight5 | mag3(0..7)+sign1 | 权重矩阵 8×8 (-7..+7) |
| **reset_on_fire_mask16** | u16 | 16 比特 | 触发时自动重置域 |

### 运行时状态 (Runtime)

| 参数 | 类型 | 描述 |
|------|------|------|
| **thr_cur16** | i16 | 当前累加器 (-32768..+32767) |
| **locked** | 0/1 | 熔丝锁定 (latched) |
| **drive_vec[8]** | u8[8] | 输出值 (0..15) |

> **注意：** Decay 总是应用 (如果 decay16 > 0)，即使在锁定的图块上。

---

## 🔄 ACTIVE 闭包 (激活图)

图块被认为是 **ACTIVE** 如果它在"活动链"中，可以读取 BUS16、计算、应用 decay。

### 激活规则

```
Seed: ACTIVE[t] = 1 如果 BUS_R == 1 (源/链根)

Propagate: ACTIVE[t] = 1 如果存在 p ∈ Parents(t) 使得:
           ACTIVE[p] == 1 且 locked_before[p] == 1
```

这被计算为**单调闭包直到稳定** (最小不动点)。

### 分支坍缩

如果 `ACTIVE[t] == 0`，图块被认为是"死区"：

```
thr_cur16 := 0
locked := 0
drive_vec := {0, 0, 0, 0, 0, 0, 0, 0}
```

权重、decay、drive 在此 tick 中不应用。

---

## 📥 图块输入 (VSB_INGRESS)

如果 `ACTIVE[t] == 1`，图块**只读取 VSB_INGRESS16** (所有 8 lanes)：

```
for i in 0..7:
  in16[t][i] = clamp15(VSB_INGRESS16[i])
  IN_CLIP[t][i] = (VSB_INGRESS16[i] > 15)
```

> **重要：** BUS16 不跟 VSB 累加。BUS16 在 READ 阶段的唯一作用是语义的：带有 BUS_R 标志的图块成为激活图源 (ACTIVE seed)。总线数据不参与 in16 计算。

### 继电器 (2 ticks)

```
Tick N:   祖先融合 → 在 PHASE_WRITE 中驱动总线
Tick N+1: 后代通过 BUS_R 激活 → 读取 VSB_INGRESS → 计算
```

---

## 🔒 FUSE-LOCK 机制

### Decay-to-Zero + 范围熔丝

**Decay 总是应用** (如果 decay16 > 0)，即使在锁定图块上。

如果 `locked_before == 0`：

```python
# 1. 计算 row-pipeline
delta_raw = Σ row16_signed[r]  # 范围 [-6720..+6720]

# 2. 更新累加器
thr_tmp = thr_cur16 + delta_raw

# 3. Decay 拉向 0，不跳过 0
if decay16 > 0:
  if thr_tmp > 0:
    thr_tmp = max(thr_tmp - decay16, 0)
  elif thr_tmp < 0:
    thr_tmp = min(thr_tmp + decay16, 0)

thr_cur16 = clamp_range(thr_tmp, -32768, 32767)

# 4. 范围熔丝
range_active = (thr_lo16 < thr_hi16)
in_range = range_active AND (thr_lo16 <= thr_cur16) AND (thr_cur16 <= thr_hi16)

has_signal = (delta_raw != 0)
entered_by_decay = (decay16 > 0) AND (in_range == true) AND (in_range_before_decay == false)

locked_after = (BAKE_APPLIED == 1) AND in_range AND (has_signal OR entered_by_decay)
```

如果 `locked_before == 1`：

```python
locked_after := 1
# 权重不应用 (passthrough)

# Decay 总是应用 (如果 decay16 > 0)
if decay16 > 0:
  if thr_cur16 > 0:
    thr_cur16 = max(thr_cur16 - decay16, 0)
  elif thr_cur16 < 0:
    thr_cur16 = min(thr_cur16 + decay16, 0)
```

### Locked Passthrough

如果 `locked_after == 1`，图块作为"铜桥"：

- 权重矩阵 W **不应用**
- `drive_vec[i] = in16[i]` 对所有 i=0..7 (passthrough)
- **Decay 应用** (如果 decay16 > 0, thr_cur16 拉向 0)

---

## 🧮 RowOut Pipeline (PHASE_READ)

对每行 r=0..7：

### Signed 乘法

```
row_raw_signed[r] = Σ_{i=0..7} (in16[i] * Wmag[r][i] * sign)
# 范围：[-840..+840]
```

### 线路/Drive (无负数)

```
row16_out[r] = clamp15((max(row_raw_signed[r], 0) + 7) / 8)
# 范围：0..15
```

### 累加器 (signed)

```
row16_signed[r] = row_raw_signed[r]
# 范围：[-840..+840]
```

---

## 🚗 Drive 选择 (WRITE)

在 READ 结束时：

```
if locked_after == 1:
  drive_vec[i] = in16[i]  # passthrough
else:
  drive_vec[i] = row16_out[i]  # 从权重计算
```

---

## 🎯 图块事件

| 事件 | 条件 |
|------|------|
| **LOCK_TRANSITION(t)** | locked_before==0 && locked_after==1 |
| **FIRE(t)** | LOCK_TRANSITION(t) |

### 不变量 v0.2

```
locked == 1 ⇒ thr_lo16 <= thr_cur16 <= thr_hi16
```

---

## 📐 SignedWeight5

权重：mag3∈[0..7], sign1∈{0,1} (1="+", 0="−")。

```
mul_signed_raw(a, mag, sign) = (sign ? +1 : -1) * (a * mag)
# a∈[0..15], mag∈[0..7] → [-105..+105] 每项
# 8 项每行 → [-840..+840] 每行
```

---

**Bake the Future. Build the Substrate.** 🛠️⚡️
