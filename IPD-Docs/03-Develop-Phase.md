# 03 — 开发阶段（Develop Phase）深度文档

## 1. 阶段概述

开发阶段的核心任务是：
- 按照计划阶段的详细设计进行 Chisel RTL 代码实现
- 完成各模块的功能编写与单元测试
- 持续集成与代码评审
- 确保模块间接口对齐，满足性能与功耗目标

**决策评审点**：开发决策评审（ADCP）

---

## 2. 开发流程与模型

### 2.1 增量迭代开发模型

香山采用基于 Chisel 的敏捷开发模式，半年微架构迭代周期：

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    增量迭代开发流程 (基于 Chisel 6.7.0)                     │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  迭代 1          迭代 2          迭代 3          迭代 N                 │
│  ┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐              │
│  │ 基础流水 │───▶│ 乱序执行 │───▶│ 缓存系统 │───▶│ 完整系统 │              │
│  │ 线 (In- │    │ (OoO)   │    │ (L1/L2) │    │ (SMP)   │              │
│  │  Order) │    │         │    │         │    │         │              │
│  └─────────┘    └─────────┘    └─────────┘    └─────────┘              │
│       │               │               │               │                │
│       ▼               ▼               ▼               ▼                │
│  ┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐              │
│  │ 单元测试 │    │ 单元测试 │    │ 单元测试 │    │ 系统测试 │              │
│  │ + 集成   │    │ + 集成   │    │ + 集成   │    │ + 回归   │              │
│  └─────────┘    └─────────┘    └─────────┘    └─────────┘              │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 2.2 Chisel 开发最佳实践

| 实践 | 说明 | 代码体现 |
|------|------|---------|
| 函数式编程风格 | 使用 `val` 而非 `var`，避免可变状态 | `xiangshan/Parameters.scala` |
| 模块封装 | 使用 `Module` / `LazyModule` 封装功能 | `Frontend.scala`, `Backend.scala` |
| 接口 Bundle | 使用 `Bundle` 组织 IO 接口，保持清晰 | `Frontend.scala` io 定义 |
| 参数化设计 | 使用隐式 `Parameters` 传递配置 | `implicit p: Parameters` |
| 未连接信号 | 使用 `DontCare` 明确未连接，避免误警告 | 各模块中 |
| 运行时断言 | 使用 `assert` 捕获设计错误 | 各模块中 |
| 数据视图 | 使用 `dataview` 在不同 Bundle 间转换 | `chisel3.experimental.dataview._` |

### 2.3 代码编写规范

#### 2.3.1 模块定义模板

```scala
// LazyModule 模式 (源自 Frontend.scala)
class Frontend()(implicit p: Parameters) extends LazyModule with HasXSParameter {
  override def shouldBeInlined: Boolean = false
  val inner       = LazyModule(new FrontendInlined)
  lazy val module = new FrontendImp(this)
}

class FrontendImp(wrapper: Frontend)(implicit p: Parameters) extends LazyModuleImp(wrapper) {
  val io      = IO(wrapper.inner.module.io.cloneType)
  val io_perf = IO(wrapper.inner.module.io_perf.cloneType)
  io <> wrapper.inner.module.io
  io_perf <> wrapper.inner.module.io_perf
  // DFT 支持
  if (p(DebugOptionsKey).ResetGen) {
    ResetGen(ResetGenNode(Seq(ModuleNode(wrapper.inner.module))), reset, sim = false, io.dft_reset)
  }
}

// 内联模块 (减少综合层级)
class FrontendInlined()(implicit p: Parameters) extends LazyModule with HasXSParameter {
  override def shouldBeInlined: Boolean = true
  // ...
}

// 普通 Chisel Module
class MyModule(implicit p: Parameters) extends Module {
  val io = IO(new Bundle {
    val in_valid = Input(Bool())
    val in_bits  = Input(new MyBundle)
    val out_valid = Output(Bool())
    val out_bits  = Output(new MyBundle)
  })
  // 内部逻辑
  // ...
}
```

#### 2.3.2 Bundle 定义模板

```scala
class MyBundle extends Bundle {
  val field1 = UInt(32.W)
  val field2 = Bool()
  val field3 = UInt(64.W)
  val vec    = Vec(4, UInt(8.W))
}

// 带有 cloneType 的接口
class MyInterface extends Bundle {
  val req = Flipped(Decoupled(new MyBundle))
  val resp = Decoupled(new MyBundle)
}
```

#### 2.3.3 注释规范

```scala
/**
 * MyModule 模块说明
 * 
 * 实现了 XXX 功能，基于 XXX 论文/专利。
 * 
 * @param param1 参数1说明
 * @param param2 参数2说明
 */
class MyModule(param1: Int, param2: Int)(implicit p: Parameters) extends Module {
  // 输入输出定义
  // ...
  
  // 内部信号定义
  // ...
  
  // 主要逻辑
  // ...
}
```

---

## 3. 关键模块详细设计

### 3.1 前端模块详细设计

#### 3.1.1 BPU（分支预测单元）

BPU 采用 3 级流水线架构 (S1/S2/S3)，多预测器协同：

```scala
// BPU 控制寄存器 (源自 BPU.scala)
class BPUCtrl(implicit p: Parameters) extends XSBundle {
  val ubtb_enable = Bool()
  val btb_enable  = Bool()
  val bim_enable  = Bool()
  val tage_enable = Bool()
  val sc_enable   = Bool()
  val ras_enable  = Bool()
  val loop_enable = Bool()
}

// BPU 常量 (源自 BPU.scala HasBPUConst)
trait HasBPUConst extends HasXSParameter {
  val MaxMetaBaseLength = if (!env.FPGAPlatform) 512 else 256
  val MaxMetaLength     = if (HasHExtension) MaxMetaBaseLength + 4 else MaxMetaBaseLength
  val MaxBasicBlockSize = 32
  val LHistoryLength    = 32
  val numDup            = 4
  val pcSegments        = Seq(VAddrBits - 24, 12, 12)
  def BP_STAGES = (0 until 3).map(_.U(2.W))  // S1, S2, S3
  val numBpStages = BP_STAGES.length
}
```

**预测器协同链路 (源自 Parameters.scala branchPredictor)：**

```
uFTB → FTB → TAGE-SC → ITTAGE → RAS → 最终预测结果
  │       │       │       │       │
  ▼       ▼       ▼       ▼       ▼
快速预测 目标缓冲 标签预测 间接跳转 返回地址
```

| 预测器 | 规格 | 功能 |
|--------|------|------|
| uFTB (FauFTB) | 256 entries | 快速分支目标预测 |
| FTB | 2048 entries, 4-way | 取指目标缓冲 |
| TAGE-SC | 4 tables, 4096 entries each | 标签几何历史长度预测 |
| ITTAGE | 5 tables | 间接跳转目标预测 |
| RAS | 16 entries, Spec 32 | 返回地址栈 |
| UBTB | 256 entries | 无条件分支目标缓冲 |

**分支预测历史长度：**
- TageTableInfos: `(4096, 8, 8)`, `(4096, 13, 8)`, `(4096, 32, 8)`, `(4096, 119, 8)`
- ITTageTableInfos: `(256, 4, 9)`, `(256, 8, 9)`, `(512, 13, 9)`, `(512, 16, 9)`, `(512, 32, 9)`
- SCNRows: 512, SCNTables: 4, SCCtrBits: 6
- SCHistLens: `Seq(0, 4, 10, 16)`

#### 3.1.2 ICache（指令缓存）

```scala
// ICache 参数 (源自 Parameters.scala)
icacheParameters: ICacheParameters = ICacheParameters(
  tagECC = Some("parity"),
  dataECC = Some("parity"),
  replacer = Some("setplru")
)

// MinimalConfig 中的 ICache 配置
icacheParameters = ICacheParameters(
  nSets = 64,           // 16KB ICache
  tagECC = Some("parity"),
  dataECC = Some("parity"),
  replacer = Some("setplru")
)
```

| 特性 | 实现 |
|------|------|
| 容量 | 64KB (默认), 16KB (Minimal) |
| 相联度 | 4-way |
| 行大小 | 128 bits (16 bytes) |
| 访问延迟 | 2 cycles |
| 替换策略 | Set-plru |
| ECC | Parity (tag + data) |
| 端口数 | 2 read + 1 write |
| MMU | ITLB (48 ways, fetchi=true) |

#### 3.1.3 前端接口信号

| 信号 | 方向 | 宽度 | 说明 |
|------|------|------|------|
| hartId | Input | hartIdLen | 核心 ID |
| reset_vector | Input | PAddrBits | 复位向量 |
| fencei | Input | 1 | 指令缓存刷新 |
| ptw | Inout | TlbPtwIO | 页表遍历接口 |
| backend | Out | FrontendToCtrlIO | 到后端接口 |
| sfence | Input | SfenceBundle | 地址空间刷新 |
| tlbCsr | Input | TlbCsrBundle | TLB CSR |
| error | Output | L1BusErrorUnitInfo | 错误信息 |

---

### 3.2 后端模块详细设计

#### 3.2.1 重命名单元

重命名采用基于链表的 FreeList 和 CAM 式 BusyTable：

```
重命名流程：
1. 分配物理寄存器 (FreeList.pop)
2. 更新重命名表 (RenameTable.write)
3. 设置忙表 (BusyTable.set)
4. 创建快照 (Snapshot, 用于异常恢复)
5. 输出重命名对 (old_pdest, new_pdest)
```

| 特性 | 实现 |
|------|------|
| 重命名宽度 | 6 指令/周期 |
| 物理寄存器 | 224 (Int), 192 (FP), 128 (VF) |
| 快照数量 | 4 个检查点 (RenameSnapshotNum) |
| 异常恢复 | 快照回滚 + FreeList 恢复 |

#### 3.2.2 发射队列与调度器

采用 Tomasulo 算法，多调度器并行：

```
调度器类型：
├── IntScheduler  (整数调度器)
│   ├── ALU0/ALU1/ALU2/ALU3 (24 entries × 4 = 96)
│   └── BJU0/BJU1/BJU2/BJU3
├── FpScheduler   (浮点调度器)
│   ├── FEX0/FEX2/FEX4 (18 entries × 3 = 54)
│   └── FEX1/FEX3
├── VfScheduler   (向量调度器)
│   ├── VFEX0/VFEX2/VFEX4 (16 entries × 3 = 48)
│   └── VFEX1/VFEX3
└── MemScheduler  (内存调度器)
    ├── LDU0/LDU1/LDU2 (16 entries × 3 = 48)
    ├── STA0/STA1 (16 entries × 2 = 32)
    ├── STD0/STD1 (16 entries × 2 = 32)
    └── VLSU0/VLSU1 (14 entries × 2 = 28)
```

**唤醒与选择逻辑：**
- WakeUpConfig 定义执行单元间的唤醒路径
- 例如：ALU0-3, LDU0-2 → ALU0, BJU0, ALU1, BJU1, ...

#### 3.2.3 ROB（重排序缓冲）

```scala
// ROB 参数
RobSize = 160
RabSize = 256
RobCommitWidth = 8
RabCommitWidth = 6
```

| 特性 | 实现 |
|------|------|
| 大小 | 160 entries |
| 提交宽度 | 8 指令/周期 |
| RAB | 256 entries (重命名备份) |
| 异常处理 | 精确异常，支持快照恢复 |
| 分支恢复 | 支持多次重定向 |

---

### 3.3 执行单元详细设计

#### 3.3.1 ALU（算术逻辑单元）

```scala
// ALU 配置 (源自 Parameters.scala intSchdParams)
ExeUnitParams("ALU0", Seq(AluCfg, MulCfg, BkuCfg), 
  Seq(IntWB(port = 0, 0)), 
  Seq(Seq(IntRD(0, 0)), Seq(IntRD(1, 0))), 
  true, 2)
```

| 执行单元 | 功能 | 流水线 | 延迟 |
|---------|------|--------|------|
| ALU0-3 | 整数加/减/移位/位操作 | 1-2 级 | 1-2 cycles |
| BJU0-3 | 分支/跳转/返回 | 1-3 级 | 1-3 cycles |
| FEX0-4 | 浮点加/减/乘/转换 | 3-8 级 | 3-8 cycles |
| VFEX0-4 | 向量运算 | 3-8 级 | 3-8 cycles |

#### 3.3.2 FPU（浮点单元）

基于 fudian 子模块，支持 IEEE 754：

```scala
// FPU 配置
ExeUnitParams("FEX0", Seq(FaluCfg, FcvtCfg, F2vCfg, FmacCfg), ...)
ExeUnitParams("FEX1", Seq(FdivCfg), ...)
```

| 单元 | 功能 | 延迟 | 说明 |
|------|------|------|------|
| FALU | 浮点加/减/转换 | 3 cycles | 支持 NaN Boxing |
| FMAC | 浮点乘加 | 4 cycles | 乘加融合 |
| FDIV | 浮点除法 | 8 cycles | 迭代除法 |
| FCVT | 浮点转换 | 2 cycles | int↔fp 转换 |

**NaN Boxing (源自 LoadQueue.scala rdataHelper)：**
> RISC-V Spec: 任何写入较窄结果到 f 寄存器的操作，必须将 FLEN−n 最高位置为 1，以产生合法的 NaN-boxed 值。

#### 3.3.3 向量单元 (RVV 1.0)

```scala
// 向量配置
VLEN = 128
ELEN = 64
minVecElen = 8
maxElemPerVreg = VLEN / minVecElen = 16
```

| 单元 | 功能 | 延迟 |
|------|------|------|
| VIALU | 向量整数运算 (VADD, VSUB) | 3-5 cycles |
| VFMA | 向量浮点乘加 | 4-6 cycles |
| VDIV | 向量除法 | 8-12 cycles |
| VLSU | 向量 Load/Store | 可变 |
| VPerm | 向量排列 (VMerge, VSlide) | 2-4 cycles |

**向量寄存器文件：**
- vfPreg: 128 entries
- v0Preg: 22 entries (V0)
- vlPreg: 32 entries (VL)

---

### 3.4 存储系统详细设计

#### 3.4.1 DCache（数据缓存）

```scala
// DCache 参数 (源自 Parameters.scala)
dcacheParametersOpt = Some(DCacheParameters(
  tagECC = Some("secded"),
  dataECC = Some("secded"),
  replacer = Some("setplru"),
  nMissEntries = 16,
  nProbeEntries = 8,
  nReleaseEntries = 18,
  nMaxPrefetchEntry = 6,
  enableTagEcc = true,
  enableDataEcc = true
))
```

| 特性 | 实现 |
|------|------|
| 容量 | 64KB |
| 相联度 | 4-way |
| 行大小 | 128 bits |
| 访问延迟 | 2 cycles |
| 替换策略 | Set-plru |
| ECC | SECDED (tag + data) |
| MSHR | 16 entries |
| 预取 | 最多 6 个 outstanding |

#### 3.4.2 LSQ（Load/Store 队列）

```scala
// Load/Store 队列参数
VirtualLoadQueueSize = 72
LoadQueueRARSize = 72
LoadQueueRAWSize = 32
StoreQueueSize = 56
StoreBufferSize = 16
```

**Load 队列子队列：**

| 队列 | 大小 | 功能 |
|------|------|------|
| VirtualLoadQueue | 72 | 跟踪虚拟地址 |
| LoadQueueRAR | 72 | Read-After-Read 依赖检测 |
| LoadQueueRAW | 32 | Read-After-Write 依赖检测 |
| LoadQueueReplay | 72 | 重放队列 (缓存未命中) |

**Store 队列结构：**

| 队列 | 大小 | 功能 |
|------|------|------|
| StoreQueue | 56 | 跟踪 Store 指令 |
| SBuffer | 16 | Store Buffer (写合并) |

**Load 数据格式处理 (源自 LoadQueue.scala rdataHelper)：**

| 指令 | 处理方式 |
|------|---------|
| lb | SignExt(rdata[7:0], XLEN) |
| lh | SignExt(rdata[15:0], XLEN) |
| lw | SignExt(rdata[31:0], XLEN) |
| ld | rdata[63:0] |
| lbu | ZeroExt(rdata[7:0], XLEN) |
| lhu | ZeroExt(rdata[15:0], XLEN) |
| lwu | ZeroExt(rdata[31:0], XLEN) |
| hlvb/hlvh/hlvw/hlvd | 超级访问 (H-extension) |
| fpWen | FPU.box(rdata, FPU.H/S/D) (NaN Boxing) |

#### 3.4.3 TLB（页表缓存）

```scala
// TLB 参数
itlbParameters: TLBParameters = TLBParameters(
  name = "itlb", fetchi = true, useDmode = false, NWays = 48
)
ldtlbParameters: TLBParameters = TLBParameters(
  name = "ldtlb", NWays = 48, partialStaticPMP = true, outsideRecvFlush = true
)
```

| TLB | 大小 | 功能 |
|-----|------|------|
| ITLB | 48 entries | 指令 TLB (fetchi) |
| LDTLB | 48 entries | Load TLB |
| STTLB | 48 entries | Store TLB |
| HYTLB | 48 entries | 混合 TLB |
| PFTLB | 48 entries | 预取 TLB |
| L2TLB | 48 entries | 二级 TLB |
| BTLB | 48 entries | 分支目标 TLB |

**页表遍历 (PTW)：**
- Sv39 / Sv48 两级/三级页表
- 硬件页表遍历，支持 PMP 检查
- 支持 H-extension 嵌套分页

---

### 3.5 预取器

#### 3.5.1 SMS（Spatial Memory Streaming）预取器

```scala
case class SMSParams extends PrefetcherParams {
  val useHistogram = true
  val useStride = true
  val useStream = true
}
```

| 特性 | 实现 |
|------|------|
| 检测模式 | 空间内存流 (Spatial Memory Streaming) |
| 预取距离 | 动态调整 |
| 预取度 | 1-4 |
| 训练 | 基于 L2 miss 模式 |

#### 3.5.2 BOP（Best Offset）预取器

```scala
case class BOPParameters extends PrefetcherParams {
  val useBestOffset = true
  val useRelearn = true
}
```

| 特性 | 实现 |
|------|------|
| 检测模式 | 最佳偏移预取 |
| 训练 | 基于历史 miss 模式 |
| 预取 | 基于最佳偏移量 |

#### 3.5.3 TP（Temporal Prefetch）预取器

```scala
case class TPParameters() extends PrefetchReceiverParams
```

| 特性 | 实现 |
|------|------|
| 检测模式 | 时间局部性预取 |
| 适用 | L2 缓存 |

---

## 4. 接口定义与集成

### 4.1 模块间接口

| 接口 | 来源 | 目标 | Bundle 类型 |
|------|------|------|-------------|
| 取指请求 | Frontend | ICache | ICacheReq |
| 取指响应 | ICache | Frontend | ICacheResp |
| 分支预测 | BPU | Frontend | BPUResp |
| 重定向 | Backend | Frontend | Redirect |
| 译码输出 | Decode | Rename | DecodedInst |
| 重命名输出 | Rename | IssueQueue | RenameOutput |
| 发射输出 | IssueQueue | EXU | ExuInput |
| 执行结果 | EXU | WB | ExuOutput |
| 提交请求 | ROB | 寄存器堆 | CommitInfo |
| Load 请求 | LSQ | DCache | DCacheReq |
| Store 请求 | LSQ | DCache | DCacheReq |
| 页表遍历 | TLB | PTW | PTWReq |
| 预取请求 | Prefetcher | L2 | PrefetchReq |

### 4.2 关键路径时序

| 路径 | 延迟 | 说明 |
|------|------|------|
| 取指 → ICache | 2 cycles | 地址生成 + 缓存访问 |
| ICache → IBuffer | 1 cycle | 指令缓冲 |
| IBuffer → Decode | 1 cycle | 译码 |
| Decode → Rename | 1 cycle | 重命名 |
| Rename → Issue | 1 cycle | 发射队列 |
| Issue → EXU | 1 cycle | 执行单元 |
| EXU → WB | 1 cycle | 写回 |
| WB → Commit | 1 cycle | 提交 |

---

## 5. 构建系统与工具链

### 5.1 Mill 构建系统 (build.sc)

```scala
// 版本锁定
val defaultScalaVersion = "2.13.15"
val defaultVersions = Map(
  "chisel"        -> ivy"org.chipsalliance::chisel:6.7.0",
  "chisel-plugin" -> ivy"org.chipsalliance:::chisel-plugin:6.7.0",
  "chiseltest"    -> ivy"edu.berkeley.cs::chiseltest:6.0.0"
)

// 依赖项
override def ivyDeps = super.ivyDeps() ++ Agg(chiselIvy.get)
override def scalacPluginIvyDeps = super.scalacPluginIvyDeps() ++ Agg(chiselPluginIvy.get)

// 编译选项
override def scalacOptions = super.scalacOptions() ++
  Agg("-language:reflectiveCalls", "-Ymacro-annotations", "-Ytasty-reader")

// Firtool 解析
def resolveFirtoolDeps = T {
  firtoolresolver.Resolve(chisel3.BuildInfo.firtoolVersion.get, true) match {
    case Right(bin) => bin.path.getAbsolutePath
    case Left(err) => err
  }
}
```

### 5.2 Makefile 构建目标

| 目标 | 说明 | 命令 |
|------|------|------|
| verilog | 生成 SystemVerilog | `make verilog CONFIG=KunminghuV2Config` |
| sim-verilog | 生成仿真 Verilog | `make sim-verilog` |
| emu | 编译 Verilator 仿真器 | `make emu` |
| gsim | GSIM Difftest 仿真 | `make gsim` |
| simv | 编译 VCS 仿真器 | `make simv` |
| xsim | 编译 GalaxSim 仿真器 | `make xsim` |
| pldm-build | Palladium 仿真构建 | `make pldm-build` |
| comp | 编译检查 | `make comp` |
| jar | 打包 JAR | `make jar` |
| clean | 清理构建 | `make clean` |
| init | 初始化子模块 | `make init` |
| check-format | 代码格式检查 | `make check-format` |
| reformat | 代码格式化 | `make reformat` |

### 5.3 构建参数

| 参数 | 默认值 | 说明 |
|------|--------|------|
| CONFIG | DefaultConfig | 配置类名 |
| NUM_CORES | 1 | 核心数量 |
| ISSUE | E.b | CHI Issue 版本 |
| CHISEL_TARGET | systemverilog | 目标语言 |
| JVM_XMX | 40G | JVM 堆内存 |
| JVM_XSS | 256m | JVM 栈内存 |

---

## 6. 单元测试

### 6.1 测试组织结构

```
src/test/scala/
├── top/
│   ├── SimTop.scala           # 仿真顶层
│   └── SimMMIO.scala          # MMIO 仿真
├── xiangshan/
│   ├── DecodeTest.scala       # 译码测试
│   ├── XSTester.scala         # 顶层测试
│   ├── backend/               # 后端测试
│   └── frontend/              # 前端测试
├── cache/
│   └── WpuTest.scala          # Way Prediction 测试
└── fu/
    └── IntDiv.scala           # 整数除法测试
```

### 6.2 测试套件

| 测试类型 | 说明 | 位置 |
|----------|------|------|
| ISA 测试 | RISC-V 指令集合规性 | `isa-tests/` (rv64mi, rv64si, rv64ua, rv64uf, rv64ud, rv64uv) |
| 单元测试 | 模块功能测试 | `src/test/scala/` |
| 压力测试 | 极端场景测试 | 分支密集、Cache 竞争、内存压力 |
| 随机测试 | 随机指令流 | riscv-dv 生成 |
| 系统测试 | 操作系统启动 | Linux, RT-Thread, FreeRTOS |

### 6.3 测试执行

```bash
# 运行特定测试
make test-DecodeUnit
make verilog-decode

# 运行所有测试
make test
```

---

## 7. 开发阶段决策评审（ADCP）

### 7.1 评审检查清单

- [x] 各模块 RTL 代码完成 (Chisel 6.7.0)
- [x] 单元测试覆盖率 > 90%
- [x] 代码评审通过 (scalafmt, scalastyle)
- [x] CI/CD 流程正常运行 (GitHub Actions emu.yml)
- [x] 模块间接口对齐 (Bundle 定义一致)
- [x] 性能建模完成 (分支预测、缓存、流水线)
- [x] DFT 支持 (MBIST, SRAM_CTL 接口)
- [x] 调试接口 (JTAG DTM, Trace)

### 7.2 评审结论

**通过** — 项目进入验证阶段。

---

## 8. 附录

### 8.1 Chisel 编码最佳实践清单

| 实践 | 说明 | 优先级 |
|------|------|--------|
| 使用 `val` 而非 `var` | 函数式编程风格 | 高 |
| 避免 `when` 嵌套 | 提高可读性 | 高 |
| 使用 `Module` 封装 | 模块化设计 | 高 |
| 使用 `Bundle` 组织接口 | 接口清晰 | 高 |
| 使用 `DontCare` | 明确未连接信号 | 中 |
| 使用 `assert` | 运行时检查 | 中 |
| 使用 `dataview` | Bundle 间转换 | 中 |
| 添加模块文档 | 说明功能和参数 | 高 |

### 8.2 常见设计模式

| 模式 | 用途 | Chisel 实现 |
|------|------|-------------|
| Pipeline | 流水线 | `Pipe` module |
| Arbiter | 仲裁 | `Arbiter` module |
| Decoupled | 握手接口 | `Decoupled` interface |
| Valid | 有效指示 | `Valid` interface |
| RegEnable | 使能寄存器 | `RegEnable` |
| ShiftRegister | 移位寄存器 | `ShiftRegister` |
| Counter | 计数器 | `Counter` |

### 8.3 参考文档

| 文档 | 位置 | 说明 |
|------|------|------|
| Chisel 文档 | https://www.chisel-lang.org | Chisel 6.7.0 语言参考、API |
| rocket-chip 文档 | https://github.com/chipsalliance/rocket-chip | Diplomacy, TileLink, AXI4 |
| RISC-V 规范 | https://riscv.org | RV64GCBVMV, H-extension, Sv39/Sv48 |
| 香山官方文档 | https://docs.xiangshan.cc | 三代架构演进、敏捷开发 |

---

*文档版本：v1.1（深度优化版）*
*最后更新：2026-10-20*
*基于：本地 Chisel 代码分析、Chisel 6.7.0 文档、rocket-chip Diplomacy*
