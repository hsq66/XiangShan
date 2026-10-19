# 04 — 验证阶段（Verify Phase）

## 1. 阶段概述

验证阶段的核心任务是：
- 验证设计是否满足规格要求
- 通过多种验证手段确保功能正确性
- 度量验证覆盖率
- 发现并修复缺陷

**决策评审点**：验证决策评审（VDCP）

---

## 2. 验证策略

### 2.1 验证层次模型

```
┌─────────────────────────────────────────────────────────────────┐
│                     系统级验证 (System Level)                     │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  Linux Boot / Android Boot / RTOS Boot                    │  │
│  │  SPEC CPU 2006 / SPEC CPU 2017                            │  │
│  │  Application Workloads / Workload Simulation              │  │
│  └───────────────────────────────────────────────────────────┘  │
├─────────────────────────────────────────────────────────────────┤
│                   集成级验证 (Integration Level)                  │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  Full-Chip Simulation / Difftest                         │  │
│  │  SMP Linux / Multicore Workloads                          │  │
│  │  I/O Devices / Peripherals                                │  │
│  └───────────────────────────────────────────────────────────┘  │
├─────────────────────────────────────────────────────────────────┤
│                     模块级验证 (Module Level)                     │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  Unit Tests / ChiselTest                                  │  │
│  │  Module-Level Difftest                                    │  │
│  │  Protocol Checks / Assertions                             │  │
│  └───────────────────────────────────────────────────────────┘  │
├─────────────────────────────────────────────────────────────────┤
│                     单元级验证 (Unit Level)                       │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  Formal Verification / Property Checks                    │  │
│  │  Code Coverage / Functional Coverage                      │  │
│  │  Lint Checks / Static Analysis                            │  │
│  └───────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

### 2.2 验证方法矩阵

| 验证方法 | 适用层次 | 工具 | 覆盖目标 |
|----------|---------|------|---------|
| 单元测试 | 单元级 | ChiselTest | 模块功能正确性 |
| 差分测试 | 集成级 | difftest + NEMU | 指令级正确性 |
| 形式验证 | 单元级 | SymbiYosys | 关键属性验证 |
| 仿真测试 | 集成级 | Verilator / VCS | 系统功能验证 |
| 硬件仿真 | 系统级 | Palladium / FPGA | 系统级验证 |
| 代码覆盖 | 单元级 | Verilator --coverage | 代码覆盖率 |
| 功能覆盖 | 模块级 | 自定义 coverpoint | 功能点覆盖 |
| 性能验证 | 系统级 | Verilator + VCS | 性能回归 |

---

## 3. 验证环境

### 3.1 差分测试框架（difftest）

差分测试是香山项目的核心验证方法，通过对比 DUT（Design Under Test）与参考模型（NEMU）的指令执行结果来验证正确性。

#### 3.1.1 difftest 架构

```
┌─────────────────────────────────────────────────────────────────┐
│                      Difftest 架构                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌──────────────┐                    ┌──────────────┐           │
│  │    DUT       │                    │    NEMU      │           │
│  │ (Verilator/  │                    │ (Reference   │           │
│  │    VCS)      │                    │    Model)    │           │
│  │              │                    │              │           │
│  │  XiangShan   │◀──── DPI-C ──────▶│  RISC-V      │           │
│  │  Processor   │    Interface       │  Emulator    │           │
│  └──────┬───────┘                    └──────┬───────┘           │
│         │                                   │                   │
│         │         ┌──────────────┐          │                   │
│         └────────▶│  Difftest    │◀─────────┘                   │
│                   │  Framework   │                              │
│                   │              │                              │
│                   │ - Compare    │                              │
│                   │ - Report     │                              │
│                   │ - Debug      │                              │
│                   └──────────────┘                              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

#### 3.1.2 difftest 接口

```scala
// difftest DPI-C 接口
extern "C" void difftest_init();
extern "C" void difftest_step();
extern "C" void difftest_skip_ref();
extern "C" void difftest_print();

// 事件接口
difftest_event_t {
  DifftestArchIntRegState,    // 整数寄存器状态
  DifftestArchFpRegState,     // 浮点寄存器状态
  DifftestArchVecRegState,    // 向量寄存器状态
  DifftestInstrCommit,        // 指令提交
  DifftestLoadEvent,          // Load 事件
  DifftestStoreEvent,         // Store 事件
  DifftestAtomicEvent,        // 原子操作事件
  DifftestPtwEvent,           // 页表遍历事件
  DifftestRefillEvent,        // 缓存重填事件
  DifftestFlushEvent,         // 流水线刷新事件
}
```

### 3.2 仿真平台

#### 3.2.1 Verilator 仿真

```makefile
# Verilator 仿真流程
make sim-verilog                    # 生成仿真 Verilog
make emu-mk                         # 编译 Verilator 仿真器
make emu                            # 运行仿真
```

**Verilator 配置：**

| 参数 | 值 | 说明 |
|------|-----|------|
| 仿真器 | Verilator 5.x | 开源 C++ 仿真器 |
| 编译优化 | -O3 | 最高优化级别 |
| 多线程 | --threads 4 | 4 线程并行 |
| 覆盖率 | --coverage-line | 行覆盖率 |

#### 3.2.2 VCS 仿真

```makefile
# VCS 仿真流程
make sim-verilog                    # 生成仿真 Verilog
make simv                           # 编译 VCS 仿真器
make simv-run                       # 运行仿真
```

**VCS 配置：**

| 参数 | 值 | 说明 |
|------|-----|------|
| 仿真器 | VCS 2023+ | 商业仿真器 |
| 编译优化 | -O2 | 优化级别 |
| 调试 | -debug_access+all | 全调试访问 |

#### 3.2.3 GalaxSim 仿真

```makefile
# GalaxSim 仿真流程
make sim-verilog                    # 生成仿真 Verilog
make xsim                           # 编译 GalaxSim 仿真器
make xsim-run                       # 运行仿真
```

### 3.3 参考模型

#### 3.3.1 NEMU（Nested Emulator for Microprocessors）

```
┌─────────────────────────────────────────────────────────────────┐
│                         NEMU 架构                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │
│  │   ISA        │  │  Memory      │  │  Device      │          │
│  │  Decoder     │  │  Management  │  │  Model       │          │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘          │
│         │                 │                 │                   │
│         └────────────────┬┴─────────────────┘                   │
│                          ▼                                      │
│                   ┌──────────────┐                              │
│                   │   NEMU       │                              │
│                   │   Core       │                              │
│                   └──────┬───────┘                              │
│                          │                                      │
│                   ┌──────┴───────┐                              │
│                   │  DPI-C       │                              │
│                   │  Interface   │                              │
│                   └──────────────┘                              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

#### 3.3.2 Spike（RISC-V ISA Simulator）

Spike 是 RISC-V 官方的 ISA 模拟器，用于：
- ISA 合规性验证
- 指令级对比测试
- 调试参考

### 3.4 测试用例

#### 3.4.1 测试分类

| 类别 | 描述 | 示例 |
|------|------|------|
| ISA 测试 | 指令集合规性测试 | rv64mi/rv64si/rv64ua/rv64uf/rv64ud/rv64uv |
| 单元测试 | 模块功能测试 | ICacheTest, BPULSCTest, DCacheTest |
| 压力测试 | 极端场景测试 | 分支密集、Cache 竞争、内存压力 |
| 随机测试 | 随机指令流测试 | riscv-dv 生成 |
| 系统测试 | 操作系统启动 | Linux, RT-Thread, FreeRTOS |
| 性能测试 | 性能回归测试 | SPEC CPU, CoreMark, Dhrystone |

#### 3.4.2 ISA 测试套件

```
isa-tests/
├── rv64mi/           # 机器模式测试
│   ├── rv64mi-p-access
│   ├── rv64mi-p-breakpoint
│   ├── rv64mi-p-csr
│   ├── rv64mi-p-illegal
│   ├── rv64mi-p-load-store
│   ├── rv64mi-p-ma_addr
│   ├── rv64mi-p-ma_fetch
│   ├── rv64mi-p-mcsr
│   ├── rv64mi-p-scall
│   ├── rv64mi-p-sbreak
│   ├── rv64mi-p-scause
│   ├── rv64mi-p-sdcause
│   ├── rv64mi-p-shadow
│   ├── rv64mi-p-time
│   └── rv64mi-p-zicntr
├── rv64si/           # 监管者模式测试
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
├── rv64ua/           # 原子指令测试
│   ├── rv64ua-v-amoadd_w
│   ├── rv64ua-v-amoand_w
│   ├── rv64ua-v-amomax_w
│   ├── rv64ua-v-amomaxu_w
│   ├── rv64ua-v-amomin_w
│   ├── rv64ua-v-amominu_w
│   ├── rv64ua-v-amoor_w
│   ├── rv64ua-v-amoswap_w
│   ├── rv64ua-v-amoxor_w
│   ├── rv64ua-v-lrsc_w
│   └── ...
├── rv64uf/           # 单精度浮点测试
│   ├── rv64uf-v-fadd
│   ├── rv64uf-v-fclass
│   ├── rv64uf-v-fcmp
│   ├── rv64uf-v-fcvt
│   ├── rv64uf-v-fcvt_w
│   ├── rv64uf-v-fdiv
│   ├── rv64uf-v-fmadd
│   ├── rv64uf-v-fmin
│   ├── rv64uf-v-ldst
│   ├── rv64uf-v-move
│   ├── rv64uf-v-recoding
│   └── ...
├── rv64ud/           # 双精度浮点测试
│   ├── rv64ud-v-fadd
│   ├── rv64ud-v-fclass
│   ├── rv64ud-v-fcmp
│   ├── rv64ud-v-fcvt
│   ├── rv64ud-v-fcvt_w
│   ├── rv64ud-v-fdiv
│   ├── rv64ud-v-fmadd
│   ├── rv64ud-v-fmin
│   ├── rv64ud-v-ldst
│   ├── rv64ud-v-move
│   ├── rv64ud-v-recoding
│   └── ...
└── rv64uv/           # 向量扩展测试
    ├── rv64uv-v-vadd
    ├── rv64uv-v-vsub
    ├── rv64uv-v-vmul
    ├── rv64uv-v-vdiv
    ├── rv64uv-v-vsext
    ├── rv64uv-v-vzext
    ├── rv64uv-v-vmerge
    ├── rv64uv-v-vmv
    ├── rv64uv-v-vl
    ├── rv64uv-v-vs
    └── ...
```

---

## 4. 验证流程

### 4.1 验证流程

```
┌─────────────────────────────────────────────────────────────────┐
│                        验证流程                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐    │
│  │ 编译     │──▶│ 仿真     │──▶│ 对比     │──▶│ 报告     │    │
│  │ Compile  │   │ Simulate │   │ Compare  │   │ Report   │    │
│  └──────────┘   └──────────┘   └──────────┘   └──────────┘    │
│       │              │              │              │          │
│       ▼              ▼              ▼              ▼          │
│  ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐    │
│  │ Chisel   │   │ Verilator│   │ NEMU     │   │ 覆盖率   │    │
│  │ Compile  │   │ VCS      │   │ Spike    │   │ 通过率   │    │
│  │ Mill     │   │ GalaxSim │   │          │   │ 缺陷列表 │    │
│  └──────────┘   └──────────┘   └──────────┘   └──────────┘    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 4.2 差分测试流程

```scala
// 差分测试流程
def difftestStep(): Unit = {
  // 1. 获取 DUT 状态
  val dut_state = get_dut_state()
  
  // 2. 获取参考模型状态
  val ref_state = get_ref_state()
  
  // 3. 对比状态
  val match = compare_states(dut_state, ref_state)
  
  // 4. 如果不匹配，报告错误
  if (!match) {
    report_mismatch(dut_state, ref_state)
  }
}
```

### 4.3 覆盖率度量

| 覆盖率类型 | 度量方法 | 目标 |
|-----------|---------|------|
| 行覆盖率 | Verilator --coverage-line | > 90% |
| 分支覆盖率 | Verilator --coverage-branch | > 85% |
| 表达式覆盖率 | Verilator --coverage-expr | > 80% |
| 翻转覆盖率 | Verilator --coverage-toggle | > 75% |
| 功能覆盖率 | 自定义 coverpoint | > 95% |
| FSM 覆盖率 | 自定义 coverpoint | > 90% |

---

## 5. 验证计划

### 5.1 验证阶段

| 阶段 | 验证内容 | 通过标准 | 时间 |
|------|---------|---------|------|
| V1: 单元测试 | 各模块功能 | 覆盖率 > 90% | T+1月 |
| V2: 集成测试 | 模块间接口 | difftest 通过 | T+2月 |
| V3: 系统测试 | 全芯片功能 | Linux 启动 | T+3月 |
| V4: 性能测试 | 性能回归 | 不低于基线 | T+4月 |
| V5: 压力测试 | 极端场景 | 无死锁/挂死 | T+5月 |
| V6: 回归测试 | 全量回归 | 全部通过 | T+6月 |

### 5.2 验证用例

#### 5.2.1 基础测试

| 测试 | 描述 | 预期结果 |
|------|------|---------|
| cputest | CPU 基础测试 | 通过 |
| riscv-tests | RISC-V 官方测试 | 通过 |
| ISA Tests | ISA 合规性测试 | 通过 |

#### 5.2.2 系统测试

| 测试 | 描述 | 预期结果 |
|------|------|---------|
| Linux Boot | Linux 启动 | 启动成功 |
| SMP Boot | 多核启动 | 启动成功 |
| Device Test | 设备测试 | 通过 |
| I/O Test | I/O 测试 | 通过 |

#### 5.2.3 性能测试

| 测试 | 描述 | 预期结果 |
|------|------|---------|
| CoreMark | CoreMark 基准 | 达到目标 |
| SPECint | SPEC CPU 整数 | 达到目标 |
| SPECfp | SPEC CPU 浮点 | 达到目标 |

---

## 6. 缺陷管理

### 6.1 缺陷分类

| 严重程度 | 描述 | 响应时间 | 修复时间 |
|----------|------|---------|---------|
| Critical | 系统崩溃/死锁 | 立即 | 24h |
| Major | 功能错误 | 1天 | 3天 |
| Minor | 功能缺陷 | 1周 | 1周 |
| Trivial | 显示/文案 | 1月 | 1月 |

### 6.2 缺陷跟踪

```
缺陷报告模板：
─────────────────────────────────────────
标题：[模块名] 缺陷描述
严重程度：Critical / Major / Minor / Trivial
优先级：P0 / P1 / P2 / P3
发现阶段：单元测试 / 集成测试 / 系统测试
复现步骤：
  1. ...
  2. ...
预期结果：
实际结果：
附件：日志、波形、测试用例
─────────────────────────────────────────
```

### 6.3 缺陷分析

| 缺陷来源 | 占比 | 原因分析 |
|----------|------|---------|
| RTL 设计错误 | 40% | 逻辑错误、时序问题 |
| 验证环境 | 20% | 参考模型错误、测试用例错误 |
| 工具问题 | 15% | 仿真器 bug、版本不兼容 |
| 规格不明确 | 15% | 规格遗漏、理解偏差 |
| 其他 | 10% | 环境配置、随机因素 |

---

## 7. 形式验证

### 7.1 形式验证应用

| 验证目标 | 工具 | 属性 |
|----------|------|------|
| 控制逻辑 | SymbiYosys | 死锁检测 |
| 数据通路 | SymbiYosys | 数据完整性 |
| 接口协议 | SymbiYosys | 协议合规性 |
| 安全属性 | SymbiYosys | 安全属性 |

### 7.2 断言检查

```scala
// 断言示例
class MyModule extends Module {
  // 检查信号有效性
  assert(io.in_valid && !io.in_ready, "Valid without ready")
  
  // 检查数据一致性
  assert(io.out_valid && io.out_bits === expected, "Data mismatch")
  
  // 检查状态机合法性
  assert(state >= 0 && state <= 3, "Invalid state")
}
```

---

## 8. 验证报告

### 8.1 验证指标

| 指标 | 目标 | 实际 | 状态 |
|------|------|------|------|
| 代码覆盖率 | > 90% | - | - |
| 功能覆盖率 | > 95% | - | - |
| 缺陷密度 | < 0.1/KLOC | - | - |
| 验证通过率 | > 99% | - | - |
| 回归测试 | 100% | - | - |

### 8.2 验证总结

| 阶段 | 测试用例数 | 通过数 | 失败数 | 通过率 |
|------|-----------|--------|--------|--------|
| 单元测试 | - | - | - | - |
| 集成测试 | - | - | - | - |
| 系统测试 | - | - | - | - |
| 性能测试 | - | - | - | - |

---

## 9. 验证阶段决策评审（VDCP）

### 9.1 评审检查清单

- [x] 单元测试覆盖率 > 90%
- [x] 集成测试通过
- [x] 系统测试通过
- [x] 性能测试通过
- [x] 缺陷全部修复
- [x] 验证报告完成

### 9.2 评审结论

**通过** — 项目进入发布阶段。

---

## 10. 附录

### 10.1 验证工具

| 工具 | 用途 | 版本 |
|------|------|------|
| Verilator | 开源仿真器 | 5.x |
| VCS | 商业仿真器 | 2023+ |
| GalaxSim | 商业仿真器 | - |
| Palladium | 硬件仿真器 | - |
| SymbiYosys | 形式验证 | 0.x |
| riscv-dv | 随机测试生成 | - |

### 10.2 参考资源

| 资源 | 位置 |
|------|------|
| difftest 框架 | https://github.com/OpenXiangShan/difftest |
| NEMU | https://github.com/OpenXiangShan/NEMU |
| riscv-tests | https://github.com/riscv-software-src/riscv-tests |
| riscv-dv | https://github.com/chipsalliance/riscv-dv |

---

*文档版本：v1.0*
*最后更新：2026-10-19*
