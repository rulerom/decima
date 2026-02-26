# 协议 v0.2

> **事件协议 / SHM ABI / UDP**

---

## 📡 外部事件 (API)

### EV_FLASH(tag_u32)

| 参数 | 描述 |
|------|------|
| **用途** | 一个确定性 READ→WRITE 周期 |
| **返回** | readout (R0/R1), FLAGS 单独读取 |
| **条件** | 仅当 BAKE_APPLIED==1 时允许 |

```python
if BAKE_APPLIED == 0:
  return NotBaked  # 状态不变

# 执行 PHASE_READ → TURNAROUND → PHASE_WRITE → READOUT_SAMPLE
return readout, FLAGS32
```

### EV_RESET_DOMAIN(mask16)

| 参数 | 描述 |
|------|------|
| **用途** | 域重置 (thr_cur16=0, locked=0) |
| **条件** | 仅在 EV_FLASH 之间 |
| **条件** | 仅当 BAKE_APPLIED==1 |

### EV_BAKE()

| 参数 | 描述 |
|------|------|
| **用途** | 原子应用 staging BakeBlob |
| **条件** | 仅在 EV_FLASH 之间 |
| **效果** | 应用后重置运行时 |

---

## 🔄 EV_FLASH 内部子阶段

1. **PHASE_READ** — 图块采样输入
2. **TURNAROUND** — Conductor: Hi-Z
3. **PHASE_WRITE** — Island 驱动 BUS16
4. **READOUT_SAMPLE** — Conductor 读取 BUS
5. **INTERPHASE_AUTORESET** — 可选域重置

---

## 🌐 UDP 协议 (packet_v1)

> **机器级联**

### 数据包格式 (37 字节)

| Offset | 字段 | 类型 | 描述 |
|--------|------|------|------|
| **0** | magic | u32 | 'D8UP' (0x50553844) |
| **4** | version | u16 | 1 |
| **6** | flags | u16 | has_winner, has_bus, has_cycle, has_flags |
| **8** | frame_tag | u32 | 帧标签 |
| **12** | domain_id | u8 | 域 ID |
| **13** | pattern_id | u16 | 模式 ID |
| **15** | reset_mask16 | u16 | 重置掩码 |
| **17** | collision_mask16 | u16 | 碰撞掩码 |
| **19** | winner_tile_id | u16 | 获胜者 ID |
| **21** | cycle_time_us | u32 | 周期时间 |
| **25** | flags32_last | u32 | 上一周期 FLAGS |
| **29** | bus16[8] | u8×8 | 总线值 |

### Flags (u16)

| 比特 | 标志 | 描述 |
|------|------|------|
| **bit0** | has_winner | winner_tile_id/pattern_id 有效 |
| **bit1** | has_bus | bus16[8] 有效 |
| **bit2** | has_cycle | cycle_time_us 有效 |
| **bit3** | has_flags | flags32_last 有效 |

---

## 🎯 COLLIDE: 域和获胜者

### 定义

```
FIRE(t) = (locked_before[t]==0 && locked_after[t]==1)
FIRED_SET(d) = { t | domain_id(t)=d && FIRE(t)=1 }
cnt(d) = |FIRED_SET(d)|
```

### 规则

| cnt(d) | 获胜者 | COLLIDE(d) |
|--------|--------|------------|
| **0** | 无 | 0 |
| **1** | 单个 | 0 |
| **≥2** | 选择 | 1 |

### 获胜者选择 (cnt≥2)

```
1. max priority8
2. 平局时 min tile_id
```

---

**Bake the Future. Build the Substrate.** 🛠️⚡️
