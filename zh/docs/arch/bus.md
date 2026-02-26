# BUS16

> **诚实地累加所有图块贡献**

---

## 📊 总线物理

### 配置

| 参数 | 值 |
|------|-----|
| **线路** | 8 lane (BUS16[0..7]) |
| **电平** | Level16 (0..15) |
| **模式** | WRITE 时累加 |
| **策略** | 饱和 (clamp15)，无 wrap |

---

## 🧮 贡献累加

在 PHASE_WRITE 中对每条线路 i=0..7：

```python
contrib_from_all_tiles[i] = Σ drive_vec[t][i]
  对所有 t 其中:
    - (routing_flags16[t] & BUS_W) != 0
    - locked self 或 locked_ancestor

bus_raw[i] = contrib_from_all_tiles[i]
BUS16[i] = clamp15(bus_raw[i])
BUS_CLIP[i] = (bus_raw[i] > 15)
```

### 范围

| 值 | 范围 |
|-----|------|
| **bus_raw** | [0..3840] (256 图块 × 15) |
| **BUS16** | [0..15] (clamped) |

---

## 🚨 CLIP / OVF 标志

### BUS_CLIP

```
BUS_CLIP[i] = (bus_raw[i] > 15)
BUS_CLIP_ANY = OR_i BUS_CLIP[i]
```

### BUS_OVF

```
BUS_OVF_ANY = BUS_CLIP_ANY
OVF_ANY = BUS_OVF_ANY
```

> **策略 v0.2：** 只饱和 (clamp15)，无 wrap/divide。

---

## 🔄 阶段纪律

### READ Phase

- Conductor 驱动 VSB[0..7]
- Island **不**驱动 BUS
- 图块读取 VSB_INGRESS

### TURNAROUND

```
Conductor: Hi-Z / no-drive VSB
Island: 准备驱动 BUS
```

### WRITE Phase

- Conductor: Hi-Z VSB
- Island 驱动 BUS16
- 带有 BUS_W=1 和 locked 的图块写入 drive_vec

### READOUT_SAMPLE

```
R0_RAW_BUS: readout = BUS16[0..7] 为 8×Level16
```

---

## 🎯 BUS 写入条件

图块写入 BUS16 仅当：

1. **BUS_W == 1** (routing_flags16 & 0x0200)
2. **locked self** 或 **locked_ancestor** (在激活图中有 locked 祖先)

### Drive Vector

```
if locked_after == 1:
  drive_vec[i] = in16[i]  # passthrough
else:
  drive_vec[i] = row16_out[i]  # 从权重计算
```

---

## 📐 计算示例

### 给定

- 3 图块写入线路 0
- drive_vec[0] = [10, 5, 8]

### 计算

```
bus_raw[0] = 10 + 5 + 8 = 23
BUS16[0] = clamp15(23) = 15
BUS_CLIP[0] = true (23 > 15)
```

---

## 🔗 与 VSB 的关系

| 平面 | 方向 | 阶段 |
|------|------|------|
| **VSB[0..7]** | Conductor → Island | READ |
| **BUS16[0..7]** | Island → Conductor | WRITE |

> **重要：** 数据不发送给邻居。邻居只形成激活图以允许 BUS 读取。

---

**Bake the Future. Build the Substrate.** 🛠️⚡️
