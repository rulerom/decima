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

### v0.2 关键原则

| 原则 | 描述 |
|------|------|
| **Level16** | 8 条线路每条传输 0..15 的数据 |
| **双向 VSB** | Conductor 在 READ 前设置输入，Island 在 WRITE 时驱动 |
| **Tile = 最小实体** | RuleROM 直接寻址图块 |
| **BUS16 (8 lane)** | 所有数据通过公共总线，邻居不传输数据 |
| **激活图** | 邻居形成继电器以读取 BUS |
| **范围熔丝** | LOCK 如果 thr_cur16 ∈ [thr_lo16..thr_hi16] |
| **Decay-to-Zero** | 累加器衰减到 0，永不跳过 |
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
