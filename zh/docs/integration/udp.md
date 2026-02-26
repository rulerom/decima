# UDP 协议

> **DECIMA-8 机器级联**

---

## 📦 数据包格式

**packet_v1** — 固定二进制格式

| 特征 | 值 |
|------|-----|
| **大小** | 37 字节 |
| **Endianness** | Little-endian |
| **用途** | 机器级联 (IN/OUT) |

---

## 🗂️ 数据包结构

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

---

## 🚩 Flags (u16)

| 比特 | 标志 | 描述 |
|------|------|------|
| **bit0** | has_winner | winner_tile_id/pattern_id 有效 |
| **bit1** | has_bus | bus16[8] 有效 |
| **bit2** | has_cycle | cycle_time_us 有效 |
| **bit3** | has_flags | flags32_last 有效 |

---

## 📋 数据包字段说明

### magic (u32)
```
'D8UP' (0x50553844)
```

### version (u16)
```
1
```

### flags (u16)
```
bit0: has_winner
bit1: has_bus
bit2: has_cycle
bit3: has_flags
```

### frame_tag (u32)
```
用于同步的唯一帧标签
```

### domain_id (u8)
```
域 ID (0..15)
```

### pattern_id (u16)
```
模式 ID (0..32767)
如果 flags.has_winner 则有效
```

### reset_mask16 (u16)
```
用于 RESET_DOMAIN 的域掩码
每位对应一个域
```

### collision_mask16 (u16)
```
具有碰撞的域掩码 (cnt≥2)
如果 flags.has_winner 则有效
```

### winner_tile_id (u16)
```
获胜图块 ID
如果 flags.has_winner 则有效
```

### cycle_time_us (u32)
```
周期时间 (微秒)
如果 flags.has_cycle 则有效
```

### flags32_last (u32)
```
上一周期 FLAGS32:
bit0: READY_LAST
bit1: OVF_ANY_LAST
bit2: COLLIDE_ANY_LAST
如果 flags.has_flags 则有效
```

### bus16[8] (u8×8)
```
BUS16[0..7] 值
如果 flags.has_bus 则有效
```

---

## 💻 使用示例

### 发送数据包

```python
import socket

sock = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)

packet = struct.pack('<4sHHIIBHHHHIIBBBBBBBB',
    b'D8UP',           # magic
    1,                 # version
    0x000F,            # flags (所有字段有效)
    frame_tag,
    domain_id,
    pattern_id,
    reset_mask16,
    collision_mask16,
    winner_tile_id,
    cycle_time_us,
    flags32_last,
    *bus16             # 8 bytes
)

sock.sendto(packet, ('127.0.0.1', 9999))
```

### 接收数据包

```python
data, addr = sock.recvfrom(37)

magic, version, flags, frame_tag, domain_id, pattern_id, \
reset_mask16, collision_mask16, winner_tile_id, cycle_time_us, \
flags32_last, *bus16_bytes = struct.unpack('<4sHHIIBHHHHIIBBBBBBBB', data)

bus16 = list(bus16_bytes)

has_winner = bool(flags & 0x0001)
has_bus = bool(flags & 0x0002)
has_cycle = bool(flags & 0x0004)
has_flags = bool(flags & 0x0008)
```

---

## 🔗 集成

### 输入流 (IN)

```
DECIMA-8 接收数据包:
- reset_mask16 → EV_RESET_DOMAIN
- pattern_id → 目标模式
- bus16 → 输入数据 (如果 has_bus)
```

### 输出流 (OUT)

```
DECIMA-8 发送数据包:
- frame_tag → 同步
- winner_tile_id → 识别的模式
- bus16 → 总线状态
- flags32_last → 周期标志
```

---

**Bake the Future. Build the Substrate.** 🛠️⚡️
