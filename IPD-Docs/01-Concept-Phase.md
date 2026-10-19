# 01 — 概念阶段（Concept Phase）深度文档

## 1. 阶段概述

概念阶段是 IPD 流程的起点，核心任务是：
- 分析 RISC-V 处理器市场需求与技术可行性
- 基于香山官方架构演进路线（雁栖湖→南湖→昆明湖）定义产品概念
- 评估项目风险与资源需求
- 输出产品需求文档（PRD）与项目可行性分析报告

**决策评审点**：概念决策评审（CDCP）

---

## 2. 香山项目架构背景（基于 docs.xiangshan.cc）

### 2.1 三代架构演进

根据香山官方文档（docs.xiangshan.cc），香山处理器经历了三代架构演进，昆明湖 V2 是第三代的重要版本：

```
架构演进时间线
─────────────────────────────────────────────────────────────────────────
2019.06  香山项目启动（中国科学院计算技术研究所牵头）
  │
  ▼
2021.07  第一代：雁栖湖 (Yanqihu) 投片
  │  RV64GC, 28nm, 1.3GHz, 单核
  │  SPEC CPU2006: 7分 @1GHz (DDR4-1600)
  │  2022.01 回片点亮，运行 Linux/Debian
  │
  ▼
2023.02  第二代：南湖 (Nanhu) RTL 冻结
  │  RV64GCBK, 14nm, 2GHz, 双核
  │  2023.11 投片，GDSII 2023.06 冻结
  │
  ▼
2023.02  南湖 V2 完成，增加 MBIST 等 DFT 特性
  │  2023.04 投片，2023.10 回片启动 Linux
  │
  ▼
2024-2026 第三代：昆明湖 (Kunminghu) V2 (本项目)
  │  RV64GCBVMV + H-extension + Sv39/Sv48
  │  6-wide 乱序执行，TAGE-SC 分支预测
  │  CHI Issue E.b 一致性协议，多核 SMP
  │  V2R2 接近收尾，V3 积极开发中
─────────────────────────────────────────────────────────────────────────
```

### 2.2 香山敏捷开发理念

香山项目不仅探索微架构创新，还探索**高性能处理器的敏捷开发流程**：

- **半年左右微架构迭代周期**：保持快速迭代与流片验证
- **开源工具链**：基于 Chisel + Mill + firtool + Verilator/VCS 的开源设计流
- **Difftest 框架**：与 NutShell 处理器框架相比，增加了仿真检查点、压缩文件加载、多核验证支持
- **Mulan PSL v2 许可证**：无传染性，中英文双语，鼓励商业与学术使用

---

## 3. 市场需求分析

### 3.1 RISC-V 生态背景

| 维度 | 分析 | 依据 |
|------|------|------|
| 指令集趋势 | RISC-V 成为继 x86、ARM 之后的第三大 ISA，开源免授权费 | RISC-V International 官方生态报告 |
| 高性能需求 | AI、数据中心、边缘计算对高性能开源处理器的迫切需求 | 香山项目定位：面向世界的体系结构创新开源平台 |
| 供应链安全 | 地缘政治背景下，自主可控处理器成为战略需求 | 中国科学院支持，BOSC 联盟 16 家企业联合发起 |
| 工具链成熟度 | GCC/LLVM/Linux/BSD 对 RISC-V 支持完善 | RISC-V 规范持续演进，Sv39/Sv48 分页标准化 |

### 3.2 竞争格局与定位

| 项目 | 特点 | 香山差异化定位 |
|------|------|---------------|
| SiFive P550/P650 | 商业高性能 RISC-V IP，闭源授权 | 香山：全开源，Mulan PSL v2，社区驱动 |
| T-Head C920 | 玄铁系列，偏嵌入式/服务器 | 香山：学术与产业双导向，敏捷迭代，半年周期 |
| 香山 Kunminghu V2 | 学术级开源，最先进乱序微架构 | 6-wide 乱序，TAGE-SC，CHI 一致性，RVV 1.0，H-extension |

### 3.3 目标用户

| 用户类别 | 使用场景 | 价值主张 |
|----------|---------|---------|
| 学术研究机构 | 微架构创新验证、体系结构论文 | 开源 RTL，可修改、可实验、可流片 |
| 芯片设计公司 | 处理器 IP 参考设计、SoC 核心 | 成熟微架构，减少重复开发，降低风险 |
| 系统厂商 | 高性能 SoC 开发 | RV64GCBVMV 指令集，Linux 启动，生态成熟 |
| 教育培训 | 计算机体系结构教学与实验 | 完整开源流程，Chisel 教学友好 |

---

## 4. 产品概念定义（昆明湖 V2）

### 4.1 产品定位

> **开源、高性能、可综合的 RISC-V 处理器核，面向学术研究与产业参考，支持 6-wide 乱序执行、向量计算、虚拟化、多核一致性。**

### 4.2 核心特性与代码映射

| 特性 | 规格 | 代码位置 | 设计依据 |
|------|------|---------|---------|
| 指令集架构 | RV64GCBVMV + H-extension | `src/main/scala/xiangshan/package.scala` | RISC-V 规范 |
| 虚拟内存 | Sv39 / Sv48 两级页表 | `src/main/scala/xiangshan/cache/mmu/` | RISC-V 特权架构规范 |
| 流水线宽度 | 取指 8，译码 6，重命名 6，发射 6，提交 8 | `src/main/scala/xiangshan/Parameters.scala` | 香山微架构设计 |
| 乱序窗口 | ROB 160 条目，RAB 256 条目 | `xiangshan/Parameters.scala` | Tomasulo 算法（IBMJ 1967） |
| 分支预测 | FTB + TAGE-SC + ITTAGE + RAS + UBTB | `src/main/scala/xiangshan/frontend/BPU.scala` | 预测器协同，3 级流水线 (S1/S2/S3) |
| 指令缓存 | L1 I-Cache 64KB, 4-way, SECDED | `src/main/scala/xiangshan/frontend/icache/` | ICacheParameters |
| 数据缓存 | L1 D-Cache 64KB, 4-way, SECDED | `src/main/scala/xiangshan/cache/dcache/` | DCacheParameters |
| 二级缓存 | L2 1MB, 8-way, Inclusive, 4 banks | `coupledL2/` (submodule) | huancun 缓存框架 |
| 三级缓存 | L3 16MB, 16-way, Non-inclusive, 4 banks | `src/main/scala/top/Configs.scala` | L3CacheConfig |
| 一致性协议 | CHI Issue E.b / TileLink | `coupledL2/tl2chi/`, `openLLC/` | ARM CHI 协议，rocket-chip TileLink |
| 向量扩展 | RVV 1.0, VLEN=128, ELEN=64 | `src/main/scala/xiangshan/backend/fu/vector/` | RISC-V 向量规范 |
| 浮点单元 | F/D 扩展，FMAC 4 级流水线 | `src/main/scala/xiangshan/backend/fu/fpu/` | IEEE 754，NaN Boxing |
| 调试支持 | JTAG DTM，Trace 接口 | `src/main/scala/top/Top.scala`, `backend/trace/` | RISC-V Debug 规范 |
| DFT 支持 | MBIST, SRAM_CTL | `DFTOptionsKey`, `src/main/scala/utils/` | 可测试性设计 |

### 4.3 关键性能指标（KPI）

| 指标 | 目标值 | 参考依据 |
|------|--------|---------|
| 主频 | ≥ 2.0 GHz（28nm/14nm） | 南湖架构 14nm 达到 2GHz，昆明湖 V2 目标同等 |
| IPC | ≥ 1.5（SPECint） | 香山官方性能基准 |
| 核心面积 | ≤ 3mm²（28nm） | 雁栖湖 28nm 实测参考 |
| 功耗 | ≤ 1W @ 2GHz | 28nm 工艺估算 |
| 流水线级数 | 13+ 级 | 取指→译码→重命名→发射→执行→提交 |

---

## 5. 技术可行性分析

### 5.1 关键技术选型

| 决策点 | 选择 | 技术依据 | 代码体现 |
|--------|------|---------|---------|
| 硬件描述语言 | Chisel 6.7.0 (Scala 2.13.15) | 面向对象+函数式编程，FIRRTL 编译，敏感硬件设计 | `build.sc` `chisel` -> `6.7.0` |
| 构建工具 | Mill 0.12.3 | 比 SBT 更快的增量编译，适合大型 Chisel 项目 | `.mill-version`, `build.sc` |
| 总线抽象 | rocket-chip Diplomacy (TileLink) | LazyModule 模式，参数化总线节点 | `src/main/scala/top/Top.scala` `BaseXSSoc` |
| 一致性协议 | CHI Issue E.b | ARM AMBA CHI，面向多核一致性 | `coupledL2/tl2chi/`, `EnableCHI` 配置 |
| 参考模型 | NEMU + Spike | NEMU 用于 difftest，Spike 用于 ISA 验证 | `difftest/` (submodule) |
| 仿真器 | Verilator + VCS | 开源+商业双覆盖 | `Makefile` `emu`, `simv` 目标 |
| 开源许可证 | Mulan PSL v2 | 无传染性，中英文双语 | `LICENSE`, 文件头版权声明 |

### 5.2 技术风险评估

| 风险 ID | 风险描述 | 概率 | 影响 | 缓解措施 | 代码/流程体现 |
|---------|---------|------|------|---------|-------------|
| R01 | Chisel 6.7.0 版本升级导致不兼容 | 中 | 高 | 锁定版本，建立升级测试流程 | `build.sc` 固定 `chisel 6.7.0`, `.mill-version 0.12.3` |
| R02 | 乱序执行验证覆盖率不足 | 高 | 高 | 加强 difftest，引入形式验证 | `difftest/` 框架，`.github/workflows/emu.yml` CI 门禁 |
| R03 | CHI 一致性协议死锁 | 中 | 高 | 协议仿真验证，死锁检测 | `coupledL2/tl2chi/CHILogger`, `openLLC/OpenNCB` |
| R04 | 物理设计时序不收敛 | 中 | 中 | 早期时序预算，多轮迭代 | `Makefile` 综合流程，`scripts/vlsi_mem_gen` SRAM 分割 |
| R05 | 关键人员流失 | 低 | 高 | 知识文档化，交叉培训 | 本 IPD 文档集，代码注释 |
| R06 | 子模块版本冲突 | 中 | 中 | 严格版本管理，CI 集成测试 | `.gitmodules`, `Makefile init/init-force` |

---

## 6. 资源与里程碑规划

### 6.1 团队结构（跨功能团队）

```
香山项目团队（IPD 跨功能团队）
├── 项目总监 (1人)           # 整体规划、资源协调、CDCP/PDCP/ADCP/VDCP/RDCP 决策
├── 架构组 (2人)             # 微架构定义、性能建模、ISA 扩展定义
│   └── 负责：Configs.scala, Parameters.scala, Top.scala
├── 前端组 (3人)             # 取指、分支预测、ICache、ITLB
│   └── 负责：xiangshan/frontend/ (BPU, Tage, ITTage, RAS, ICache, IBuffer, NewFtq)
├── 后端组 (4人)             # 译码、重命名、发射、执行、ROB、异常处理
│   └── 负责：xiangshan/backend/ (DecodeUnit, RenameTable, IssueQueue, Rob, Exu, Fu)
├── 存储组 (3人)             # DCache、LSQ、TLB、预取器、StoreBuffer
│   └── 负责：xiangshan/cache/, xiangshan/mem/ (DCache, LoadQueue, StoreQueue, PTW, Prefetcher)
├── 验证组 (4人)             # difftest、单元测试、系统测试、性能回归
│   └── 负责：difftest/, src/test/scala/, .github/workflows/
├── 物理设计组 (2人)         # 综合、P&R、时序收敛、DFT
│   └── 负责：Makefile, scripts/vlsi_mem_gen, DFTOptionsKey
└── 工具工程师 (1人)          # 开发环境、CI/CD、工具链
    └── 负责：build.sc, Makefile, scripts/, Dockerfile
```

### 6.2 里程碑规划（半年迭代周期）

| 里程碑 | 时间 | 交付物 | 决策点 |
|--------|------|--------|--------|
| M1: 基础流水线 | T+3 个月 | 顺序执行核，通过基础 ISA 测试 | 内部评审 |
| M2: 乱序执行 | T+6 个月 | 乱序核，通过 difftest 对比 NEMU | CDCP |
| M3: 缓存层次 | T+9 个月 | L1+L2+L3，通过性能测试 | PDCP |
| M4: 向量扩展 | T+12 个月 | RVV 1.0，通过向量测试套件 | 内部评审 |
| M5: 多核一致性 | T+15 个月 | CHI 一致性，SMP Linux 启动 | ADCP |
| M6: 流片版本 | T+18 个月 | GDSII，回片测试 | VDCP/RDCP |

---

## 7. 产品需求文档（PRD）摘要

```yaml
product_name: XiangShan Kunminghu V2
product_type: Open-source RISC-V Processor Core
vendor: ICT-CAS / BOSC / PCL
license: Mulan PSL v2

target_process:
  - 28nm (验证节点)
  - 14nm (量产目标)

isa:
  base: rv64i
  extensions:
    - m: 整数乘除法
    - a: 原子指令
    - f: 单精度浮点
    - d: 双精度浮点
    - c: 压缩指令
    - b: 位操作
    - v: 向量 (RVV 1.0, VLEN=128)
    - h: 超级访问/虚拟化 (H-extension)
    - zicntr, zicsr, zifencei 等标准扩展

paging:
  - sv39
  - sv48

microarchitecture:
  pipeline: Out-of-order, superscalar
  stages: 13+
  fetch_width: 8 instructions/cycle
  decode_width: 6 instructions/cycle
  rename_width: 6 instructions/cycle
  issue_width: 6 instructions/cycle
  commit_width: 8 instructions/cycle
  rob_size: 160 entries
  rab_size: 256 entries
  int_phys_regs: 192
  fp_phys_regs: 192
  vf_phys_regs: 128

cache_hierarchy:
  l1_i: 64KB, 4-way, 2-cycle, SECDED
  l1_d: 64KB, 4-way, 2-cycle, SECDED
  l2: 1MB, 8-way, inclusive, 4-banks
  l3: 16MB, 16-way, non-inclusive, 4-banks (optional)

branch_prediction:
  - ftb: 2048 entries, 4-way
  - tage_sc: 4 tables, 4096 entries each
  - ittage: 5 tables
  - ras: 16 entries
  - ubtb: 256 entries

memory_system:
  load_queue: 72 entries
  store_queue: 56 entries
  store_buffer: 16 entries
  prefetch: SMS + BOP + TP
  l1_tlb: 48 entries (I/D/S/H/PF)
  l2_tlb: 48 entries

coherence:
  protocol: CHI Issue E.b
  interconnect: NoC (Network on Chip)
  llc: OpenLLC

virtualization:
  extensions: H-extension
  paging: Sv39/Sv48
  tlb: Two-level (L1 + L2)

build_system:
  language: Chisel 6.7.0 (Scala 2.13.15)
  build_tool: Mill 0.12.3
  rtl_output: SystemVerilog (via firtool)
  simulators: Verilator, VCS, GalaxSim, Palladium
  verification: difftest + NEMU

agile_process:
  iteration_cycle: ~6 months
  tapeout_cycle: ~6 months
  ci_cd: GitHub Actions (emu.yml)
```

---

## 8. 概念阶段决策评审（CDCP）

### 8.1 评审检查清单

- [x] 市场需求明确，目标用户清晰（学术/产业/教育）
- [x] 产品概念完整（昆明湖 V2，RV64GCBVMV + H-extension）
- [x] 技术可行性得到验证（Chisel 6.7.0 + Mill + rocket-chip Diplomacy）
- [x] 资源需求评估完成（跨功能团队，12-18个月周期）
- [x] 风险识别与应对策略就绪（6 项主要风险）
- [x] 里程碑计划合理（半年迭代，6 个里程碑）
- [x] 团队结构完整（7 个职能组）
- [x] 与香山官方架构演进路线一致（第三代昆明湖）

### 8.2 评审结论

**通过** — 项目进入计划阶段。

---

## 9. 参考资源

| 资源 | 链接 | 说明 |
|------|------|------|
| 香山官方文档 | https://docs.xiangshan.cc | 三代架构演进、敏捷开发理念、微架构设计文档 |
| Chisel 文档 | https://www.chisel-lang.org | Chisel 6.7.0 语言参考、FIRRTL 编译器 |
| RISC-V 规范 | https://riscv.org/technical/specifications/ | RV64GCBVMV、H-extension、Sv39/Sv48 |
| rocket-chip 文档 | https://github.com/chipsalliance/rocket-chip | Diplomacy、TileLink、AXI4、LazyModule |
| 香山 GitHub | https://github.com/OpenXiangShan/XiangShan | 源码仓库，Release v1.0/v2.0/v2.1 |
| Mulan PSL v2 | http://license.coscl.org.cn/MulanPSL2 | 开源许可证 |

---

*文档版本：v1.1（深度优化版）*
*最后更新：2026-10-20*
*基于：docs.xiangshan.cc 官方文档、本地 Chisel 代码分析*
