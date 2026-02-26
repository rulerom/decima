# Bake TLV 规范 v0.2

> **烘焙神经形态个性的二进制格式**

---

## 📐 一般规则

| 规则 | 描述 |
|------|------|
| **Endianness** | Little-endian |
| **TLV Padding** | 0, value 对齐到 4 字节边界 |
| **CRC32** | IEEE (zlib/crc32) 从 offset 0 到 TLV_CRC32 开始的所有字节 |

---

## 📋 Header (28 字节)

### BakeBlobHeader

| Offset | 字段 | 类型 | 值 |
|--------|------|------|-----|
| **0** | magic | char[4] | "D8BK" |
| **4** | ver_major | u16 | 2 |
| **6** | ver_minor | u16 | 0 |
| **8** | flags | u32 | BAKE_FLAG_DOUBLE_STRAIT (bit0) |
| **12** | total_len | u32 | 完整 blob 大小 |
| **16** | bake_id | u32 | 烘焙 ID |
| **20** | profile_id | u32 | 配置文件 ID |
| **24** | reserved0 | u32 | 0 |

### Header 标志

| 比特 | 标志 | 描述 |
|------|------|------|
| **bit0** | BAKE_FLAG_DOUBLE_STRAIT | Conductor 对每个输入和弦进行双重注入 |

---

## 🧩 TLV Header (8 字节)

| Offset | 字段 | 类型 |
|--------|------|------|
| **0** | type | u16 |
| **2** | tflags | u16 |
| **4** | len | u32 |

---

## 📦 TLV 类型映射 v0.2

| TLV 类型 | ID | 必需 | 描述 |
|----------|-----|------|------|
| **TLV_TOPOLOGY** | 0x0100 | ✅ | 阵列拓扑 |
| **TLV_TILE_PARAMS_V2** | 0x0121 | ✅ | 图块参数 (13 字节/图块) |
| **TLV_TILE_ROUTING_FLAGS16** | 0x0131 | ✅ | 路由标志 |
| **TLV_READOUT_POLICY** | 0x0140 | ✅ | 读出策略 |
| **TLV_RESET_ON_FIRE_MASK16** | 0x0150 | ✅ | 触发时自动重置 |
| **TLV_TILE_WEIGHTS_PACKED** | 0x0160 | ✅ | 权重 8×8 |
| **TLV_TILE_FIELD_LIMIT** | 0x0170 | ❌ | 图块限制 |
| **TLV_CRC32** | 0xFFFE | ✅ | 校验和 (最后) |

---

## ✅ 加载时验证

Bake 加载器必须验证：

1. **Header:** magic/ver/total_len
2. **TLV 存在:** 所有必需 TLV
3. **严格长度:**
   - TILE_PARAMS_V2: tile_count × 13
   - TILE_ROUTING_FLAGS16: tile_count × 2
   - TILE_WEIGHTS_PACKED: tile_count × 40
   - RESET_ON_FIRE_MASK16: tile_count × 2
4. **CRC32:** "TLV_CRC32 之前"的规则
5. **Reserved 字段:** = 0
6. **不变量:** thr_lo16 <= thr_hi6, tile_count == tile_w × tile_h

---

**Bake the Future. Build the Substrate.** 🛠️⚡️
