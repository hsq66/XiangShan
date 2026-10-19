# 02 — 计划阶段（Plan Phase）深度文档

## 1. 阶段概述

计划阶段的核心任务是：
- 将概念阶段的产品需求转化为详细的技术规格说明书（PRS）
- 完成系统架构设计与模块划分
- 定义 Chisel 参数化配置系统与模块间接口
- 制定项目计划、资源分配与验证策略

**决策评审点**：计划决策评审（PDCP）

---

## 2. 系统架构设计

### 2.1 顶层架构（基于 Top.scala 与 BaseXSSoc）

香山昆明湖 V2 采用 **LazyModule + Diplomacy** 的层次化架构，基于 rocket-chip 的 Diplomacy 框架构建：

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           XSTop (LazyModule)                             │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                       SoC (BaseXSSoc)                           │   │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │   │
│  │  │   XSTile #0  │  │   XSTile #1  │  │   XSTile #N  │          │   │
│  │  │  ┌─────────┐ │  │  ┌─────────┐ │  │  ┌─────────┐ │          │   │
│  │  │  │ XSCore  │ │  │  │ XSCore  │ │  │  │ XSCore  │ │          │   │
│  │  │  │         │ │  │  │         │ │  │  │         │ │          │   │
│  │  │  │ Frontend│ │  │  │ Frontend│ │  │  │ Frontend│ │          │   │
│  │  │  │ +Backend│ │  │  │ +Backend│ │  │  │ +Backend│ │          │   │
│  │  │  │ +MemBlk │ │  │  │ +MemBlk │ │  │  │ +MemBlk │ │          │   │
│  │  │  └────┬────┘ │  │  └────┬────┘ │  │  └────┬────┘ │          │   │
│  │  │   ┌────┴────┐ │  │   ┌────┴────┐ │  │   ┌────┴────┐ │      │   │
│  │  │   │   L2    │ │  │   │   L2    │ │  │   │   L2    │ │      │   │
│  │  │   │(Inclusive)│ │  │   │(Inclusive)│ │  │   │(Inclusive)│ │   │   │
│  │  │   └────┬────┘ │  │   └────┬────┘ │  │   └────┬────┘ │      │   │
│  │  └────────┼──────┘  └────────┼──────┘  └────────┼──────┘      │   │
│  │           │                  │                  │              │   │
│  │  ┌────────┴──────────────────┴──────────────────┴────────┐    │   │
│  │  │           NoC Interconnect (CHI Issue E.b)            │    │   │
│  │  └──────────────────┬────────────────────────────────────┘    │   │
│  │                     │                                           │   │
│  │  ┌──────────────────┴────────────────────────────────────┐    │   │
│  │  │              L3 / LLC (OpenLLC, OpenNCB)              │    │   │
│  │  └────────────────────────────────────────────────────────┘    │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │              Peripheral Xbar (TileLink)                         │   │
│  │  UART, Timer, PLIC, RAM, Flash, VGA, Keyboard, Debug...        │   │
│  └─────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────┘
```

### 2.2 模块化设计原则（基于 rocket-chip Diplomacy）

| 设计原则 | 实现方式 | 代码体现 |
|---------|---------|---------|
| LazyModule 模式 | 编译时延迟实例化，支持参数化总线节点 | `src/main/scala/top/Top.scala` `class XSTop extends BaseXSSoc()` |
| Diplomacy 节点 | TileLink/AXI4 总线节点，自动处理端口连接 | `freechips.rocketchip.diplomacy._` |
| 参数化配置 | CDE (Configurable Design Environment) Key-Field 模式 | `org.chipsalliance.cde.config.{Field, Parameters}` |
| 模块内联 (Inlined) | `shouldBeInlined = true` 减少模块层级，优化综合 | `FrontendInlined`, `BackendInlined` |
| DFT 支持 | ResetGen 节点插入，MBIST/SRAM_CTL 接口 | `ResetGenNode`, `SramBroadcastBundle` |

### 2.3 核心微架构流水线（13+ 级）

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    香山昆明湖 V2 乱序执行流水线 (13+ 级)                       │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  S1-S2: 取指阶段 (Fetch)                                                    │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐                    │
│  │  PC Gen  │─▶│  ICache  │─▶│ PreDecode│─▶│   IBuf   │                    │
│  │ + BPU    │  │ + ITLB   │  │ + RAS    │  │ (48 ent) │                    │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘                    │
│       │                          BPU 3 stages: S1/S2/S3                    │
│       ▼ (FTQ: 64 entries)                                                   │
│                                                                             │
│  S3: 译码阶段 (Decode)  6-wide                                              │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐                                  │
│  │  Decode  │─▶│ VecDec   │─▶│ InstSplit│                                  │
│  │ (6-wide) │  │          │  │          │                                  │
│  └──────────┘  └──────────┘  └──────────┘                                  │
│       ▼                                                                     │
│                                                                             │
│  S4: 重命名阶段 (Rename)  6-wide                                            │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐                                  │
│  │ FreeList │─▶│ RenameTab│─▶│ BusyTable│                                  │
│  │ + Dispatch│  │          │  │ Update   │                                  │
│  └──────────┘  └──────────┘  └──────────┘                                  │
│       ▼ (Snapshot: 4 checkpoints)                                           │
│                                                                             │
│  S5-S6: 发射阶段 (Issue)  6-wide                                            │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐                    │
│  │ IssueQ   │─▶│ Wakeup   │─▶│ Select   │─▶│ Bypass   │                    │
│  │ (Int/Fp/Vf/Mem)│       │          │          │ Network                  │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘                    │
│                                                                             │
│  S7-S10: 执行阶段 (Execute)                                                 │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐                    │
│  │ RegFile  │─▶│   EXU    │─▶│   FU     │─▶│   WB     │                    │
│  │ Read     │  │(ALU/BJU) │  │(FPU/Vec) │  │+Bypass   │                    │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘                    │
│                                                                             │
│  S11-S13: 提交阶段 (Commit)  8-wide                                         │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐                                  │
│  │   ROB    │─▶│   RAB    │─▶│  Commit  │                                  │
│  │ (160 ent)│  │ (256 ent)│  │ (8-wide) │                                  │
│  └──────────┘  └──────────┘  └──────────┘                                  │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 3. 参数化配置系统（基于 CDE + Chisel 6.7.0）

### 3.1 配置层次结构

香山采用 rocket-chip 的 CDE (Configurable Design Environment) 实现参数化配置，配置层次如下：

```
BaseConfig (n: Int)                          ← 基础配置 (XLen=64, DebugOptions, SoCParams)
├── MinimalConfig (n=1)                      ← 最小可综合配置 (无 L2, 有 L3)
├── MinimalSimConfig                         ← 仿真专用 (无 DCache, soft PTW)
├── DefaultConfig (n=1)                      ← 默认配置 (L3 16MB + L2 1MB)
├── KunminghuV2Config (n=1)                  ← 昆明湖 V2 配置 (CHI Enabled)
│   ├── L2CacheConfig("1MB", inclusive=true, banks=4, tp=false)
│   ├── DefaultConfig
│   └── WithCHI
├── KunminghuV2MinimalConfig                 ← 昆明湖 V2 最小配置
├── XSNoCTopConfig                           ← NoC 顶层配置
├── XSNoCDiffTopConfig                       ← NoC Difftest 配置
├── FpgaDefaultConfig                        ← FPGA 配置 (L3 3MB)
├── FpgaDiffDefaultConfig                    ← FPGA Difftest 配置
├── FpgaDiffKMHV2Config                      ← FPGA KMHV2 Difftest 配置
├── CVMConfig / CVMTestConfig                ← 内存加密配置
├── MediumConfig (n=1)                       ← 中等配置 (L3 4MB + L2 512KB)
└── FuzzConfig                               ← 模糊测试配置
```

### 3.2 关键配置参数（源自 Parameters.scala）

#### 3.2.1 核心参数 (XSCoreParameters)

| 参数类别 | 参数名 | 默认值 | 说明 |
|---------|--------|--------|------|
| 指令集 | `HasMExtension` | `true` | M 扩展 (乘除法) |
| | `HasCExtension` | `true` | C 扩展 (压缩指令) |
| | `HasHExtension` | `true` | H 扩展 (虚拟化) |
| | `HasDiv` | `true` | 硬件除法 |
| | `HasFPU` | `true` | 浮点单元 |
| | `HasVPU` | `true` | 向量单元 |
| | `EnableSv48` | `true` | Sv48 分页 |
| 流水线宽度 | `FetchWidth` | `8` | 每周期取指 |
| | `DecodeWidth` | `6` | 每周期译码 |
| | `RenameWidth` | `6` | 每周期重命名 |
| | `CommitWidth` | `8` | 每周期提交 |
| | `RobCommitWidth` | `8` | ROB 提交宽度 |
| 乱序窗口 | `RobSize` | `160` | 重排序缓冲 |
| | `RabSize` | `256` | 重命名备份缓冲 |
| | `FtqSize` | `64` | 取指传输队列 |
| | `IBufSize` | `48` | 指令缓冲 |
| 物理寄存器 | `intPreg.numEntries` | `224` | 整数物理寄存器 |
| | `fpPreg.numEntries` | `192` | 浮点物理寄存器 |
| | `vfPreg.numEntries` | `128` | 向量物理寄存器 |
| | `v0Preg.numEntries` | `22` | V0 寄存器 |
| | `vlPreg.numEntries` | `32` | VL 寄存器 |
| 分支预测 | `FtbSize` | `2048` | FTB 条目 |
| | `FtbWays` | `4` | FTB 相联度 |
| | `TageTableInfos` | 4 tables | TAGE 表配置 |
| | `ITTageTableInfos` | 5 tables | ITTAGE 表配置 |
| | `RasSize` | `16` | 返回地址栈 |
| | `RasSpecSize` | `32` | RAS 投机栈 |
| 缓存 | `icacheParameters` | 64KB, 4-way | L1 I-Cache |
| | `dcacheParametersOpt` | 64KB, 4-way | L1 D-Cache |
| | `L2CacheParamsOpt` | 1MB, 8-way | L2 缓存 |
| | `L2NBanks` | `1` | L2 Bank 数 |
| 内存系统 | `VirtualLoadQueueSize` | `72` | Load 队列 |
| | `LoadQueueRARSize` | `72` | RAR 队列 |
| | `LoadQueueRAWSize` | `32` | RAW 队列 |
| | `StoreQueueSize` | `56` | Store 队列 |
| | `StoreBufferSize` | `16` | Store Buffer |
| 向量 | `VLEN` | `128` | 向量长度 (bits) |
| | `ELEN` | `64` | 元素长度 (bits) |

#### 3.2.2 调度器配置 (BackendParams)

| 调度器 | 发射块 | 每块条目 | 总条目 | 执行单元 |
|--------|--------|---------|--------|---------|
| IntScheduler | 4 块 | 24 | 96 | ALU0-3, BJU0-3 |
| FpScheduler | 3 块 | 18 | 54 | FEX0-4 |
| VfScheduler | 3 块 | 16 | 48 | VFEX0-4 |
| MemScheduler | 6 块 | 16 | 96 | LDU0-2, STA0-1, STD0-1, VLSU0-1 |

### 3.3 配置组合方式

```scala
// KunminghuV2Config 组合示例 (源自 Configs.scala)
class KunminghuV2Config(n: Int = 1) extends Config(
  L2CacheConfig("1MB", inclusive = true, banks = 4, tp = false)  // L2 配置
    ++ new DefaultConfig(n)                                       // 默认配置
    ++ new WithCHI                                                // 启用 CHI 一致性
)

// 编程式配置修改 (alter)
class XSNoCTopConfig(n: Int = 1) extends Config(
  (new KunminghuV2Config(n)).alter((site, here, up) => {
    case SoCParamsKey => up(SoCParamsKey).copy(UseXSNoCTop = true)
  })
)
```

---

## 4. 模块划分与接口定义

### 4.1 前端模块 (Frontend)

| 模块 | 类名 | 功能 | 关键参数 |
|------|------|------|---------|
| Frontend 顶层 | `Frontend` | 前端 LazyModule 包装 | - |
| Frontend 内联 | `FrontendInlined` | 内联实现，减少模块层级 | - |
| 分支预测单元 | `BPU` | 多预测器协同 (FTB/TAGE/ITTAGE/RAS/UBTB) | 3 级流水线 S1/S2/S3 |
| 取指目标缓冲 | `FTB` | 分支目标预测 | 2048 entries, 4-way |
| TAGE 预测器 | `Tage_SC` | 标签几何历史长度预测 | 4 tables, 4096 entries |
| ITTAGE 预测器 | `ITTage` | 间接跳转预测 | 5 tables |
| 返回地址栈 | `RAS` | 函数返回地址预测 | 16 entries |
| 快速 FTB | `FauFTB` | 快速分支预测 | 256 entries |
| 取指传输队列 | `NewFtq` | 取指到后端的队列 | 64 entries |
| 指令缓存 | `ICache` | L1 I-Cache | 64KB, 4-way, SECDED |
| 指令缓冲 | `IBuffer` | 指令缓冲 | 48 entries, 6 banks |
| 预译码 | `PreDecode` | 指令预解析 | 识别分支/跳转 |

**前端接口 Bundle (源自 Frontend.scala)：**

```scala
val io = IO(new Bundle() {
  val hartId       = Input(UInt(hartIdLen.W))           // 核心 ID
  val reset_vector = Input(UInt(PAddrBits.W))           // 复位向量
  val fencei       = Input(Bool())                      // 指令缓存刷新
  val ptw          = new TlbPtwIO()                     // 页表遍历接口
  val backend      = new FrontendToCtrlIO               // 到后端接口
  val softPrefetch = Vec(backendParams.LduCnt, Flipped(Valid(new SoftIfetchPrefetchBundle)))
  val sfence       = Input(new SfenceBundle)            // 地址空间刷新
  val tlbCsr       = Input(new TlbCsrBundle)            // TLB CSR
  val csrCtrl      = Input(new CustomCSRCtrlIO)         // 自定义 CSR 控制
  val error        = Output(new L1BusErrorUnitInfo)     // 错误信息
  val frontendInfo = new Bundle {                       // 前端信息
    val ibufFull = Output(Bool())
    val bpuInfo  = new Bundle { val bpRight, bpWrong = Output(UInt(XLEN.W)) }
  }
  val dft          = Option.when(hasDFT)(Input(new SramBroadcastBundle))
  val dft_reset    = Option.when(hasMbist)(Input(new DFTResetSignals()))
})
```

### 4.2 后端模块 (Backend)

| 模块 | 类名 | 功能 | 关键参数 |
|------|------|------|---------|
| Backend 顶层 | `Backend` | 后端 LazyModule 包装 | `BackendParams` |
| Backend 内联 | `BackendInlined` | 内联实现 | `BackendParams` |
| 译码单元 | `DecodeUnit` | 指令译码 | 6-wide |
| 重命名表 | `RenameTable` | 逻辑→物理寄存器重命名 | 32 逻辑寄存器 |
| 空闲列表 | `FreeList` | 物理寄存器分配 | 224 (Int), 192 (FP) |
| 忙表 | `BusyTable` | 寄存器状态跟踪 | - |
| 发射队列 | `IssueQueue` | 指令发射 | Int/Fp/Vf/Mem 调度器 |
| 重排序缓冲 | `Rob` | 乱序提交 | 160 entries |
| 重命名备份 | `Rab` | 异常恢复备份 | 256 entries |
| 寄存器堆 | `RegFile` | 物理寄存器堆 | 多端口读写 |

**后端调度器配置 (源自 Parameters.scala intSchdParams)：**

```scala
val intSchdParams = {
  implicit val schdType: SchedulerType = IntScheduler()
  SchdBlockParams(Seq(
    IssueBlockParams(Seq(
      ExeUnitParams("ALU0", Seq(AluCfg, MulCfg, BkuCfg), ...),
      ExeUnitParams("BJU0", Seq(BrhCfg, JmpCfg), ...),
    ), numEntries = IssueQueueSize, numEnq = 2, numComp = IssueQueueCompEntrySize),
    // ... ALU1/BJU1, ALU2/BJU2, ALU3/BJU3
  ), numPregs = intPreg.numEntries, ...)
}
```

### 4.3 执行单元 (Execution Units)

| 执行单元 | 功能 | 流水线级数 | WB 端口 |
|---------|------|-----------|---------|
| ALU0-3 | 整数运算 (加/减/移位/位操作) | 1-2 级 | IntWB |
| BJU0-3 | 分支/跳转 (条件/无条件/返回) | 1-3 级 | IntWB |
| FEX0-4 | 浮点运算 (加/减/乘/转换) | 3-8 级 | FpWB |
| VFEX0-4 | 向量运算 (VADD/VMUL/VFMA 等) | 3-8 级 | VfWB/V0WB/VlWB |
| LDU0-2 | Load 地址生成 | 3 级 | IntWB/FpWB |
| STA0-1 | Store 地址 | 2 级 | FakeIntWB |
| STD0-1 | Store 数据 | 2 级 | - |
| VLSU0-1 | 向量 Load/Store | 可变 | VfWB/V0WB/VlWB |

### 4.4 存储系统 (Memory System)

| 模块 | 功能 | 关键参数 |
|------|------|---------|
| DCache | L1 数据缓存 | 64KB, 4-way, SECDED, 128B 行 |
| LoadQueue | Load 队列 | 72 entries, RAR/RAW/Replay 子队列 |
| StoreQueue | Store 队列 | 56 entries |
| SBuffer | Store Buffer | 16 entries |
| TLB | 一级 TLB | 48 entries (I/D/S/H/PF) |
| L2TLB | 二级 TLB | 48 entries |
| PTW | 页表遍历 | 硬件实现，Sv39/Sv48 |
| Prefetcher | 预取器 | SMS + BOP + TP |

---

## 5. 接口标准与 Bundle 定义

### 5.1 指令微操作 (MicroOp)

```scala
class MicroOp extends Bundle {
  val pc          = UInt(VAddrBits.W)          // 程序计数器
  val instr       = UInt(32.W)                 // 指令编码
  val exceptionVec = UInt(13.W)                // 异常向量
  val isVec       = Bool()                     // 是否向量指令
  val is128Bit    = Bool()                     // 是否 128 位访存
  val fuOpType    = UInt()                     // 功能单元操作类型
  val fuType      = UInt()                     // 功能单元类型
  val ctrl        = new CtrlSignals            // 控制信号
  val src         = Vec(3, new SrcInfo)        // 源操作数
  val dst         = new DstInfo                // 目标操作数
  val mem         = new MemInfo                // 访存信息
  val debug       = new DebugInfo              // 调试信息
}
```

### 5.2 重命名接口

```scala
class RenamePair extends Bundle {
  val old_pdest = UInt(PhyRegIdxWidth.W)       // 旧物理寄存器
  val new_pdest = UInt(PhyRegIdxWidth.W)       // 新物理寄存器
}

class RenameInterface(implicit p: Parameters) extends Bundle {
  val ren_src = Vec(RenameWidth, Vec(2, Valid(UInt(PhyRegIdxWidth.W))))
  val ren_dst = Vec(RenameWidth, Valid(UInt(PhyRegIdxWidth.W)))
  val ren_src_snapshot = Vec(RenameWidth, Valid(UInt(PhyRegIdxWidth.W)))
}
```

### 5.3 发射队列接口

```scala
class IssueQueueEntry extends Bundle {
  val uop     = new MicroOp                    // 微操作
  val bankMask = UInt(4.W)                     // Bank 掩码
  val selectSig = UInt(4.W)                    // 选择信号
}
```

### 5.4 写回接口

```scala
class WbBundle extends Bundle {
  val uop     = new MicroOp
  val data    = UInt(DataBits.W)               // 写回数据
  val valid   = Bool()                         // 写回有效
}
```

---

## 6. 项目计划与资源分配

### 6.1 开发阶段划分

| 阶段 | 时间 | 主要任务 | 交付物 |
|------|------|---------|--------|
| P1: 基础设施 | T+1 月 | 搭建 Mill 环境、CI/CD、代码规范 | 开发指南、CI 配置 |
| P2: 前端开发 | T+2~4 月 | 取指、分支预测、ICache | 前端 RTL + 单元测试 |
| P3: 后端开发 | T+3~6 月 | 译码、重命名、发射、执行 | 后端 RTL + 单元测试 |
| P4: 存储系统 | T+5~7 月 | DCache、LSQ、TLB、预取 | 存储 RTL + 单元测试 |
| P5: 系统集成 | T+7~9 月 | 模块集成、接口对齐、NoC | 完整 RTL |
| P6: 验证调试 | T+8~12 月 | difftest、性能调优、回归 | 验证报告 |
| P7: 物理设计 | T+11~15 月 | 综合、P&R、时序收敛 | GDSII |
| P8: 流片 | T+15~18 月 | 流片、封装、测试 | 样片 |

### 6.2 资源分配

| 角色 | 人数 | 主要职责 | 负责模块 |
|------|------|---------|---------|
| 项目总监 | 1 | 整体规划、决策评审、风险管理 | 全局 |
| 架构师 | 2 | 微架构定义、性能建模、参数配置 | `Configs.scala`, `Parameters.scala` |
| 前端工程师 | 3 | 取指、分支预测、ICache、ITLB | `xiangshan/frontend/` |
| 后端工程师 | 4 | 译码、重命名、发射、执行、ROB | `xiangshan/backend/` |
| 存储工程师 | 3 | DCache、LSQ、TLB、PTW、预取 | `xiangshan/cache/`, `xiangshan/mem/` |
| 验证工程师 | 4 | Difftest、单元测试、系统测试 | `difftest/`, `src/test/scala/` |
| 物理设计工程师 | 2 | 综合、P&R、时序、DFT | `Makefile`, `scripts/` |
| 工具工程师 | 1 | CI/CD、构建系统、Docker | `build.sc`, `Makefile`, `Dockerfile` |

### 6.3 依赖管理

```
XiangShan 主仓库 (kunminghu-v2 分支)
├── rocket-chip          # RISC-V 基础库 (Diplomacy, TileLink, AXI4)
├── difftest             # 仿真验证框架 (DPI-C 接口)
├── huancun              # L2/L3 缓存框架
├── fudian               # 浮点运算单元
├── yunsuan              # 向量/ALU 运算库
├── coupledL2            # Coupled L2 缓存 (CHI)
├── openLLC              # OpenLLC (CHI LLC, OpenNCB)
├── ChiselAIA            # AIA/IMSIC/APLIC 中断控制器
├── ChiselIOPMP          # I/O PMP
├── utility              # 工具库 (MBIST, SRAM, Arbiter)
└── ready-to-run         # 预编译仿真二进制

初始化命令: make init / make init-force
```

---

## 7. 验证策略

### 7.1 验证层次模型

```
┌─────────────────────────────────────────────────────────────────────────┐
│                     系统级验证 (System Level)                             │
│  Linux Boot / Android Boot / RTOS Boot                                   │
│  SPEC CPU 2006 / SPEC CPU 2017                                           │
│  Application Workloads                                                   │
├─────────────────────────────────────────────────────────────────────────┤
│                   集成级验证 (Integration Level)                           │
│  Full-Chip Simulation / Difftest (DPI-C + NEMU)                          │
│  SMP Linux / Multicore Workloads                                         │
│  I/O Devices / Peripherals (UART, Timer, PLIC, ...)                      │
├─────────────────────────────────────────────────────────────────────────┤
│                     模块级验证 (Module Level)                              │
│  Unit Tests (ChiselTest 6.0.0)                                           │
│  Module-Level Difftest                                                   │
│  Protocol Checks / Assertions                                            │
├─────────────────────────────────────────────────────────────────────────┤
│                     单元级验证 (Unit Level)                                │
│  Formal Verification (SymbiYosys)                                        │
│  Code Coverage (Verilator --coverage)                                    │
│  Functional Coverage                                                     │
│  Lint Checks (scalafmt, scalastyle)                                      │
└─────────────────────────────────────────────────────────────────────────┘
```

### 7.2 验证方法矩阵

| 验证方法 | 适用层次 | 工具 | 覆盖目标 |
|----------|---------|------|---------|
| 单元测试 | 单元级 | ChiselTest 6.0.0 | 模块功能正确性 |
| 差分测试 | 集成级 | Difftest + NEMU (DPI-C) | 指令级正确性 |
| 形式验证 | 单元级 | SymbiYosys | 关键属性 (死锁、数据一致性) |
| 仿真测试 | 集成级 | Verilator / VCS / GalaxSim | 系统功能验证 |
| 硬件仿真 | 系统级 | Palladium / FPGA | 系统级验证 |
| 代码覆盖 | 单元级 | Verilator --coverage | 行/分支/表达式/翻转覆盖率 |
| 功能覆盖 | 模块级 | 自定义 coverpoint | 功能点覆盖 |
| 性能验证 | 系统级 | Verilator + VCS | 性能回归 |

### 7.3 Difftest 接口事件

```scala
// Difftest DPI-C 接口事件类型 (源自 difftest 框架)
difftest_event_t {
  DifftestArchIntRegState,     // 整数寄存器状态对比
  DifftestArchFpRegState,      // 浮点寄存器状态对比
  DifftestArchVecRegState,     // 向量寄存器状态对比
  DifftestInstrCommit,         // 指令提交对比
  DifftestLoadEvent,           // Load 事件对比
  DifftestStoreEvent,          // Store 事件对比
  DifftestAtomicEvent,         // 原子操作事件对比
  DifftestPtwEvent,            // 页表遍历事件对比
  DifftestRefillEvent,         // 缓存重填事件对比
  DifftestFlushEvent,          // 流水线刷新事件对比
}
```

### 7.4 验证计划

| 阶段 | 验证内容 | 通过标准 | 时间 |
|------|---------|---------|------|
| V1: 单元测试 | 各模块功能 | 代码覆盖率 > 90% | T+1 月 |
| V2: 集成测试 | 模块间接口 | Difftest 全部通过 | T+2 月 |
| V3: 系统测试 | 全芯片功能 | Linux 启动成功 | T+3 月 |
| V4: 性能测试 | 性能回归 | 不低于基线 | T+4 月 |
| V5: 压力测试 | 极端场景 | 无死锁/挂死 | T+5 月 |
| V6: 回归测试 | 全量回归 | 100% 通过 | T+6 月 |

---

## 8. 风险管理计划

| 风险 ID | 风险描述 | 概率 | 影响 | 风险等级 | 应对策略 | 负责人 |
|---------|---------|------|------|---------|---------|--------|
| R01 | Chisel 6.7.0 版本升级不兼容 | 中 | 高 | 高 | 锁定版本，升级前回归测试 | 工具组 |
| R02 | 乱序执行验证覆盖率不足 | 高 | 高 | 高 | 加强 Difftest，引入形式验证 | 验证组 |
| R03 | CHI 一致性协议死锁 | 中 | 高 | 高 | 协议仿真，死锁检测，CHILogger | 存储组 |
| R04 | 物理设计时序不收敛 | 中 | 中 | 中 | 早期时序预算，多轮迭代 | 物理设计组 |
| R05 | 关键人员流失 | 低 | 高 | 中 | 知识文档化，交叉培训 | 项目总监 |
| R06 | 子模块版本冲突 | 中 | 中 | 中 | 严格版本管理，CI 集成测试 | 工具组 |
| R07 | 多核扩展性问题 | 中 | 中 | 中 | SMP 测试，NoC 压力测试 | 架构组 |

---

## 9. 质量保证与 CI/CD

### 9.1 代码规范

| 规范 | 工具 | 检查内容 | 配置文件 |
|------|------|---------|---------|
| 代码格式 | scalafmt | 缩进、空格、换行 | `.scalafmt.conf` |
| 代码风格 | scalastyle | 命名、复杂度、注释 | `scalastyle-config.xml` |
| 测试风格 | scalastyle-test | 测试代码规范 | `scalastyle-test-config.xml` |

### 9.2 CI/CD 流程 (基于 .github/workflows/emu.yml)

```yaml
# CI 主要步骤 (kunminghu-v2 分支)
1. 代码格式检查 (scalafmt checkFormat)
2. 编译检查 (mill compile)
3. Verilog 生成检查 (make verilog)
4. EMU 基础测试 (make emu)
5. EMU CHI 测试 (CHI 一致性)
6. EMU 性能测试 (SPEC06)
7. EMU 多核测试 (SMP)
8. SIMV 基础测试 (VCS 商业仿真)
9. Docker 镜像构建 (xsdev 镜像)
10. 子模块版本检查
11. 代码覆盖率上传 (actions/upload-artifact)
```

### 9.3 提交规范

```
feat:     新功能
fix:      修复 Bug
docs:     文档更新
style:    代码格式调整
refactor: 重构
test:     测试相关
chore:    构建/工具相关
```

### 9.4 分支策略

```
main/master          # 主分支，稳定版本
├── develop          # 开发分支
├── feature/*        # 功能分支
├── bugfix/*         # Bug 修复分支
├── release/*        # 发布分支
├── hotfix/*         # 热修复分支
└── kunminghu-v2     # 昆明湖 V2 分支 (当前)
```

---

## 10. 计划阶段决策评审 (PDCP)

### 10.1 评审检查清单

- [x] 系统架构设计完成 (Top.scala + BaseXSSoc + Diplomacy)
- [x] 模块划分与接口定义完成 (Frontend/Backend/Mem)
- [x] 配置系统设计完成 (CDE + Chisel 参数化)
- [x] 项目计划制定完成 (8 个阶段，18 个月)
- [x] 资源分配完成 (7 个职能组，24 人)
- [x] 验证策略制定完成 (6 层验证模型)
- [x] 风险管理计划完成 (7 项主要风险)
- [x] 质量保证措施就绪 (CI/CD，代码规范)

### 10.2 评审结论

**通过** — 项目进入开发阶段。

---

## 11. 附录

### 11.1 术语表

| 术语 | 全称 | 说明 |
|------|------|------|
| CDE | Configurable Design Environment | rocket-chip 参数化配置系统 |
| LazyModule | 延迟模块 | Diplomacy 框架中的模块模式 |
| Diplomacy | 总线抽象框架 | 自动处理总线节点连接 |
| TileLink | 片上总线协议 | rocket-chip 标准总线 |
| CHI | Coherent Hub Interface | ARM 一致性接口协议 |
| Difftest | 差分测试 | DUT 与参考模型 (NEMU) 对比验证 |
| DPI-C | Direct Process Interface C | Verilator/VCS 与 C/C++ 交互接口 |
| FIRRTL | Flexible Intermediate Representation for RTL | Chisel 编译器中间表示 |
| MBIST | Memory Built-In Self-Test | SRAM 自测试 |
| PPA | Performance, Power, Area | 性能、功耗、面积 |

### 11.2 参考文档

| 文档 | 位置 | 说明 |
|------|------|------|
| 香山官方设计文档 | https://docs.xiangshan.cc | 三代架构演进、敏捷开发理念 |
| Chisel 文档 | https://www.chisel-lang.org | Chisel 6.7.0 语言参考 |
| RISC-V 规范 | https://riscv.org/technical/specifications/ | RV64GCBVMV, H-extension, Sv39/Sv48 |
| rocket-chip 文档 | https://github.com/chipsalliance/rocket-chip | Diplomacy, TileLink, AXI4, LazyModule |
| 香山 GitHub | https://github.com/OpenXiangShan/XiangShan | 源码仓库，Release v1.0/v2.0/v2.1 |

---

*文档版本：v1.1（深度优化版）*
*最后更新：2026-10-20*
*基于：docs.xiangshan.cc、Chisel 6.7.0、RISC-V 规范、rocket-chip Diplomacy、本地代码分析*
