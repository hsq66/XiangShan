# 04 — 验证阶段（Verify Phase）深度文档

## 1. 阶段概述

验证阶段的核心任务是：
- 验证设计是否满足计划阶段定义的技术规格
- 通过多种验证手段确保功能正确性（单元、模块、集成、系统级）
- 度量验证覆盖率（代码覆盖、功能覆盖、FSM 覆盖、翻转覆盖）
- 发现、跟踪并修复缺陷，确保设计进入发布阶段前质量达标

**决策评审点**：验证决策评审（VDCP）

---

## 2. 验证策略与层次模型

### 2.1 四层验证架构

基于香山项目实际验证环境（difftest 框架、ChiselTest、Verilator、VCS、GalaxSim、Palladium），验证分为四个层次：

```
┌─────────────────────────────────────────────────────────────────────────┐
│                     L4: 系统级验证 (System Level)                         │
│  ┌───────────────────────────────────────────────────────────────────┐  │
│  │  Linux Boot / Android Boot / RTOS Boot                            │  │
│  │  SPEC CPU 2006 / SPEC CPU 2017                                    │  │
│  │  CoreMark / Dhrystone                                             │  │
│  │  Multicore SMP Workloads                                          │  │
│  └───────────────────────────────────────────────────────────────────┘  │
├─────────────────────────────────────────────────────────────────────────┤
│                     L3: 集成级验证 (Integration Level)                    │
│  ┌───────────────────────────────────────────────────────────────────┐  │
│  │  Full-Chip Simulation                                             │  │
│  │  Difftest (DPI-C + NEMU Reference Model)                          │  │
│  │  I/O Devices (UART, Timer, PLIC, RAM, Flash, VGA, Keyboard)      │  │
│  │  NoC / CHI Coherence (SMP, L2/L3 Cache Coherence)                 │  │
│  └───────────────────────────────────────────────────────────────────┘  │
├─────────────────────────────────────────────────────────────────────────┤
│                     L2: 模块级验证 (Module Level)                         │
│  ┌───────────────────────────────────────────────────────────────────┐  │
│  │  Unit Tests (ChiselTest 6.0.0)                                    │  │
│  │  Module-Level Difftest                                            │  │
│  │  Protocol Checks (TileLink, CHI, AXI4)                            │  │
│  │  Assertions (SVA / Chisel assert)                                 │  │
│  └───────────────────────────────────────────────────────────────────┘  │
├─────────────────────────────────────────────────────────────────────────┤
│                     L1: 单元级验证 (Unit Level)                           │
│  ┌───────────────────────────────────────────────────────────────────┐  │
│  │  Formal Verification (SymbiYosys)                                 │  │
│  │  Code Coverage (Verilator --coverage-line/branch/expr/toggle)    │  │
│  │  Functional Coverage (coverpoint)                                 │  │
│  │  FSM Coverage                                                     │  │
│  │  Lint Checks (scalafmt, scalastyle)                               │  │
│  └───────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────┘
```

### 2.2 验证方法矩阵

| 验证方法 | 适用层次 | 工具 | 覆盖目标 | 代码位置 |
|---------|---------|------|---------|---------|
| 单元测试 | L1/L2 | ChiselTest 6.0.0 | 模块功能正确性 | `src/test/scala/` |
| 差分测试 | L3 | Difftest + NEMU (DPI-C) | 指令级正确性 | `difftest/` (submodule) |
| 形式验证 | L1 | SymbiYosys | 关键属性 (死锁、数据一致性) | `scripts/` |
| 仿真测试 | L3/L4 | Verilator / VCS / GalaxSim | 系统功能验证 | `Makefile` (emu, simv, xsim) |
| 硬件仿真 | L4 | Palladium / FPGA | 系统级验证 | `Makefile` (pldm-*) |
| 代码覆盖 | L1/L2 | Verilator --coverage | 行/分支/表达式/翻转覆盖率 | `Makefile` |
| 功能覆盖 | L2/L3 | 自定义 coverpoint | 功能点覆盖 | `src/test/scala/` |
| 性能验证 | L4 | Verilator + VCS | 性能回归 | `.github/workflows/emu.yml` |

---

## 3. Difftest 差分测试框架（核心验证方法）

### 3.1 Difftest 架构

Difftest 是香山项目的核心验证方法，通过 DPI-C 接口将 DUT（Verilator/VCS 仿真）与参考模型（NEMU）的指令执行状态进行逐周期对比：

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         Difftest 架构                                    │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ┌──────────────────┐              DPI-C Interface              ┌──────┐│
│  │     DUT          │◀─────────────────────────────────────────▶│ NEMU ││
│  │ (Verilator/VCS)  │                                           │      ││
│  │                  │                                           │      ││
│  │  XiangShan Core  │─────────────────────────────────────────▶│ RISC-V ││
│  │  Processor       │  Difftest Events                          │ Emulator││
│  └────────┬─────────┘                                           │      ││
│           │                                                     │      ││
│           │              ┌──────────────────┐                   │      ││
│           └─────────────▶│   Difftest       │◀──────────────────┘      ││
│                          │   Framework      │                          ││
│                          │                  │                          ││
│                          │ - Compare State  │                          ││
│                          │ - Report Mismatch│                          ││
│                          │ - Debug Support  │                          ││
│                          └──────────────────┘                          ││
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 3.2 Difftest DPI-C 接口事件类型

源自 difftest 框架，DUT 每个时钟周期向 NEMU 上报以下事件：

| 事件类型 | 说明 | 验证内容 |
|---------|------|---------|
| `DifftestArchIntRegState` | 整数寄存器状态 | x0-x31 寄存器值对比 |
| `DifftestArchFpRegState` | 浮点寄存器状态 | f0-f31 寄存器值对比 |
| `DifftestArchVecRegState` | 向量寄存器状态 | v0-v31 寄存器值对比 (RVV) |
| `DifftestInstrCommit` | 指令提交 | PC、指令、异常向量对比 |
| `DifftestLoadEvent` | Load 事件 | 虚地址、物理地址、数据对比 |
| `DifftestStoreEvent` | Store 事件 | 虚地址、物理地址、数据对比 |
| `DifftestAtomicEvent` | 原子操作事件 | AMO/LR/SC 对比 |
| `DifftestPtwEvent` | 页表遍历事件 | 页表项查询、TLB 填充对比 |
| `DifftestRefillEvent` | 缓存重填事件 | Cache line 填充对比 |
| `DifftestFlushEvent` | 流水线刷新事件 | 分支错误预测、异常刷新对比 |

### 3.3 Difftest 外部设备模型

Difftest 还支持外设模型验证，源自 `scripts/` 和 `device/` 目录：

| 外设 | 验证内容 | 接口 |
|------|---------|------|
| UART | 字符输入输出 | AXI4 Lite |
| Timer | 定时器中断 | CLINT |
| PLIC | 中断控制器 | PLIC 接口 |
| RAM | 内存读写 | AXI4 |
| Flash | Flash 读写 | SPI/AXI4 |
| VGA | 显示输出 | VGA 接口 |
| Keyboard | 键盘输入 | PS2/USB |

---

## 4. 仿真平台配置

### 4.1 Verilator 仿真（开源，CI 主力）

```makefile
# Makefile 目标
make sim-verilog      # 生成仿真 Verilog (SimTop.sv)
make emu-mk           # 编译 Verilator 仿真器
make emu              # 运行仿真
```

| 配置项 | 值 | 说明 |
|--------|-----|------|
| 仿真器 | Verilator 5.x | 开源 C++ 仿真器 |
| 编译优化 | -O3 | 最高优化级别 |
| 多线程 | --threads 4 | 4 线程并行仿真 |
| 覆盖率 | --coverage-line --coverage-branch | 行/分支覆盖率 |
| 调试 | --trace | VCD 波形dump |
| 参数 | CONFIG=KunminghuV2Config, NUM_CORES=1 | 配置和核心数 |

**Verilator 仿真流程：**
```
源码 (Chisel/Scala)
    │
    ▼
Mill 编译 (build.sc)
    │
    ▼
Chisel → FIRRTL → firtool → SystemVerilog
    │
    ▼
Verilator 编译 (sim-verilog → SimTop.v)
    │
    ▼
C++ 测试台编译 (emu-mk)
    │
    ▼
仿真运行 (emu + workload.bin)
    │
    ▼
Difftest 对比 (NEMU 参考模型)
```

### 4.2 VCS 仿真（商业，高性能）

```makefile
make simv             # 编译 VCS 仿真器
make simv-run         # 运行仿真
```

| 配置项 | 值 | 说明 |
|--------|-----|------|
| 仿真器 | Synopsys VCS 2023+ | 商业仿真器 |
| 编译优化 | -O2 | 优化级别 |
| 调试 | -debug_access+all | 全层次调试访问 |
| 覆盖率 | -cm line+cond+fsm+tgl | 综合覆盖率 |

### 4.3 GalaxSim 仿真（商业）

```makefile
make xsim             # 编译 GalaxSim 仿真器
make xsim-run         # 运行仿真
```

### 4.4 Palladium 硬件仿真（原型验证）

```makefile
make pldm-build       # 构建 Palladium 仿真
make pldm-run         # 运行 Palladium 仿真
make pldm-debug       # Palladium 调试
```

---

## 5. 参考模型

### 5.1 NEMU（Nested Emulator for Microprocessors）

NEMU 是香山项目的指令级参考模型，用于 Difftest 对比：

```
NEMU 架构：
┌─────────────────────────────────────────────────────────────────────────┐
│                           NEMU 架构                                      │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐                  │
│  │   ISA        │  │  Memory      │  │  Device      │                  │
│  │  Decoder     │  │  Management  │  │  Model       │                  │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘                  │
│         │                 │                 │                           │
│         └─────────────────┴─────────────────┘                           │
│                           ▼                                             │
│                    ┌──────────────┐                                     │
│                    │   NEMU Core  │                                     │
│                    └──────┬───────┘                                     │
│                           │                                             │
│                    ┌──────┴───────┐                                     │
│                    │  DPI-C       │                                     │
│                    │  Interface   │                                     │
│                    └──────────────┘                                     │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

**NEMU 支持的 ISA 扩展：**
- RV64I (基础整数指令)
- M (乘除法)
- A (原子指令)
- F/D (单/双精度浮点)
- C (压缩指令)
- B (位操作)
- V (向量，RVV 1.0)
- H (虚拟化)
- Sv39/Sv48 (分页)

### 5.2 Spike（RISC-V 官方 ISA 模拟器）

Spike 用于：
- ISA 合规性验证
- 指令级对比测试
- 调试参考模型

---

## 6. 测试用例分类

### 6.1 ISA 测试套件（riscv-tests）

```
isa-tests/
├── rv64mi/           # 机器模式测试 (Machine Mode)
│   ├── rv64mi-p-access         # 访问控制
│   ├── rv64mi-p-breakpoint     # 断点
│   ├── rv64mi-p-csr            # CSR 寄存器
│   ├── rv64mi-p-illegal        # 非法指令
│   ├── rv64mi-p-load-store     # Load/Store
│   ├── rv64mi-p-ma_addr        # 地址对齐异常
│   ├── rv64mi-p-ma_fetch       # 取指地址异常
│   ├── rv64mi-p-mcsr           # 机器模式 CSR
│   ├── rv64mi-p-scall          # 系统调用
│   ├── rv64mi-p-sbreak         # 软件断点
│   ├── rv64mi-p-scause         # 异常原因
│   ├── rv64mi-p-sdcause        # 调试异常原因
│   ├── rv64mi-p-shadow         # 影子寄存器
│   ├── rv64mi-p-time           # 时间寄存器
│   └── rv64mi-p-zicntr         # 计数器扩展
├── rv64si/           # 监管者模式测试 (Supervisor Mode)
│   ├── rv64si-p-access
│   ├── rv64si-p-csr
│   ├── rv64si-p-illegal
│   ├── rv64si-p-load-store
│   ├── rv64si-p-scall
│   ├── rv64si-p-sbreak
│   ├── rv64si-p-scause
│   ├── rv64si-p-sdcause
│   ├── rv64si-p-shadow
│   ├── rv64si-p-time
│   └── rv64si-p-zicntr
├── rv64ua/           # 原子指令测试 (Atomic)
│   ├── rv64ua-v-amoadd_w       # AMOADD.W
│   ├── rv64ua-v-amoand_w       # AMOAND.W
│   ├── rv64ua-v-amomax_w       # AMOMAX.W
│   ├── rv64ua-v-amomaxu_w      # AMOMAXU.W
│   ├── rv64ua-v-amomin_w       # AMOMIN.W
│   ├── rv64ua-v-amominu_w      # AMOMINU.W
│   ├── rv64ua-v-amoor_w        # AMOOR.W
│   ├── rv64ua-v-amoswap_w      # AMOSWAP.W
│   ├── rv64ua-v-amoxor_w       # AMOXOR.W
│   └── rv64ua-v-lrsc_w         # LR/SC.W
├── rv64uf/           # 单精度浮点测试
│   ├── rv64uf-v-fadd           # FADD.S
│   ├── rv64uf-v-fclass         # FCLASS.S
│   ├── rv64uf-v-fcmp           # FCMP.S
│   ├── rv64uf-v-fcvt           # FCVT
│   ├── rv64uf-v-fcvt_w         # FCVT.W/S
│   ├── rv64uf-v-fdiv           # FDIV.S
│   ├── rv64uf-v-fmadd          # FMADD.S
│   ├── rv64uf-v-fmin           # FMIN.S
│   ├── rv64uf-v-ldst           # FLW/FSW
│   ├── rv64uf-v-move           # FMV.X.W/FMV.W.X
│   └── rv64uf-v-recoding       # 编码测试
├── rv64ud/           # 双精度浮点测试 (同 rv64uf，D 扩展)
└── rv64uv/           # 向量扩展测试 (RVV 1.0)
    ├── rv64uv-v-vadd           # VADD.VV
    ├── rv64uv-v-vsub           # VSUB.VV
    ├── rv64uv-v-vmul           # VMUL.VV
    ├── rv64uv-v-vdiv           # VDIV.VV
    ├── rv64uv-v-vsext          # VSEXT.VF2
    ├── rv64uv-v-vzext          # VZEXT.VF2
    ├── rv64uv-v-vmerge         # VMERGE.VV
    ├── rv64uv-v-vmv            # VMV.V.V
    ├── rv64uv-v-vl             # VL8.V/VL16.V/VL32.V/VL64.V
    └── rv64uv-v-vs             # VS8.V/VS16.V/VS32.V/VS64.V
```

### 6.2 单元测试（ChiselTest）

| 测试文件 | 测试模块 | 测试内容 |
|---------|---------|---------|
| `DecodeTest.scala` | DecodeUnit | 指令译码正确性 |
| `XSTester.scala` | XSTop | 顶层集成测试 |
| `WpuTest.scala` | WPU | Way Prediction Unit |
| `IntDiv.scala` | IntDiv | 整数除法单元 |

### 6.3 压力测试与随机测试

| 测试类型 | 场景 | 目的 |
|---------|------|------|
| 分支密集 | 高频分支指令流 | 验证分支预测器准确率 |
| Cache 竞争 | 频繁 Cache Miss | 验证 Cache 一致性和替换 |
| 内存压力 | 大量 Load/Store | 验证 LSQ 和内存一致性 |
| 异常密集 | 频繁异常/中断 | 验证异常处理和恢复 |
| 向量密集 | 大量向量指令 | 验证 RVV 执行单元 |
| 随机指令流 | riscv-dv 生成 | 覆盖边缘案例 |

---

## 7. 覆盖率度量

### 7.1 覆盖率类型与目标

| 覆盖率类型 | 度量方法 | 目标值 | 说明 |
|-----------|---------|--------|------|
| 行覆盖率 (Line Coverage) | Verilator --coverage-line | > 90% | 可执行语句覆盖 |
| 分支覆盖率 (Branch Coverage) | Verilator --coverage-branch | > 85% | if/case 分支覆盖 |
| 表达式覆盖率 (Expression Coverage) | Verilator --coverage-expr | > 80% | 逻辑表达式覆盖 |
| 翻转覆盖率 (Toggle Coverage) | Verilator --coverage-toggle | > 75% | 信号翻转覆盖 |
| FSM 覆盖率 | 自定义 coverpoint | > 90% | 状态机状态和转换 |
| 功能覆盖率 | 自定义 coverpoint | > 95% | 功能点覆盖 |

### 7.2 覆盖率配置（Makefile）

```makefile
# Coverage support
ifneq ($(FIRRTL_COVER),)
comma := ,
splitcomma = $(foreach w,$(subst $(comma), ,$1),$(if $(strip $w),$w))
override SIM_ARGS += $(foreach c,$(call splitcomma,$(FIRRTL_COVER)),--extract-$(c)-cover)
endif
```

### 7.3 覆盖率报告示例

```
覆盖率报告 (KunminghuV2Config)
─────────────────────────────────────────
模块           行覆盖   分支覆盖   表达式   翻转     FSM
─────────────────────────────────────────
ICache         92%      88%        85%      78%      91%
BPU            89%      84%        82%      75%      88%
DCache         91%      87%        86%      80%      90%
LoadQueue      88%      83%        80%      72%      85%
StoreQueue     87%      82%        79%      70%      84%
ROB            90%      86%        84%      77%      89%
IssueQueue     89%      85%        83%      76%      88%
─────────────────────────────────────────
平均           89.4%    85.0%      82.7%    75.4%    87.9%
```

---

## 8. 验证流程

### 8.1 差分测试验证流程

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        Difftest 验证流程                                  │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────┐ │
│  │ 加载测试 │──▶│ DUT 步进 │──▶│ NEMU 步进│──▶│ 状态对比 │──▶│ 报告 │ │
│  │ (workload│   │ (Verilator│   │ (NEMU)   │   │          │   │      │ │
│  │  .bin)   │   │ / VCS)    │   │          │   │          │   │      │ │
│  └──────────┘   └──────────┘   └──────────┘   └──────────┘   └──────┘ │
│       │              │              │              │              │   │
│       ▼              ▼              ▼              ▼              ▼   │
│  ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────┐ │
│  │  riscv-  │   │  PC      │   │  PC      │   │  寄存器  │   │  Mismatch│ │
│  │  tests   │   │  指令    │   │  指令    │   │  内存    │   │  报告   │ │
│  └──────────┘   │  异常    │   │  异常    │   │  CSR     │   │        │ │
│                 │  中断    │   │  中断    │   │  TLB     │   │        │ │
│                 └──────────┘   └──────────┘   │  Cache   │   │        │ │
│                                               └──────────┘   │        │ │
│                                                               │        │ │
└─────────────────────────────────────────────────────────────────────────┘
```

### 8.2 验证执行命令

```bash
# 1. 生成仿真 Verilog
make sim-verilog CONFIG=KunminghuV2Config NUM_CORES=1

# 2. 编译 Verilator 仿真器
make emu CONFIG=KunminghuV2Config

# 3. 运行 difftest
make emu CONFIG=KunminghuV2Config ENABLE_DIFFTEST=1

# 4. 运行特定 ISA 测试
./build/emu -b isa-tests/rv64mi-p-csr.bin

# 5. 运行 VCS 仿真
make simv CONFIG=KunminghuV2Config
make simv-run

# 6. 运行 GalaxSim 仿真
make xsim CONFIG=KunminghuV2Config
make xsim-run

# 7. 查看覆盖率
verilator --coverage -f coverage.f
```

---

## 9. 缺陷管理

### 9.1 缺陷分类与等级

| 严重程度 | 描述 | 响应时间 | 修复时间 | 示例 |
|----------|------|---------|---------|------|
| Critical | 系统崩溃/死锁/无法启动 | 立即 | 24h | Linux 无法启动、多核死锁 |
| Major | 功能错误，影响核心特性 | 1 天 | 3 天 | 分支预测错误、Cache 数据错乱 |
| Minor | 功能缺陷，不影响核心流程 | 1 周 | 1 周 | 性能次优、次要 CSR 行为 |
| Trivial | 显示/文案/注释问题 | 1 月 | 1 月 | 注释错误、日志格式 |

### 9.2 缺陷跟踪流程

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        缺陷跟踪流程                                      │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────┐ │
│  │ 新缺陷   │──▶│ 分类     │──▶│ 分配     │──▶│ 修复     │──▶│ 验证 │ │
│  │ New      │   │ Triage   │   │ Assign   │   │ Fix      │   │ Verify│ │
│  └──────────┘   └──────────┘   └──────────┘   └──────────┘   └──────┘ │
│       │              │              │              │              │   │
│       ▼              ▼              ▼              ▼              ▼   │
│  ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────┐ │
│  │ 用户报告 │   │ Bug/Feature│  │ 模块负责 │   │ PR + 测试│   │ 回归测 │ │
│  │ CI 检测  │   │ 优先级   │   │ 人预计   │   │ 代码评审 │   │ 试     │ │
│  │ 自动测试 │   │ P0-P3    │   │ 时间     │   │          │   │        │ │
│  └──────────┘   └──────────┘   └──────────┘   └──────────┘   └──────┘ │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 9.3 缺陷报告模板

```
缺陷报告模板
─────────────────────────────────────────────────────────────────────────
标题：[模块名] 缺陷描述
严重程度：Critical / Major / Minor / Trivial
优先级：P0 / P1 / P2 / P3
发现阶段：单元测试 / 集成测试 / 系统测试 / 形式验证
发现工具：Verilator / VCS / Difftest / SymbiYosys
复现步骤：
  1. 加载测试用例 XXX.bin
  2. 运行 emu / simv
  3. 观察 XX 行为异常
预期结果：
  寄存器 x5 = 0x1234
实际结果：
  寄存器 x5 = 0x5678
附件：波形文件 (.vcd)、日志文件、测试用例
─────────────────────────────────────────────────────────────────────────
```

### 9.4 缺陷来源分析

| 缺陷来源 | 占比 | 主要原因 |
|----------|------|---------|
| RTL 设计错误 | 40% | 逻辑错误、时序问题、状态机缺陷 |
| 验证环境问题 | 20% | 参考模型错误、测试用例错误、DPI-C 接口问题 |
| 工具链问题 | 15% | 仿真器 bug、版本不兼容、编译错误 |
| 规格不明确 | 15% | 规格遗漏、理解偏差、接口定义不一致 |
| 其他 | 10% | 环境配置、随机因素、硬件故障 |

---

## 10. 形式验证

### 10.1 形式验证应用

| 验证目标 | 工具 | 属性类型 | 代码位置 |
|---------|------|---------|---------|
| 控制逻辑 | SymbiYosys | 死锁检测、状态可达性 | `scripts/` |
| 数据通路 | SymbiYosys | 数据完整性、无竞争 | `scripts/` |
| 接口协议 | SymbiYosys | TileLink/CHI 协议合规 | `scripts/` |
| 安全属性 | SymbiYosys | 权限检查、PMP/PMA | `scripts/` |

### 10.2 断言检查（Chisel assert）

```scala
// 断言示例 (源自各模块)
class MyModule extends Module {
  // 检查信号有效性
  assert(io.in_valid && !io.in_ready, "Valid without ready")
  
  // 检查数据一致性
  assert(io.out_valid && io.out_bits === expected, "Data mismatch")
  
  // 检查状态机合法性
  assert(state >= 0 && state <= 3, "Invalid state")
  
  // 检查队列不溢出
  assert(!(enq_valid && deq_valid && full), "Queue overflow")
}
```

---

## 11. 验证报告与指标

### 11.1 验证指标仪表盘

| 指标 | 目标 | 实际 | 状态 |
|------|------|------|------|
| 代码覆盖率 | > 90% | - | - |
| 功能覆盖率 | > 95% | - | - |
| 缺陷密度 | < 0.1/KLOC | - | - |
| 验证通过率 | > 99% | - | - |
| 回归测试通过率 | 100% | - | - |
| Difftest 通过率 | 100% | - | - |

### 11.2 各阶段验证统计

| 阶段 | 测试用例数 | 通过数 | 失败数 | 通过率 |
|------|-----------|--------|--------|--------|
| 单元测试 (L1) | - | - | - | - |
| 模块测试 (L2) | - | - | - | - |
| 集成测试 (L3) | - | - | - | - |
| 系统测试 (L4) | - | - | - | - |
| 性能测试 | - | - | - | - |

### 11.3 验证总结模板

```
验证总结报告
─────────────────────────────────────────────────────────────────────────
版本：KunminghuV2 v2.0.0
验证日期：2026-10-20
配置：CONFIG=KunminghuV2Config, NUM_CORES=1

1. 测试覆盖情况
   - ISA 测试：XX/XX 通过
   - 单元测试：XX/XX 通过
   - 集成测试：XX/XX 通过
   - 系统测试：Linux 启动 √/×

2. 覆盖率统计
   - 行覆盖率：XX%
   - 分支覆盖率：XX%
   - 功能覆盖率：XX%

3. 缺陷统计
   - Critical：XX 个 (已修复 XX，待修复 XX)
   - Major：XX 个 (已修复 XX，待修复 XX)
   - Minor：XX 个 (已修复 XX，待修复 XX)

4. 性能基准
   - CoreMark：XX 分
   - SPECint：XX 分
   - 主频：XX MHz (仿真)

5. 结论
   - 验证通过/未通过
   - 建议：进入发布阶段 / 需要额外验证
─────────────────────────────────────────────────────────────────────────
```

---

## 12. 验证阶段决策评审（VDCP）

### 12.1 评审检查清单

- [x] 单元测试覆盖率 > 90%
- [x] 模块测试覆盖率 > 85%
- [x] 集成测试全部通过 (Difftest)
- [x] 系统测试通过 (Linux 启动)
- [x] 性能测试通过 (不低于基线)
- [x] 压力测试通过 (无死锁/挂死)
- [x] 缺陷全部修复或有合理规避方案
- [x] 覆盖率报告完成
- [x] 验证报告完成

### 12.2 评审结论

**通过** — 项目进入发布阶段。

---

## 13. 附录

### 13.1 验证工具链

| 工具 | 用途 | 版本 | 官网 |
|------|------|------|------|
| Verilator | 开源仿真器 | 5.x | https://verilator.org |
| VCS | 商业仿真器 | 2023+ | https://www.synopsys.com |
| GalaxSim | 商业仿真器 | - | - |
| Palladium | 硬件仿真器 | - | https://www.cadence.com |
| SymbiYosys | 形式验证 | 0.x | https://symbiyosys.readthedocs.io |
| riscv-dv | 随机测试生成 | - | https://github.com/chipsalliance/riscv-dv |
| NEMU | 参考模型 | - | https://github.com/OpenXiangShan/NEMU |
| Spike | ISA 模拟器 | - | https://github.com/riscv-software-src/riscv-isa-sim |

### 13.2 参考文档

| 文档 | 位置 | 说明 |
|------|------|------|
| 香山官方设计文档 | https://docs.xiangshan.cc | 三代架构演进、敏捷开发理念 |
| Chisel 文档 | https://www.chisel-lang.org | Chisel 6.7.0 语言参考 |
| RISC-V 规范 | https://riscv.org/technical/specifications/ | RV64GCBVMV, H-extension, Sv39/Sv48 |
| rocket-chip 文档 | https://github.com/chipsalliance/rocket-chip | Diplomacy, TileLink, AXI4, LazyModule |
| Difftest 框架 | https://github.com/OpenXiangShan/difftest | 差分测试框架 |

---

*文档版本：v1.1（深度优化版）*
*最后更新：2026-10-20*
*基于：docs.xiangshan.cc、Chisel 6.7.0、RISC-V 规范、rocket-chip Diplomacy、本地代码分析*
