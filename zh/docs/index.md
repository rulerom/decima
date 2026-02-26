# DECIMA-8 🧠 — 神经形态引擎

> **用于神经形态计算的确定性节奏：Emulator → Proto (PCB) → FPGA → ASIC**

**状态：** v0.2 DESIGN FREEZE

**代号：** Siberian Tank Interface

---

## 📖 概述

**DECIMA-8** 是一种具有确定性节奏和可编程图块结构的神经形态引擎架构。

### 开放规范

DECIMA-8 规范开放供实现。我们欢迎创建与此规范兼容的替代内核。唯一的要求是通过 Rule-ROM PKI 验证以保持确定性。

---

##  systolic 筛网：技术宣言 📜⚙️

**现代计算架构被物流过载。** 在经典 AI 中，数据不断被复制、路由和排队等待处理器 (Bus Contention)。这产生了无法通过软件修复的延迟。

**Decima8 是 systolic 筛网。** 这里没有路由，没有队列，没有"决策"。只有几何和共振。

### 1. 两相脉冲 (The Heartbeat)

系统如同时钟控制的灌溉网络工作。

- **PHASE_READ:** 数据 ("水") 填充公共 VSB 总线。所有 tiles 同时访问这个流。这是最大熵的时刻。
- **PHASE_WRITE:** "识别"输入模式的 tiles 锁存 (Latch) 并将它们的 ID 传输到专用通道，或者 — 如果设置了 BUS_W 标志 — 将它们的权重输出回总线用于下一个 cascade。

### 2. 筛分机制

Tile 在这个系统中不是计算器，而是筛网的单元格。

- 输入数据流飞过 substrate。
- 如果数据流的配置与 tile 中"孔"的几何形状匹配 (配置的阈值和权重)，发生物理共振。
- Tile 瞬间触发，不浪费时间进行算术求和操作。这就是确定性：要么模式通过筛网，要么没有。

### 3. 无路由器的层次结构

我们使用"父 - 子"层次结构代替复杂的网络协议。

- 上层节点 (Conductor/Father) 为下层节点 (Islands/Children) 打开"闸门" (激活图)。
- 数据总是公共的，但对数据的访问由网络本身的结构控制。这消除了冲突，允许达到 20-40 µs 的周期。

### 4. 合成目标

Systolic 筛网将市场噪音、视频流或音频信号转换为 **Pattern ID**。我们不"分析"数据 — 我们从中筛除多余，只留下纯净的 **Intent**。

---

### v0.2 关键原则

| 原则 | 描述 |
|------|------|
| **Level16** | 8 条物理线路，每条传输 16 级离散状态 (0..15) |
| **双向 VSB** | PHASE_READ: Conductor 设置输入; PHASE_WRITE: Island 驱动总线 |
| **Tile = 最小实体** | RuleROM 直接寻址图块 |
| **BUS16 (8 lane)** | 所有数据通过公共总线，邻居不传输数据 |
| **激活图 (Gating)** | 邻居仅形成逻辑选通 (Strobe) 以允许从公共总线读取 |
| **范围熔丝 (Latch)** | 硬件级锁存：仅当电平处于 [lo..hi] 范围内时激活 |
| **Decay-to-Zero** | 累加器在每个周期确定性衰减至 0，无残留状态 |
| **分支坍缩** | 非活动图块重置为 0 |

---

## 🏗 架构

### 组件

```
┌─────────────────────────────────────────────────────┐
│  Conductor (Digital Island)                         │
│  - CPU / 仿真器                                     │
│  - 设置 VSB_INGRESS                                 │
│  - WRITE 后读取 BUS16                               │
│  - 控制 EV_FLASH / EV_RESET / EV_BAKE               │
└─────────────────────────────────────────────────────┘
                         │
                         │ VSB[0..7] + BUS16[0..7]
                         ▼
┌─────────────────────────────────────────────────────┐
│  Island / Swarm (Analog Core)                       │
│  ┌─────────────────────────────────────────────┐    │
│  │  图块阵列 (16×16 = 256)                     │    │
│  │  ┌─────┬─────┬─────┐                        │    │
│  │  │ Tile│ Tile│ ... │                        │    │
│  │  ├─────┼─────┼─────┤  每个图块：            │    │
│  │  │ ... │ ... │ ... │  - 8 输入/输出 lanes   │    │
│  │  └─────┴─────┴─────┘  - FUSE (thr/lock)     │    │
│  │         │                - 权重 8×8          │    │
│  └─────────┼───────────────────────────────────┘    │
│             │                                        │
│  ┌──────────▼──────────────────────────────────┐    │
│  │  BUS16 (公共总线 8 lane)                    │    │
│  │  诚实地累加贡献                             │    │
│  └─────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────┘
```

---

## 📚 文档

### 俄语版
- **概述：** https://decima.rulerom.com/ru/
- **架构：** https://decima.rulerom.com/ru/arch/
- **规范：** https://decima.rulerom.com/ru/spec/

### English Version
- **Overview:** https://decima.rulerom.com/en/
- **Architecture:** https://decima.rulerom.com/en/arch/
- **Specification:** https://decima.rulerom.com/en/spec/

### 中文版
- **概述：** https://decima.rulerom.com/zh/
- **架构：** https://decima.rulerom.com/zh/arch/
- **规范：** https://decima.rulerom.com/zh/spec/

### 章节

| 章节 | 描述 |
|------|------|
| **[图块架构](arch/tiles.md)** | 图块模型、FUSE-LOCK、ACTIVE 闭包 |
| **[BUS16](arch/bus.md)** | 诚实地累加、CLIP/OVF 标志 |
| **[READ/WRITE 阶段](arch/phase.md)** | 标准周期 EV_FLASH |
| **[路由](arch/routing.md)** | 激活图、RoutingFlags16 |
| **[Bake TLV](spec/bake.md)** | 二进制烘焙格式 |
| **[协议](spec/protocol.md)** | EV_FLASH、EV_RESET、UDP |
| **[IDE](tools/ide.md)** | 可视化烘焙环境 |

---

## 🛠️ 快速开始

### 运行文档

```bash
# 俄语版
cd ru
mkdocs serve

# 英语版
cd en
mkdocs serve

# 中文版
cd zh
mkdocs serve
```

### 项目结构

```
decima/
├── ru/                     # 俄语文档
│   ├── mkdocs.yml
│   └── docs/
├── en/                     # English documentation
│   ├── mkdocs.yml
│   └── docs/
├── zh/                     # 中文文档
│   ├── mkdocs.yml
│   └── docs/
├── old/                    # 归档材料
├── README.md
├── llms.txt
└── ai-plugin.json
```

---

## 🔄 标准周期 (EV_FLASH)

```mermaid
graph LR
    A[Setup: VSB_INGRESS] --> B[PHASE_READ]
    B --> C[TURNAROUND]
    C --> D[PHASE_WRITE]
    D --> E[READOUT_SAMPLE]
    E --> F[INTERPHASE_AUTORESET]
```

| 阶段 | 描述 |
|------|------|
| **PHASE_READ** | 图块采样输入，更新运行时 |
| **TURNAROUND** | Conductor: Hi-Z, Island: 准备驱动 |
| **PHASE_WRITE** | Island 驱动 BUS16 |
| **READOUT_SAMPLE** | Conductor 读取 BUS16[0..7] |
| **AUTORESET** | 可选的域重置 |

---

## 📊 硬件常量 v0.2

| 常量 | 值 |
|------|-----|
| **VSB** | 8 条数据线路 VSB[0..7] |
| **BUS16** | 8 lane, WRITE 时累加 |
| **Domains** | 16 个域 (0..15) |
| **Level16** | 0..15 (4 比特) |
| **RoutingFlags16** | 10 比特：N,E,S,W,NE,SE,SW,NW,BUS_R,BUS_W |

---

## 🔗 生态系统

| 项目 | 描述 | URL |
|------|------|-----|
| **🌿 Intent-Garden** | 确定性 C/C++ 验证引擎 | https://intent-garden.org |
| **📜 Rule-Rom** | 全球意图库 | https://rulerom.com |
| **🏛️ Swarm Council** | 16 位长老在集群核心 | https://intent-garden.org/swarm.html |
| **🧬 Personality Lab** | 神经形态个性烘焙坊 | https://intent-garden.org/bakery.html |

---

## 📧 联系方式

- **文档：** https://decima.rulerom.com
- **Email:** vsb@decima8.org

---

## 💰 支持

- **Boosty:** https://boosty.to/intentgarden

---

**Bake the Future. Build the Substrate.** 🛠️⚡️
