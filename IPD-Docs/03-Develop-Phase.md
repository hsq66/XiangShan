# 03 — 开发阶段（Develop Phase）

## 1. 阶段概述

开发阶段的核心任务是：
- 按照计划阶段的详细设计进行编码实现
- 完成各模块的 RTL 代码编写
- 进行单元测试与模块级验证
- 持续集成与代码评审

**决策评审点**：开发决策评审（ADCP）

---

## 2. 开发流程

### 2.1 开发模型

香山项目采用 **增量迭代开发模型**：

```
┌─────────────────────────────────────────────────────────────┐
│                    增量迭代开发流程                           │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐ │
│  │ 迭代 1   │──▶│ 迭代 2   │──▶│ 迭代 3   │──▶│ 迭代 N   │ │
│  │ 基础流水 │   │ 乱序执行 │   │ 缓存系统 │   │ 完整系统 │ │
│  └──────────┘   └──────────┘   └──────────┘   └──────────┘ │
│       │              │              │              │       │
│       ▼              ▼              ▼              ▼       │
│  ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐ │
│  │ 单元测试 │   │ 单元测试 │   │ 单元测试 │   │ 系统测试 │ │
│  │ 集成测试 │   │ 集成测试 │   │ 集成测试 │   │ 回归测试 │ │
│  └──────────┘   └──────────┘   └──────────┘   └──────────┘ │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 2.2 代码组织结构

```
src/main/scala/
├── xiangshan/                    # 核心微架构包
│   ├── package.scala             # 包定义、公共类型
│   ├── Parameters.scala          # 微架构参数定义
│   ├── XSTile.scala              # Tile 顶层封装
│   ├── XSCore.scala              # 核心集成
│   ├── L2Top.scala               # L2 顶层
│   ├── frontend/                 # 前端模块
│   │   ├── Frontend.scala        # 前端顶层
│   │   ├── BPU.scala             # 分支预测单元
│   │   ├── FTB.scala             # 取指目标缓冲
│   │   ├── Tage.scala            # TAGE 预测器
│   │   ├── ITTage.scala          # ITTAGE 预测器
│   │   ├── RAS.scala             # 返回地址栈
│   │   ├── FauFTB.scala          # 快速 FTB
│   │   ├── IBuffer.scala         # 指令缓冲
│   │   ├── PreDecode.scala       # 预译码
│   │   ├── NewFtq.scala          # 取指传输队列
│   │   └── icache/               # 指令缓存
│   │       ├── ICache.scala      # ICache 顶层
│   │       ├── ICacheMainPipe.scala
│   │       └── ...
│   ├── backend/                  # 后端模块
│   │   ├── Backend.scala         # 后端顶层
│   │   ├── BackendParams.scala   # 后端参数
│   │   ├── Bundles.scala         # 接口 Bundle 定义
│   │   ├── ctrlblock/            # 控制块
│   │   │   ├── MemCtrl.scala     # 内存控制
│   │   │   └── RedirectGenerator.scala
│   │   ├── decode/               # 译码
│   │   │   ├── DecodeUnit.scala
│   │   │   ├── Instructions.scala
│   │   │   ├── VecDecoder.scala
│   │   │   └── isa/
│   │   ├── dispatch/             # 分派
│   │   ├── issue/                # 发射
│   │   │   ├── IssueQueue.scala
│   │   │   ├── Bypass.scala
│   │   │   ├── Wakeup.scala
│   │   │   └── Scheduler.scala
│   │   ├── rename/               # 重命名
│   │   │   ├── FreeList.scala
│   │   │   ├── RenameTable.scala
│   │   │   └── BusyTable.scala
│   │   ├── rob/                  # 重排序缓冲
│   │   │   ├── Rob.scala
│   │   │   └── Rab.scala
│   │   ├── regfile/              # 寄存器堆
│   │   │   ├── RegFile.scala
│   │   │   └── ...
│   │   ├── exu/                  # 执行单元
│   │   ├── fu/                   # 功能单元
│   │   │   ├── ALU.scala
│   │   │   ├── Branch.scala
│   │   │   ├── CSR.scala
│   │   │   ├── Fence.scala
│   │   │   ├── Jump.scala
│   │   │   ├── Multiplier.scala
│   │   │   ├── divider/
│   │   │   ├── fpu/
│   │   │   ├── vector/
│   │   │   ├── PMA.scala
│   │   │   └── PMP.scala
│   │   ├── datapath/             # 数据通路
│   │   │   ├── Bypass.scala
│   │   │   └── ...
│   │   └── trace/                # 追踪
│   ├── cache/                    # 缓存系统
│   │   ├── L1Cache.scala
│   │   ├── dcache/               # 数据缓存
│   │   │   ├── DCache.scala
│   │   │   ├── DCacheMainPipe.scala
│   │   │   └── ...
│   │   ├── mmu/                  # 内存管理单元
│   │   │   ├── TLB.scala
│   │   │   ├── L2TLB.scala
│   │   │   ├── PTW.scala
│   │   │   └── ...
│   │   └── wpu/                  #  way prediction
│   ├── mem/                      # 内存系统
│   │   ├── MemBlock.scala        # 内存块顶层
│   │   ├── Bundles.scala         # 接口定义
│   │   ├── lsqueue/              # Load/Store 队列
│   │   │   ├── LoadQueue.scala
│   │   │   ├── StoreQueue.scala
│   │   │   └── ...
│   │   ├── pipeline/             # 流水线
│   │   │   ├── LoadPipeline.scala
│   │   │   ├── StorePipeline.scala
│   │   │   └── ...
│   │   ├── sbuffer/              # Store Buffer
│   │   ├── prefetch/             # 预取器
│   │   │   ├── SMSPrefetcher.scala
│   │   │   ├── StridePrefetcher.scala
│   │   │   └── ...
│   │   ├── mdp/                  # 内存依赖预测
│   │   │   ├── StoreSet.scala
│   │   │   ├── WaitTable.scala
│   │   │   └── ...
│   │   └── vector/               # 向量内存
│   └── transforms/               # FIRRTL 变换
├── top/                          # 顶层
│   ├── Top.scala                 # XSTop 顶层
│   ├── Configs.scala             # 配置定义
│   ├── Generator.scala           # 代码生成器
│   ├── XiangShanStage.scala      # 自定义阶段
│   ├── XSNoCTop.scala            # NoC 顶层
│   ├── ExternalLLC.scala         # 外部 LLC
│   ├── ArgParser.scala           # 命令行解析
│   ├── YamlParser.scala          # YAML 配置解析
│   └── BusPerfMonitor.scala      # 总线性能监控
├── system/                       # SoC 系统
│   └── SoC.scala                 # SoC 参数与集成
├── device/                       # 设备
│   ├── AXI4/
│   ├── standalone/
│   └── TLPMA/
└── utils/                        # 工具库
    ├── ArbiterHelper.scala
    ├── AXI4Lite.scala
    ├── DebugMem.scala
    ├── Trigger.scala
    ├── OverrideableQueue.scala
    ├── PipeWithFlush.scala
    ├── LowPowerState.scala
    └── ...
```

---

## 3. 关键模块详细设计

### 3.1 前端模块

#### 3.1.1 BPU（分支预测单元）

```scala
// BPU 顶层结构
class BPU(implicit p: Parameters) extends BasePredictor {
  // 预测器实例化
  val ftb = Module(new FTB)          // 取指目标缓冲
  val uftb = Module(new FauFTB)      // 快速 FTB
  val tage = Module(new Tage_SC)     // TAGE-SC
  val ittage = Module(new ITTage)    // ITTAGE
  val ras = Module(new RAS)          // 返回地址栈
  
  // 预测流水线
  // Stage 1: uftb 快速预测
  // Stage 2: tage + ftb
  // Stage 3: ittage + ras (backing predictor)
}
```

**关键设计要点：**

| 特性 | 实现 |
|------|------|
| 预测宽度 | 8 指令/周期 |
| FTB 大小 | 2048 entries, 4-way |
| TAGE 表 | 4 tables, 4096 entries each |
| 历史长度 | 8, 13, 32, 119 |
| ITTAGE 表 | 5 tables |
| RAS 大小 | 16 entries |
| 恢复机制 | 检查点 + 回滚 |

#### 3.1.2 ICache（指令缓存）

```scala
// ICache 参数
case class ICacheParameters(
  nSets: Int = 128,              // 64KB / 4-way / 128B line
  nWays: Int = 4,
  tagECC: Option[String] = Some("parity"),
  dataECC: Option[String] = Some("parity"),
  replacer: Option[String] = Some("setplru")
)
```

**关键设计要点：**

| 特性 | 实现 |
|------|------|
| 容量 | 64KB |
| 相联度 | 4-way |
| 行大小 | 128 bits (16 bytes) |
| 访问延迟 | 2 cycles |
| 替换策略 | Set-plru |
| ECC | Parity (tag + data) |
| 端口数 | 2 read + 1 write |

### 3.2 后端模块

#### 3.2.1 重命名单元

```scala
// 重命名接口
class RenameInterface(implicit p: Parameters) extends Bundle {
  val ren_src = Vec(RenameWidth, Vec(2, Valid(UInt(PhyRegIdxWidth.W))))
  val ren_dst = Vec(RenameWidth, Valid(UInt(PhyRegIdxWidth.W)))
  val ren_src_snapshot = Vec(RenameWidth, Valid(UInt(PhyRegIdxWidth.W)))
}
```

**关键设计要点：**

| 特性 | 实现 |
|------|------|
| 重命名宽度 | 6 指令/周期 |
| 物理寄存器 | 192 (Int) |
| 空闲列表 | 基于链表的 FreeList |
| 忙表 | 基于 CAM 的 BusyTable |
| 快照 | 4 个检查点用于恢复 |

#### 3.2.2 发射队列

```scala
// 调度器类型
sealed trait SchedulerType
case class IntScheduler() extends SchedulerType    // 整数调度器
case class FpScheduler() extends SchedulerType     // 浮点调度器
case class VfScheduler() extends SchedulerType     // 向量调度器
case class MemScheduler() extends SchedulerType    // 内存调度器
```

**调度器配置：**

| 调度器 | 发射块数 | 每块条目数 | 总条目数 |
|--------|---------|-----------|---------|
| IntScheduler | 4 | 24 | 96 |
| FpScheduler | 3 | 18 | 54 |
| VfScheduler | 3 | 16 | 48 |
| MemScheduler | 6 | 16 | 96 |

#### 3.2.3 ROB（重排序缓冲）

```scala
// ROB 参数
case class RobSize = 160
case class RabSize = 256
case class RobCommitWidth = 8
```

**关键设计要点：**

| 特性 | 实现 |
|------|------|
| 大小 | 160 entries |
| 提交宽度 | 8 指令/周期 |
| RAB | 256 entries |
| 异常处理 | 精确异常 |
| 分支恢复 | 支持多次重定向 |

### 3.3 执行单元

#### 3.3.1 ALU

```scala
// ALU 配置
case class AluCfg extends ExeUnitParams {
  val name = "ALU"
  val fuConfigs = Seq(AluCfg, MulCfg, BkuCfg)
  val wbPorts = Seq(IntWB)
}
```

**ALU 流水线：**

| 阶段 | 功能 |
|------|------|
| EX1 | 运算执行 |
| WB | 写回寄存器堆 |

#### 3.3.2 FPU（浮点单元）

```scala
// FPU 配置
case class FuConfig {
  val falu = ExeUnitParams("FALU", Seq(FaluCfg, FcvtCfg, F2vCfg, FmacCfg))
  val fdiv = ExeUnitParams("FDIV", Seq(FdivCfg))
}
```

**FPU 流水线：**

| 单元 | 延迟 | 功能 |
|------|------|------|
| FALU | 3 cycles | 浮点加/减/转换 |
| FMAC | 4 cycles | 浮点乘加 |
| FDIV | 8 cycles | 浮点除法 |

#### 3.3.3 向量单元

```scala
// 向量配置
case class VecConfig {
  val vlen = 128
  val elen = 64
  val vlenb = vlen / 8  // 16 bytes
}
```

**向量流水线：**

| 单元 | 功能 |
|------|------|
| VIALU | 向量整数运算 |
| VFMA | 向量浮点乘加 |
| VDIV | 向量除法 |
| VLSU | 向量 Load/Store |

### 3.4 存储系统

#### 3.4.1 DCache（数据缓存）

```scala
// DCache 参数
case class DCacheParameters(
  nSets: Int = 1024,           // 64KB / 4-way / 128B line
  nWays: Int = 4,
  tagECC: Option[String] = Some("secded"),
  dataECC: Option[String] = Some("secded"),
  replacer: Option[String] = Some("setplru"),
  nMissEntries: Int = 16,
  nProbeEntries: Int = 8,
  nReleaseEntries: Int = 18,
  nMaxPrefetchEntry: Int = 6
)
```

**关键设计要点：**

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
case class LoadQueueSize = 72
case class StoreQueueSize = 56
case class StoreBufferSize = 16
```

**Load 队列结构：**

| 队列 | 大小 | 功能 |
|------|------|------|
| VirtualLoadQueue | 72 | 跟踪虚拟地址 |
| LoadQueueRAR | 72 | Read-After-Read 依赖 |
| LoadQueueRAW | 32 | Read-After-Write 依赖 |
| LoadQueueReplay | 72 | 重放队列 |

**Store 队列结构：**

| 队列 | 大小 | 功能 |
|------|------|------|
| StoreQueue | 56 | 跟踪 Store 指令 |
| SBuffer | 16 | Store Buffer |

#### 3.4.3 TLB（页表缓存）

```scala
// TLB 参数
case class TLBParameters(
  name: String,
  NWays: Int = 48,
  partialStaticPMP: Boolean = true,
  outsideRecvFlush: Boolean = true,
  lgMaxSize: Int = 4
)
```

**TLB 层次：**

| TLB | 大小 | 功能 |
|-----|------|------|
| ITLB | 48 entries | 指令 TLB |
| LDTLB | 48 entries | Load TLB |
| STTLB | 48 entries | Store TLB |
| HYTLB | 48 entries | 混合 TLB |
| PFTLB | 48 entries | 预取 TLB |
| L2TLB | 48 entries | 二级 TLB |

### 3.5 预取器

#### 3.5.1 SMS 预取器

```scala
// SMS 预取参数
case class SMSParams extends PrefetcherParams {
  val useHistogram = true
  val useStride = true
  val useStream = true
}
```

**SMS 预取器特性：**

| 特性 | 实现 |
|------|------|
| 检测模式 | Spatial Memory Streaming |
| 预取距离 | 动态调整 |
| 预取度 | 1-4 |
| 训练 | 基于 L2 miss |

#### 3.5.2 BOP 预取器

```scala
// BOP 预取参数
case class BOPParameters extends PrefetcherParams {
  val useBestOffset = true
  val useRelearn = true
}
```

**BOP 预取器特性：**

| 特性 | 实现 |
|------|------|
| 检测模式 | Best Offset Prefetching |
| 训练 | 基于历史 miss 模式 |
| 预取 | 基于最佳偏移量 |

---

## 4. 接口定义

### 4.1 标准接口

```scala
// 指令接口
class DecodedInst extends Bundle {
  val instr = UInt(32.W)
  val pc = UInt(VAddrBits.W)
  val foldpc = UInt(MemPredPCWidth.W)
  val exceptionVec = UInt(13.W)
  val isVsetvl = Bool()
  val isVsetvli = Bool()
  val isVsetivli = Bool()
  val isVec = Bool()
  val is128Bit = Bool()
  val scaled1 = Bool()
  val scaled2 = Bool()
  val scaled3 = Bool()
  val isFirst = Bool()
  val isSecond = Bool()
  val isThird = Bool()
  val isFourth = Bool()
  val isFifth = Bool()
  val isSixth = Bool()
  val isSeventh = Bool()
  val isEighth = Bool()
  val isLast = Bool()
  val debug_isCnt = Bool()
}

// 重命名接口
class RenamePair extends Bundle {
  val old_pdest = UInt(PhyRegIdxWidth.W)
  val new_pdest = UInt(PhyRegIdxWidth.W)
}

// 发射接口
class IssueQueueEntry extends Bundle {
  val uop = new MicroOp
  val bankMask = UInt(4.W)
  val selectSig = UInt(4.W)
}

// 写回接口
class WbBundle extends Bundle {
  val uop = new MicroOp
  val data = UInt(DataBits.W)
  val valid = Bool()
}
```

### 4.2 关键路径时序

| 路径 | 延迟 | 说明 |
|------|------|------|
| 取指 → ICache | 2 cycles | 地址生成 + 缓存访问 |
| ICache → IBuffer | 1 cycle | 指令缓冲 |
| IBuffer → Decode | 1 cycle | 译码 |
| Decode → Rename | 1 cycle | 重命名 |
| Rename → Issue | 1 cycle | 发射队列 |
| Issue → EXU | 1 cycles | 执行单元 |
| EXU → WB | 1 cycle | 写回 |

---

## 5. 编码规范

### 5.1 命名规范

| 类型 | 规范 | 示例 |
|------|------|------|
| 模块名 | 大驼峰 | `ICache`, `BPU`, `RenameTable` |
| 信号名 | 小驼峰 | `io_req_valid`, `io_resp_bits_data` |
| 参数名 | 小驼峰 | `nSets`, `nWays`, `robSize` |
| 常量 | 大写下划线 | `FETCH_WIDTH`, `DECODE_WIDTH` |
| 方法名 | 小驼�峰 | `doRename`, `doIssue` |

### 5.2 代码风格

```scala
// 模块定义模板
class MyModule(implicit p: Parameters) extends Module {
  val io = IO(new Bundle {
    // 输入
    val in_valid = Input(Bool())
    val in_bits = Input(new MyBundle)
    
    // 输出
    val out_valid = Output(Bool())
    val out_bits = Output(new MyBundle)
  })
  
  // 内部逻辑
  // ...
}

// Bundle 定义模板
class MyBundle extends Bundle {
  val field1 = UInt(32.W)
  val field2 = Bool()
  val field3 = UInt(64.W)
}
```

### 5.3 注释规范

```scala
/**
 * MyModule 模块说明
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

## 6. 持续集成

### 6.1 CI 流程

```yaml
# CI 主要步骤
stages:
  - lint          # 代码格式检查
  - compile       # 编译检查
  - verilog       # Verilog 生成检查
  - emu           # EMU 测试
  - simv          # VCS 测试
  - coverage      # 覆盖率检查
```

### 6.2 提交规范

```
feat: 新功能
fix: 修复 bug
docs: 文档更新
style: 代码格式调整
refactor: 重构
test: 测试相关
chore: 构建/工具相关
```

### 6.3 分支策略

```
main/master       # 主分支，稳定版本
├── develop       # 开发分支
├── feature/*     # 功能分支
├── bugfix/*      # 修复分支
├── release/*     # 发布分支
└── hotfix/*      # 热修复分支
```

---

## 7. 开发阶段决策评审（ADCP）

### 7.1 评审检查清单

- [x] 各模块 RTL 代码完成
- [x] 单元测试覆盖率 > 90%
- [x] 代码评审通过
- [x] CI/CD 流程正常运行
- [x] 模块间接口对齐
- [x] 性能建模完成

### 7.2 评审结论

**通过** — 项目进入验证阶段。

---

## 8. 附录

### 8.1 Chisel 编码最佳实践

| 实践 | 说明 |
|------|------|
| 使用 `val` 而非 `var` | 函数式编程风格 |
| 避免使用 `when` 嵌套 | 提高可读性 |
| 使用 `Module` 封装 | 模块化设计 |
| 使用 `Bundle` 组织接口 | 接口清晰 |
| 使用 `DontCare` 明确未连接 | 避免误警告 |
| 使用 `assert` 进行运行时检查 | 捕获设计错误 |

### 8.2 常见设计模式

| 模式 | 用途 | 示例 |
|------|------|------|
| Pipeline | 流水线 | `Pipe` module |
| Arbiter | 仲裁 | `Arbiter` module |
| Decoupled | 握手 | `Decoupled` interface |
| Valid | 有效指示 | `Valid` interface |
| RegEnable | 使能寄存器 | `RegEnable` |

---

*文档版本：v1.0*
*最后更新：2026-10-19*
