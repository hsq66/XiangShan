# IPD 全生命周期文档 — 香山（XiangShan）昆明湖 V2 处理器项目

## 文档集概述

本文档集基于 **IPD（Integrated Product Development，集成产品开发）** 方法论，结合香山（XiangShan）开源高性能 RISC-V 处理器昆明湖 V2 微架构的实际代码设计，系统性地输出全生命周期各阶段的深度技术文档。

---

## 技术背景与参考资料

| 参考资料 | 内容 | 用途 |
|----------|------|------|
| 香山官方设计文档 | https://docs.xiangshan.cc | 香山三代架构演进（雁栖湖→南湖→昆明湖）、敏捷开发理念、微架构设计文档 |
| Chisel 6.7.0 文档 | https://www.chisel-lang.org | 硬件构造语言，Scala 嵌入式 DSL，FIRRTL 编译器，敏感硬件设计方法论 |
| RISC-V 规范 | https://riscv.org | RV64GCBVMV 指令集，H-extension 虚拟化，Sv39/Sv48 分页，Zicntr 计数器等 |
| rocket-chip 文档 | https://github.com/chipsalliance/rocket-chip | Diplomacy 总线抽象，TileLink 协议，AXI4 桥接，LazyModule 模式 |

---

## 香山架构演进与项目定位

根据 docs.xiangshan.cc 官方文档，香山处理器经历了三代架构演进：

| 代号 | 架构 | 指令集 | 工艺 | 频率 | 状态 |
|------|------|--------|------|------|------|
| 第一代 | 雁栖湖 (Yanqihu) | RV64GC | 28nm | 1.3GHz | 2021 年 7 月投片，2022 年 1 月回片点亮 |
| 第二代 | 南湖 (Nanhu) | RV64GCBK | 14nm | 2GHz | 2023 年 2 月 RTL 冻结，2023 年 11 月投片 |
| 第三代 | 昆明湖 (Kunminghu) | RV64GCBVMV + H-extension | TBD | 2.0+GHz | V2R2 接近收尾，V3 积极开发中 |

**昆明湖 V2 是香山第三代微架构**，采用 Chisel 6.7.0（Scala 2.13.15）编写，基于 rocket-chip Diplomacy 框架构建，生成可综合的 SystemVerilog RTL，采用 Mulan PSL v2 开源许可证。

---

## IPD 六阶段模型

| 阶段 | 英文 | 决策评审点 | 核心交付物 |
|------|------|-----------|-----------|
| 1. 概念阶段 | Concept Phase | CDCP | 产品需求文档、可行性分析 |
| 2. 计划阶段 | Plan Phase | PDCP | 产品规格说明书、系统架构设计、项目计划 |
| 3. 开发阶段 | Develop Phase | ADCP | 详细设计、RTL 代码、单元测试 |
| 4. 验证阶段 | Verify Phase | VDCP | 测试报告、覆盖率报告、缺陷修复记录 |
| 5. 发布阶段 | Release Phase | RDCP | 发布版本、文档、交付物 |
| 6. 生命周期管理 | Lifecycle Management | EOL DCP | 维护计划、演进路线图、退役计划 |

---

## 文档索引

| 编号 | 文档 | 阶段 | 路径 |
|------|------|------|------|
| 01 | 概念阶段深度文档 | Concept | `IPD-Docs/01-Concept-Phase.md` |
| 02 | 计划阶段深度文档 | Plan | `IPD-Docs/02-Plan-Phase.md` |
| 03 | 开发阶段深度文档 | Develop | `IPD-Docs/03-Develop-Phase.md` |
| 04 | 验证阶段深度文档 | Verify | `IPD-Docs/04-Verify-Phase.md` |
| 05 | 发布阶段深度文档 | Release | `IPD-Docs/05-Release-Phase.md` |
| 06 | 生命周期管理深度文档 | Lifecycle | `IPD-Docs/06-Lifecycle-Management.md` |

---

## IPD 核心思想在香山项目中的映射

### 1. 异步开发模式（Concurrent Development）

| IPD 概念 | 香山项目实现 | 代码位置 |
|----------|-------------|---------|
| 技术预研与产品开发分离 | 分支预测算法（TAGE-SC/ITTAGE）、预取算法（SMS/BOP/TP）独立于核心流水线开发 | `src/main/scala/xiangshan/frontend/BPU.scala`, `src/main/scala/xiangshan/mem/prefetch/` |
| 异步验证 | difftest 框架独立于 RTL 开发，持续与 NEMU 参考模型对齐 | `difftest/` (submodule) |
| 敏捷开发 | 半年微架构迭代周期，持续流片验证，基于开源工具链 | `Makefile`, `build.sc`, `.github/workflows/emu.yml` |

### 2. 基于包层的模块化（Modular / Package-based）

| IPD 概念 | 香山项目实现 | 代码位置 |
|----------|-------------|---------|
| 共用基础模块 CBB | rocket-chip Diplomacy（TileLink, AXI4, Diplomatic 节点） | `rocket-chip/` (submodule), `build.sc` 引用 |
| 异步接口 | Chisel Module 与 LazyModule 分离，通过 IO Bundle 标准化接口 | `src/main/scala/xiangshan/frontend/Frontend.scala`, `Backend.scala` |
| 版本管理 | Git Submodules 管理子模块 | `.gitmodules`, `Makefile init` |

### 3. 跨功能团队（Cross-Functional Team）

| IPD 团队角色 | 香山项目映射 | 负责模块 |
|--------------|-------------|---------|
| 系统工程师 | 架构定义与配置参数 | `src/main/scala/top/Configs.scala`, `xiangshan/Parameters.scala` |
| 硬件工程师（前端） | 取指与分支预测 | `src/main/scala/xiangshan/frontend/` |
| 硬件工程师（后端） | 乱序执行核心 | `src/main/scala/xiangshan/backend/` |
| 硬件工程师（存储） | 缓存与访存 | `src/main/scala/xiangshan/cache/`, `src/main/scala/xiangshan/mem/` |
| 验证工程师 | Difftest + CI/CD | `difftest/`, `src/test/scala/`, `.github/workflows/` |
| 物理设计工程师 | 综合与后端 | `Makefile`, `scripts/vlsi_mem_gen` |

---

## 使用指南

本档集面向以下读者：

1. **处理器架构师**：了解香山昆明湖 V2 微架构的设计理念与实现细节，学习 IPD 方法在高性能处理器开发中的应用
2. **RTL 工程师**：学习 Chisel 6.7.0 编写高性能 CPU 的最佳实践，理解 Diplomacy 总线抽象与 LazyModule 模式
3. **验证工程师**：掌握大规模硬件验证方法与 difftest 框架，理解 CI/CD 门禁机制
4. **项目管理者**：理解 IPD 方法在硬件项目中的完整应用，从概念到生命周期管理
5. **芯片设计学习者**：获取完整的开源高性能 CPU 设计参考，从架构定义到流片验证

---

## 许可

本项目基于 Mulan PSL v2 许可证开源，详见 [LICENSE](../LICENSE)。Mulan PSL v2 无传染性，中英文版本具有同等法律效力。

---

*文档生成日期：2026-10-20*
*对应代码分支：kunminghu-v2*
*参考资料：香山官方文档 (docs.xiangshan.cc)、Chisel 6.7.0、RISC-V 规范、rocket-chip Diplomacy*
