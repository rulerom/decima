# 路由和激活图

> **邻居形成继电器，数据通过总线**

---

## 🗺️ RoutingFlags16

### 比特映射 (LSB 优先)

| 比特 | 标志 | 描述 |
|------|------|------|
| **bit0** | N | 北：激活上方邻居 |
| **bit1** | E | 东：激活右侧邻居 |
| **bit2** | S | 南：激活下方邻居 |
| **bit3** | W | 西：激活左侧邻居 |
| **bit4** | NE | 东北：对角线 |
| **bit5** | SE | 东南：对角线 |
| **bit6** | SW | 西南：对角线 |
| **bit7** | NW | 西北：对角线 |
| **bit8** | BUS_R | 读取总线 (ACTIVE 源) |
| **bit9** | BUS_W | 写入总线 (WRITE 阶段) |
| **bit10..15** | reserved | 必须为 0 |

### 示例

```
routing_flags16 = 0x0301
# binary: 0000 0011 0000 0001
# bits: N(0) + BUS_R(8) + BUS_W(9)
```

---

## 🧭 邻居拓扑

### 图块阵列

```
tile_id = y * tile_w + x
```

### 基本邻居

| 方向 | 公式 | 条件 |
|------|------|------|
| **N(x,y)** | (x, y-1) | y > 0 |
| **S(x,y)** | (x, y+1) | y < tile_h-1 |
| **W(x,y)** | (x-1, y) | x > 0 |
| **E(x,y)** | (x+1, y) | x < tile_w-1 |

### 对角线

| 方向 | 公式 | 条件 |
|------|------|------|
| **NE(x,y)** | (x+1, y-1) | x < tile_w-1 且 y > 0 |
| **SE(x,y)** | (x+1, y+1) | x < tile_w-1 且 y < tile_h-1 |
| **SW(x,y)** | (x-1, y+1) | x > 0 且 y < tile_h-1 |
| **NW(x,y)** | (x-1, y-1) | x > 0 且 y > 0 |

---

## 🔗 激活边

### 规则

```
如果图块 A 设置了 Dir 标志且邻居 B = neighbor(A, Dir) 存在
→ 边 A → B
```

### 重要属性

| 属性 | 描述 |
|------|------|
| **仅 ACTIVE** | 边只影响 ACTIVE 计算 |
| **无数据传输** | 邻居不发送数据 |
| **多父节点** | 允许 |
| **循环** | 允许 (通过 locked_before 确定论) |

---

## 🌳 激活图 (ACTIVE 闭包)

### ACTIVE 计算

```python
# Seed: 源 (链根)
ACTIVE[t] = 1 如果 BUS_R[t] == 1

# Propagate: 图闭包
ACTIVE[t] = 1 如果存在 p ∈ Parents(t) 使得:
           ACTIVE[p] == 1 且 locked_before[p] == 1

# 最小不动点直到稳定
```

### 继电器示例

```
Tick N:
  图块 A (BUS_R=1): ACTIVE=1, 计算，locked_after=1
  图块 B (A 的父节点，标志 N): ACTIVE=0 (因为 locked_before[A]=0)

Tick N+1:
  图块 A: locked_before=1, 驱动总线
  图块 B: ACTIVE=1 (因为 ACTIVE[A]=1 且 locked_before[A]=1)
          → 读取 VSB_INGRESS → 计算
```

---

## 🎯 Parents(t) — 父节点集

```python
Parents(t) = { p | 图块 p 有 Dir 标志且 neighbor(p, Dir) = t }
```

### 示例

```
图块 (5,5) 的父节点:
- 图块 (5,6) 如果它有 N 标志
- 图块 (4,5) 如果它有 E 标志
- 图块 (5,4) 如果它有 S 标志
- 图块 (6,5) 如果它有 W 标志
- ...
```

---

## 🚨 分支坍缩

如果根/中间图块未融合 (reset/collide)：

```
Tick N:   图块 A: locked_before=1, 驱动总线
Tick N+1: 图块 A: reset → locked=0
Tick N+2: 图块 B (A 的后代): ACTIVE=0 → 强制为 0
```

### 强制零

```python
if ACTIVE[t] == 0:
  thr_cur16 := 0
  locked := 0
  drive_vec := {0, 0, 0, 0, 0, 0, 0, 0}
  # 权重/row/decay 不应用
```

---

## 📐 配置示例

### 4×4 阵列

```
┌─────┬─────┬─────┬─────┐
│ 0,0 │ 0,1 │ 0,2 │ 0,3 │  BUS_R=1 (源)
├─────┼─────┼─────┼─────┤
│ 1,0 │ 1,1 │ 1,2 │ 1,3 │  N=1 (从下方激活)
├─────┼─────┼─────┼─────┤
│ 2,0 │ 2,1 │ 2,2 │ 2,3 │  N=1
├─────┼─────┼─────┼─────┤
│ 3,0 │ 3,1 │ 3,2 │ 3,3 │  N=1
└─────┴─────┴─────┴─────┘
```

### 激活图

```
(0,0) BUS_R=1 → ACTIVE seed
  │
  ▼ (1,0 上的 N)
(1,0) → ACTIVE 如果 locked_before[(0,0)]=1
  │
  ▼ (2,0 上的 N)
(2,0) → ACTIVE 如果 locked_before[(1,0)]=1
  │
  ▼ (3,0 上的 N)
(3,0) → ACTIVE 如果 locked_before[(2,0)]=1
```

---

## 🔗 BUS_R / BUS_W

### BUS_R (读取)

| 参数 | 值 |
|------|-----|
| **比特** | bit8 (0x0100) |
| **用途** | ACTIVE 源 (图源) |
| **效果** | ACTIVE[t]=1 不考虑父节点 |

### BUS_W (写入)

| 参数 | 值 |
|------|-----|
| **比特** | bit9 (0x0200) |
| **用途** | BUS16 写入权限 |
| **条件** | locked self 或 locked_ancestor |

---

**Bake the Future. Build the Substrate.** 🛠️⚡️
