# 01 — 概念阶段（Concept Phase）

## 1. 阶段概述

概念阶段是 IPD 流程的起点，核心任务是：
- 分析市场需求与技术可行性
- 定义产品概念与关键特性
- 评估项目风险与资源需求
- 输出产品需求文档（PRD）与项目可行性分析

**决策评审点**：概念决策评审（CDCP）

---

## 2. 香山项目概念阶段分析

### 2.1 市场需求分析

#### 2.1.1 RISC-V 生态背景

| 维度 | 分析 |
|------|------|
| 指令集趋势 | RISC-V 成为继 x86、ARM 之后的第三大 ISA，开源免授权费 |
| 高性能需求 | AI、数据中心、边缘计算对高性能开源处理器的迫切需求 |
| 供应链安全 | 地缘政治背景下，自主可控处理器成为战略需求 |
| 生态成熟度 | GCC/LLVM/Linux/BSD 对 RISC-V 支持完善，工具链成熟 |

#### 2.1.2 竞争格局

| 项目 | 特点 | 定位 |
|------|------|------|
| SiFive P550/P650 | 商业高性能 RISC-V IP | 高端应用处理器 |
| T-Head C920 | 玄铁系列，嵌入式向服务器扩展 | 物联网/服务器 |
|香山 Kunminghu | 学术级开源，最先进微架构 | 研究/教育/产业参考 |

#### 2.1.3 目标用户

- **学术研究机构**：微架构创新验证平台
- **芯片设计公司**：处理器 IP 参考设计
- **系统厂商**：高性能 SoC 核心
- **教育培训**：计算机体系结构教学与实验

### 2.2 产品概念定义

#### 2.2.1 产品定位

> **开源、高性能、可综合的 RISC-V 处理器芯片设计，面向学术研究与产业参考。**

#### 2.2.2 核心特性（Key Features）

| 特性 | 描述 | 技术挑战 |
|------|------|---------|
| 乱序执行 | 6-wide 超标量乱序流水线 | 重命名、调度、唤醒逻辑复杂度 |
| 高级分支预测 | TAGE-SC + ITTAGE + FTB + RAS | 多预测器协同、面积/功耗平衡 |
| 大缓存层次 | L1 + L2(Inclusive) + L3/LLC | 一致性协议、预取算法 |
| 向量计算 | RVV 1.0, VLEN=128 | 向量流水线、掩码处理 |
| 虚拟化 | H-extension, Sv39/Sv48 | 两级 TLB、嵌套分页 |
| 多核扩展 | CHI 一致性，支持 SMP | NoC 设计、缓存一致性 |
| 可配置性 | 参数化设计，支持多种配置 | Chisel 参数化、配置系统 |

#### 2.2.3 关键性能指标（KPI）

| 指标 | 目标值 | 对标 |
|------|--------|------|
| 主频 | ≥ 2.0 GHz（28nm） | ARM A76 |
| IPC | ≥ 1.5（SPECint） | ARM A76 |
| 核心面积 | ≤ 3mm²（28nm） | ARM A76 |
| 功耗 | ≤ 1W @ 2GHz | ARM A76 |
| 流水线级数 | 13+ 级 | 同类乱序核 |

### 2.3 技术可行性分析

#### 2.3.1 关键技术风险评估

| 风险项 | 风险等级 | 缓解措施 |
|--------|---------|---------|
| Chisel 工具链成熟度 | 中 | 基于 Chisel 6.7.0 稳定版本，社区活跃 |
| 乱序执行正确性验证 | 高 | difftest 框架 + 形式验证 + 压力测试 |
| CHI 一致性协议复杂度 | 高 | 分阶段实现，先 TL 后 CHI |
| 向量扩展验证 | 中 | 使用 Spike 参考模型对比 |
| 物理设计收敛 | 中 | 与代工厂合作，使用成熟工艺 |

#### 2.3.2 技术路线选择

| 决策点 | 选择 | 理由 |
|--------|------|------|
| 硬件描述语言 | Chisel (Scala) | 参数化能力强，开发效率高，开源生态 |
| 构建工具 | Mill | 比 SBT 更快的增量编译，适合大型项目 |
| 仿真器 | Verilator + VCS | 开源+商业双覆盖 |
| 一致性协议 | CHI → TileLink | CHI 面向多核，TL 用于单核 |
| 参考模型 | NEMU + Spike | NEMU 用于 difftest， Spike 用于 ISA 验证 |

### 2.4 资源与计划

#### 2.4.1 团队结构

```
香山项目团队
├── 架构组（Architecture）
│   ├── 微架构定义与参数化
│   ├── 性能建模与仿真
│   └── ISA 扩展定义
├── 前端组（Frontend）
│   ├── 取指与分支预测
│   ├── ICache 设计
│   └── 预取算法
├── 后端组（Backend）
│   ├── 译码与重命名
│   ├── 发射与调度
│   ├── 执行单元
│   └── 提交与异常处理
├── 存储组（Memory）
│   ├── DCache 设计
│   ├── LSQ 与内存一致性
│   ├── TLB/MMU
│   └── 预取器
├── 验证组（Verification）
│   ├── difftest 框架
│   ├── 单元测试
│   ├── 系统集成测试
│   └── 性能回归测试
└── 物理设计组（Physical Design）
    ├── 综合与 DFT
    ├── 布局布线
    └── 时序收敛
```

#### 2.4.2 里程碑规划

| 里程碑 | 时间 | 交付物 |
|--------|------|--------|
| M1: 基础流水线 | T+3月 | 顺序执行核，通过基础测试 |
| M2: 乱序执行 | T+6月 | 乱序核，通过 difftest |
| M3: 缓存层次 | T+9月 | L1+L2，通过性能测试 |
| M4: 向量扩展 | T+12月 | RVV 1.0，通过向量测试 |
| M5: 多核一致性 | T+15月 | CHI 一致性，SMP Linux 启动 |
| M6: 流片版本 | T+18月 | GDSII，回片测试 |

---

## 3. 概念阶段交付物

### 3.1 产品需求文档（PRD）摘要

```yaml
product_name: XiangShan Kunminghu V2
product_type: Open-source RISC-V Processor Core
target_process: 28nm CMOS
isa: RV64GCBMV + H-extension + Custom Extensions
microarchitecture:
  pipeline: Out-of-order, 13+ stages
  fetch_width: 8 instructions/cycle
  decode_width: 6 instructions/cycle
  rename_width: 6 instructions/cycle
  issue_width: 6 instructions/cycle
  commit_width: 8 instructions/cycle
  rob_size: 160 entries
  physical_registers: 192 (Int) + 192 (FP) + 128 (VF)
cache_hierarchy:
  L1_I: 64KB, 4-way, 2-cycle latency
  L1_D: 64KB, 4-way, 2-cycle latency
  L2: 1MB, 8-way, inclusive, 4 banks
  L3: 16MB, 16-way, non-inclusive, 4 banks (optional)
branch_prediction:
  - FTB (Fetch Target Buffer)
  - TAGE-SC (Tagged Geometric History)
  - ITTAGE (Indirect Target Prediction)
  - RAS (Return Address Stack)
  - UBTB (Unconditional Branch Target Buffer)
memory_system:
  load_queue: 72 entries
  store_queue: 56 entries
  store_buffer: 16 entries
  prefetch: SMS + BOP + TP
coherence:
  protocol: CHI Issue E.b
  interconnect: NoC (Network on Chip)
  llc: OpenLLC (Last Level Cache)
virtualization:
  extensions: H-extension
  paging: Sv39/Sv48
  tlb: L1 TLB + L2 TLB (two-level)
vector:
  specification: RVV 1.0
  vlen: 128 bits
  elen: 64 bits
```

### 3.2 可行性分析报告

| 评估维度 | 结论 | 说明 |
|----------|------|------|
| 技术可行性 | ✅ 可行 | Chisel 生态成熟，有 SiFive/U.C. Berkeley 成功案例 |
| 资源可行性 | ✅ 可行 | 团队具备完整处理器设计经验 |
| 时间可行性 | ✅ 可行 | 18个月周期合理，分阶段交付降低风险 |
| 经济可行性 | ✅ 可行 | 开源模式，无直接商业压力 |
| 生态可行性 | ✅ 可行 | RISC-V 工具链完善，操作系统支持成熟 |

### 3.3 风险登记册

| ID | 风险描述 | 概率 | 影响 | 风险等级 | 应对策略 |
|----|---------|------|------|---------|---------|
| R01 | Chisel 编译器版本升级导致不兼容 | 中 | 高 | 高 | 锁定版本，建立升级测试流程 |
| R02 | 乱序执行验证覆盖率不足 | 高 | 高 | 高 | 加强 difftest，引入形式验证 |
| R03 | CHI 一致性协议死锁 | 中 | 高 | 高 | 协议仿真验证，死锁检测 |
| R04 | 物理设计时序不收敛 | 中 | 中 | 中 | 早期时序预算，多轮迭代 |
| R05 | 关键人员流失 | 低 | 高 | 中 | 知识文档化，交叉培训 |
| R06 | 子模块版本冲突 | 中 | 中 | 中 | 严格版本管理，CI 集成测试 |

---

## 4. 概念阶段决策评审（CDCP）

### 4.1 评审检查清单

- [x] 市场需求明确，目标用户清晰
- [x] 产品概念完整，核心特性定义清楚
- [x] 技术可行性得到验证
- [x] 资源需求评估完成
- [x] 风险识别与应对策略就绪
- [x] 里程碑计划合理
- [x] 团队结构完整

### 4.2 评审结论

**通过** — 项目进入计划阶段。

---

## 5. 参考资源

| 资源 | 链接/位置 |
|------|----------|
| 香山项目官网 | https://xiangshan.cc |
| 设计文档 | https://docs.xiangshan.cc |
| GitHub 仓库 | https://github.com/OpenXiangShan/XiangShan |
| RISC-V 规范 | https://riscv.org/technical/specifications/ |
| Chisel 文档 | https://www.chisel-lang.org/ |
| IPD 方法论 | 《华为研发》— 张利华 |

---

*文档版本：v1.0*
*最后更新：2026-10-19*
