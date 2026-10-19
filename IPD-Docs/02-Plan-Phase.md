# 02 — 计划阶段（Plan Phase）

## 1. 阶段概述

计划阶段的核心任务是：
- 将产品概念转化为详细的技术规格
- 制定详细的项目计划与资源分配
- 定义系统架构与模块划分
- 建立验证策略与测试计划

**决策评审点**：计划决策评审（PDCP）

---

## 2. 系统架构设计

### 2.1 顶层架构

香山昆明湖 V2 采用自顶向下的层次化设计：

```
┌─────────────────────────────────────────────────────────────┐
│                        XSTop (顶层)                          │
│  ┌──────────────────────────────────────────────────────┐   │
│  │                    SoC (系统级)                       │   │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  │   │
│  │  │  XSTile #0  │  │  XSTile #1  │  │  XSTile #N  │  │   │
│  │  │ ┌─────────┐ │  │ ┌─────────┐ │  │ ┌─────────┐ │  │   │
│  │  │ │ XSCore  │ │  │ │ XSCore  │ │  │ │ XSCore  │ │  │   │
│  │  │ │(Frontend│ │  │ │(Frontend│ │  │ │(Frontend│ │  │   │
│  │  │ │+Backend │ │  │ │+Backend │ │  │ │+Backend │ │  │   │
│  │  │ │+MemBlk) │ │  │ │+MemBlk) │ │  │ │+MemBlk) │ │  │   │
│  │  │ └────┬────┘ │  │ └────┬────┘ │  │ └────┬────┘ │  │   │
│  │  │      │      │  │      │      │  │      │      │  │   │
│  │  │ ┌────┴────┐ │  │ ┌────┴────┐ │  │ ┌────┴────┐ │  │   │
│  │  │ │  L2     │ │  │ │  L2     │ │  │ │  L2     │ │  │   │
│  │  │ │ (Incl.) │ │  │ │ (Incl.) │ │  │ │ (Incl.) │ │  │   │
│  │  │ └────┬────┘ │  │ └────┬────┘ │  │ └────┬────┘ │  │   │
│  │  └──────┼──────┘  └──────┼──────┘  └──────┼──────┘  │   │
│  │         │                │                │         │   │
│  │  ┌──────┴────────────────┴────────────────┴──────┐  │   │
│  │  │           NoC Interconnect (CHI)              │  │   │
│  │  └──────────────────┬───────────────────────────┘  │   │
│  │                     │                              │   │
│  │  ┌──────────────────┴───────────────────────────┐  │   │
│  │  │              L3 / LLC (OpenLLC)               │  │   │
│  │  └──────────────────────────────────────────────┘  │   │
│  └──────────────────────────────────────────────────────┘   │
│  ┌──────────────────────────────────────────────────────┐   │
│  │              Devices (UART, Timer, PLIC, ...)        │   │
│  └──────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

### 2.2 核心微架构流水线

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        XiangShan Kunminghu V2 流水线                    │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  Stage 1-2: 取指 (Fetch)                                                │
│  ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐          │
│  │  PC Gen  │───▶│  ICache  │───▶│  PreDec  │───▶│   IBuf   │          │
│  │  + BPU   │    │  + TLB   │    │  + RAS   │    │ (48 ent) │          │
│  └──────────┘    └──────────┘    └──────────┘    └──────────┘          │
│       │                                                           │
│       ▼                                                           │
│  Stage 3: 译码 (Decode)                                             │
│  ┌──────────┐    ┌──────────┐    ┌──────────┐                       │
│  │  Decode  │───▶│  VecDec  │───▶│  Inst    │                       │
│  │  (6-wide)│    │          │    │  Split   │                       │
│  └──────────┘    └──────────┘    └──────────┘                       │
│       │                                                           │
│       ▼                                                           │
│  Stage 4: 重命名 (Rename)                                           │
│  ┌──────────┐    ┌──────────┐    ┌──────────┐                       │
│  │  FreeList│───▶│  Rename  │───▶│  BusyTab │                       │
│  │  + Disp  │    │  Table   │    │  Update  │                       │
│  └──────────┘    └──────────┘    └──────────┘                       │
│       │                                                           │
│       ▼                                                           │
│  Stage 5-6: 发射 (Issue)                                            │
│  ┌──────────┐    ┌──────────┐    ┌──────────┐                       │
│  │  Issue   │───▶│  Wakeup  │───▶│  Bypass  │                       │
│  │  Queue   │    │  + Select│    │  Network │                       │
│  └──────────┘    └──────────┘    └──────────┘                       │
│       │                                                           │
│       ▼                                                           │
│  Stage 7-10: 执行 (Execute)                                         │
│  ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐      │
│  │  RegFile │───▶│    EXU   │───▶│  FU      │───▶│  WB      │      │
│  │  Read    │    │ (ALU/BRU)│    │(FPU/Vec) │    │  + Bypass│      │
│  └──────────┘    └──────────┘    └──────────┘    └──────────┘      │
│       │                                                           │
│       ▼                                                           │
│  Stage 11-13: 提交 (Commit)                                        │
│  ┌──────────┐    ┌──────────┐    ┌──────────┐                       │
│  │   ROB    │───▶│  RAB     │───▶│  Commit  │                       │
│  │  (160)   │    │  Update  │    │  (8-wide)│                       │
│  └──────────┘    └──────────┘    └──────────┘                       │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 2.3 模块划分与接口定义

#### 2.3.1 前端模块（Frontend）

| 模块 | 功能 | 关键参数 |
|------|------|---------|
| `Frontend` | 前端顶层集成 | - |
| `BPU` | 分支预测顶层 | 多预测器协同 |
| `FTB` | 取指目标缓冲 | 2048 entries, 4-way |
| `Tage` | TAGE-SC 预测器 | 4 tables, 4096 entries each |
| `ITTAGE` | 间接跳转预测 | 5 tables |
| `RAS` | 返回地址栈 | 16 entries |
| `FauFTB` | 快速 FTB | 256 entries |
| `ICache` | 指令缓存 | 64KB, 4-way |
| `IBuffer` | 指令缓冲 | 48 entries, 6 banks |
| `PreDecode` | 预译码 | 识别分支/跳转 |
| `NewFtq` | 取指传输队列 | 64 entries |

#### 2.3.2 后端模块（Backend）

| 模块 | 功能 | 关键参数 |
|------|------|---------|
| `Backend` | 后端顶层集成 | - |
| `DecodeUnit` | 译码单元 | 6-wide |
| `RenameTable` | 重命名表 | 32 logical regs |
| `FreeList` | 空闲列表 | 192 physical regs |
| `BusyTable` | 忙表 | 跟踪寄存器状态 |
| `IssueQueue` | 发射队列 | 24 entries x 4 schedulers |
| `ROB` | 重排序缓冲 | 160 entries |
| `RAB` | 重命名备份 | 256 entries |
| `RegFile` | 寄存器堆 | 192 Int + 192 FP + 128 VF |

#### 2.3.3 执行单元（Execution Units）

| 执行单元 | 功能 | 流水线 |
|----------|------|--------|
| ALU0-3 | 整数运算 | 1-2 级 |
| BJU0-3 | 分支/跳转 | 1-3 级 |
| FEX0-4 | 浮点运算 | 3-8 级 |
| VFEX0-4 | 向量运算 | 3-8 级 |
| LDU0-2 | Load 单元 | 3 级 |
| STA0-1 | Store 地址 | 2 级 |
| STD0-1 | Store 数据 | 2 级 |
| VLSU0-1 | 向量 Load/Store | 可变 |

#### 2.3.4 存储系统（Memory System）

| 模块 | 功能 | 关键参数 |
|------|------|---------|
| `MemBlock` | 存储块顶层 | - |
| `DCache` | 数据缓存 | 64KB, 4-way, SECDED |
| `LoadQueue` | Load 队列 | 72 entries |
| `StoreQueue` | Store 队列 | 56 entries |
| `SBuffer` | Store Buffer | 16 entries |
| `L2TLB` | 二级 TLB | 48 entries |
| `PTW` | 页表遍历 | 硬件实现 |
| `Prefetcher` | 预取器 | SMS + BOP + TP |

### 2.4 配置系统设计

香山采用 Chisel 的 CDE（Configurable Design Environment）实现参数化配置：

```scala
// 配置层次结构
BaseConfig                    // 基础配置
├── MinimalConfig             // 最小可综合配置
├── DefaultConfig             // 默认配置
├── KunminghuV2Config         // 昆明湖 V2 配置
├── XSNoCTopConfig            // NoC 顶层配置
├── FpgaDefaultConfig         // FPGA 配置
└── FpgaDiffDefaultConfig     // FPGA Difftest 配置
```

#### 2.4.1 关键配置参数

| 参数类别 | 参数名 | 默认值 | 说明 |
|----------|--------|--------|------|
| 流水线宽度 | `FetchWidth` | 8 | 每周期取指数 |
| | `DecodeWidth` | 6 | 每周期译码数 |
| | `RenameWidth` | 6 | 每周期重命名数 |
| | `CommitWidth` | 8 | 每周期提交数 |
| 缓存 | `ICache` | 64KB, 4-way | L1 指令缓存 |
| | `DCache` | 64KB, 4-way | L1 数据缓存 |
| | `L2` | 1MB, 8-way | L2 缓存 |
| | `L3` | 16MB, 16-way | L3 缓存 |
| 分支预测 | `FtbSize` | 2048 | FTB 大小 |
| | `TageTableInfos` | 4 tables | TAGE 表配置 |
| | `RasSize` | 16 | RAS 大小 |
| 内存系统 | `RobSize` | 160 | ROB 大小 |
| | `LoadQueueSize` | 72 | Load 队列大小 |
| | `StoreQueueSize` | 56 | Store 队列大小 |
| 向量 | `VLEN` | 128 | 向量长度 |
| | `ELEN` | 64 | 元素长度 |

---

## 3. 项目计划

### 3.1 开发阶段划分

| 阶段 | 时间 | 主要任务 | 交付物 |
|------|------|---------|--------|
| P1: 基础设施 | T+1月 | 搭建开发环境、CI/CD、代码规范 | 开发指南、CI 配置 |
| P2: 前端开发 | T+2~4月 | 取指、分支预测、ICache | 前端 RTL + 单元测试 |
| P3: 后端开发 | T+3~6月 | 译码、重命名、发射、执行 | 后端 RTL + 单元测试 |
| P4: 存储系统 | T~5~7月 | DCache、LSQ、TLB、预取 | 存储 RTL + 单元测试 |
| P5: 系统集成 | T+7~9月 | 模块集成、接口对齐 | 完整 RTL |
| P6: 验证调试 | T+8~12月 | difftest、性能调优 | 验证报告 |
| P7: 物理设计 | T+11~15月 | 综合、P&R、时序收敛 | GDSII |
| P8: 流片 | T+15~18月 | 流片、封装、测试 | 样片 |

### 3.2 资源分配

| 角色 | 人数 | 职责 |
|------|------|------|
| 项目总监 | 1 | 整体规划、资源协调、风险管理 |
| 架构师 | 2 | 微架构定义、性能建模、技术决策 |
| 前端工程师 | 3 | 取指、分支预测、ICache |
| 后端工程师 | 4 | 译码、重命名、发射、执行 |
| 存储工程师 | 3 | DCache、LSQ、TLB、预取 |
| 验证工程师 | 4 | difftest、单元测试、系统测试 |
| 物理设计工程师 | 2 | 综合、P&R、时序 |
| 工具工程师 | 1 | 开发环境、CI/CD、工具链 |

### 3.3 依赖管理

```
XiangShan (主仓库)
├── rocket-chip          # RISC-V 基础库 (Diplomacy, TL, AXI4)
├── difftest             # 仿真验证框架
├── huancun              # L2/L3 缓存
├── fudian               # 浮点运算单元
├── yunsuan              # 向量/ALU 运算库
├── coupledL2            # Coupled L2 缓存
├── openLLC              # OpenLLC (CHI LLC)
├── ChiselAIA            # AIA/IMSIC/APLIC 中断
├── ChiselIOPMP          # I/O PMP
├── utility              # 工具库
└── ready-to-run         # 预编译仿真二进制
```

---

## 4. 验证策略

### 4.1 验证层次

```
┌─────────────────────────────────────────────────────────────┐
│                    System-Level Verification                 │
│  (Linux Boot, SPEC CPU, Application Workloads)              │
├─────────────────────────────────────────────────────────────┤
│                  Integration Verification                    │
│  (Full-chip Simulation, Difftest, Performance Testing)      │
├─────────────────────────────────────────────────────────────┤
│                   Unit Verification                          │
│  (Module-level Tests, ChiselTest, Formal Verification)      │
├─────────────────────────────────────────────────────────────┤
│                   Block Verification                         │
│  (Sub-system Tests, Interface Checks, Protocol Checks)      │
└─────────────────────────────────────────────────────────────┘
```

### 4.2 验证方法

| 方法 | 工具 | 覆盖目标 |
|------|------|---------|
| 单元测试 | ChiselTest | 模块功能正确性 |
| 差分测试 | difftest + NEMU | 指令级正确性 |
| 形式验证 | SymbiYosys | 关键属性验证 |
| 性能仿真 | Verilator + VCS | 性能回归 |
| 硬件仿真 | FPGA (Palladium) | 系统级验证 |
| 代码覆盖 | Verilator --coverage | 覆盖率度量 |
| 功能覆盖 | 自定义 coverpoint | 功能点覆盖 |

### 4.3 验证计划

| 阶段 | 验证内容 | 通过标准 |
|------|---------|---------|
| 单元测试 | 各模块功能 | 代码覆盖率 > 90% |
| 集成测试 | 模块间接口 | difftest 通过 |
| 系统测试 | 全芯片功能 | Linux 启动 |
| 性能测试 | 性能回归 | 不低于基线 |
| 压力测试 | 极端场景 | 无死锁/挂死 |

---

## 5. 风险管理计划

| 风险 | 概率 | 影响 | 应对措施 | 负责人 |
|------|------|------|---------|--------|
| 验证覆盖率不足 | 高 | 高 | 增加测试用例，引入形式验证 | 验证组 |
| 时序不收敛 | 中 | 高 | 早期时序预算，多轮迭代 | 物理设计组 |
| 子模块版本冲突 | 中 | 中 | 严格版本管理，CI 集成测试 | 工具组 |
| 关键路径延期 | 中 | 高 | 关键路径监控，资源调配 | 项目总监 |
| 需求变更 | 低 | 中 | 变更控制流程 | 架构组 |

---

## 6. 质量保证

### 6.1 代码规范

| 规范 | 工具 | 检查内容 |
|------|------|---------|
| 代码格式 | scalafmt | 缩进、空格、换行 |
| 代码风格 | scalastyle | 命名、复杂度、注释 |
| 命名约定 | 人工评审 | 模块/信号命名一致性 |

### 6.2 CI/CD 流程

```yaml
# .github/workflows/emu.yml 主要步骤
1. 代码格式检查 (scalafmt)
2. 编译检查 (mill compile)
3. Verilog 生成检查 (make verilog)
4. EMU 基础测试 (make emu)
5. EMU CHI 测试
6. EMU 性能测试 (SPEC06)
7. EMU 多核测试 (SMP)
8. SIMV 基础测试 (VCS)
9. Docker 镜像构建
10. 子模块版本检查
```

### 6.3 评审机制

| 评审类型 | 频率 | 参与者 | 内容 |
|----------|------|--------|------|
| 代码评审 | 每次提交 | 模块负责人 | 代码质量、风格 |
| 设计评审 | 里程碑 | 架构组 | 架构合理性 |
| 验证评审 | 阶段结束 | 验证组 | 覆盖率、通过率 |
| 项目评审 | 每周 | 全团队 | 进度、风险 |

---

## 7. 计划阶段决策评审（PDCP）

### 7.1 评审检查清单

- [x] 系统架构设计完成
- [x] 模块划分与接口定义完成
- [x] 配置系统设计完成
- [x] 项目计划制定完成
- [x] 资源分配完成
- [x] 验证策略制定完成
- [x] 风险管理计划完成
- [x] 质量保证措施就绪

### 7.2 评审结论

**通过** — 项目进入开发阶段。

---

## 8. 附录

### 8.1 术语表

| 术语 | 全称 | 说明 |
|------|------|------|
| BPU | Branch Prediction Unit | 分支预测单元 |
| FTB | Fetch Target Buffer | 取指目标缓冲 |
| TAGE | Tagged Geometric History Length | 标签几何历史长度预测器 |
| ITTAGE | Indirect Target TAGE | 间接跳转 TAGE 预测器 |
| RAS | Return Address Stack | 返回地址栈 |
| ROB | Reorder Buffer | 重排序缓冲 |
| RAB | Rename Allocation Buffer | 重命名分配缓冲 |
| LSQ | Load/Store Queue | Load/Store 队列 |
| SBuffer | Store Buffer | Store 缓冲 |
| TLB | Translation Lookaside Buffer | 页表缓存 |
| PTW | Page Table Walker | 页表遍历器 |
| CHI | Coherent Hub Interface | ARM 一致性接口协议 |
| NoC | Network on Chip | 片上网络 |
| LLC | Last Level Cache | 末级缓存 |

### 8.2 参考文档

| 文档 | 位置 |
|------|------|
| 香山设计文档 | https://docs.xiangshan.cc |
| Chisel 文档 | https://www.chisel-lang.org |
| RISC-V 规范 | https://riscv.org |
| rocket-chip 文档 | https://github.com/chipsalliance/rocket-chip |

---

*文档版本：v1.0*
*最后更新：2026-10-19*
