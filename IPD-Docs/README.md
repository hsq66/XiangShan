# IPD 全生命周期文档 — 香山（XiangShan）昆明湖 V2 处理器项目

## 概述

本文档集基于 IPD（Integrated Product Development，集成产品开发）方法论，结合香山（XiangShan）开源高性能 RISC-V 处理器昆明湖 V2 微架构的实际代码设计，系统性地输出全生命周期各阶段的文档资料，供设计参考与学习。

---

## IPD 六阶段模型

IPD 将产品生命周期划分为六个阶段，每个阶段有明确的决策评审点（DCP，Decision Check Point）：

| 阶段 | 英文名称 | 核心决策点 | 关键交付物 |
|------|----------|-----------|-----------|
| 1. 概念阶段 | Concept Phase | 概念决策评审（CDCP） | 产品需求文档、项目可行性分析 |
| 2. 计划阶段 | Plan Phase | 计划决策评审（PDCP） | 产品规格说明书、项目计划 |
| 3. 开发阶段 | Develop Phase | 开发决策评审（ADCP） | 详细设计文档、RTL 代码 |
| 4. 验证阶段 | Verify Phase | 验证决策评审（VDCP） | 测试报告、验证计划 |
| 5. 发布阶段 | Release Phase | 发布决策评审（RDCP） | 产品发布说明、量产文档 |
| 6. 生命周期管理 | Lifecycle Management | 生命周期决策评审（EOL DCP） | 维护计划、演进路线图 |

---

## 文档索引

| 编号 | 文档名称 | 对应阶段 | 文件路径 |
|------|---------|---------|---------|
| 01 | 概念阶段文档 | Concept Phase | [01-Concept-Phase.md](./01-Concept-Phase.md) |
| 02 | 计划阶段文档 | Plan Phase | [02-Plan-Phase.md](./02-Plan-Phase.md) |
| 03 | 开发阶段文档 | Develop Phase | [03-Develop-Phase.md](./03-Develop-Phase.md) |
| 04 | 验证阶段文档 | Verify Phase | [04-Verify-Phase.md](./04-Verify-Phase.md) |
| 05 | 发布阶段文档 | Release Phase | [05-Release-Phase.md](./05-Release-Phase.md) |
| 06 | 生命周期管理文档 | Lifecycle Management | [06-Lifecycle-Management.md](./06-Lifecycle-Management.md) |

---

## 项目背景

### 香山（XiangShan）项目简介

香山是由中国科学院计算技术研究所、北京开源芯片研究院、鹏城实验室主导开发的开源高性能 RISC-V 处理器芯片。昆明湖（Kunminghu）V2 是其最新的微架构版本，采用 Chisel 6.7.0（Scala 嵌入式 DSL）编写，生成可综合的 SystemVerilog RTL。

### 技术特征

| 维度 | 参数 |
|------|------|
| 指令集架构 | RV64GCBMV + 扩展 |
| 流水线深度 | 13+ 级乱序执行 |
| 发射宽度 | 6 条/周期（Decode/Rename） |
| 提交宽度 | 8 条/周期 |
| 分支预测 | FTB + TAGE-SC + ITTAGE + RAS + UBTB |
| 缓存层次 | L1 I/D + L2（Inclusive）+ L3/LLC |
| 一致性协议 | CHI Issue E.b / TileLink |
| 向量扩展 | RVV 1.0，VLEN=128 |
| 虚拟化 | H-extension, Sv39/Sv48 |
| 构建工具 | Mill 0.12.3 + Chisel 6.7.0 + firtool |
| 验证工具 | Verilator / VCS / GalaxSim / Palladium |

### 代码组织结构

```
XiangShan-kunminghu-v2/
├── src/main/scala/
│   ├── xiangshan/          # 核心微架构
│   │   ├── frontend/       # 取指前端（ICache, BPU, FTQ, IBuffer）
│   │   ├── backend/        # 执行后端（Decode, Rename, Issue, ROB）
│   │   │   ├── ctrlblock/  # 控制块（MemCtrl, Redirect）
│   │   │   ├── decode/     # 译码单元
│   │   │   ├── dispatch/   # 分派单元
│   │   │   ├── issue/      # 发射逻辑（IssueQueue, Bypass, Wakeup）
│   │   │   ├── rename/     # 重命名（FreeList, RenameTable, BusyTable）
│   │   │   ├── rob/        # 重排序缓冲
│   │   │   ├── regfile/    # 寄存器堆
│   │   │   ├── exu/        # 执行单元
│   │   │   ├── fu/         # 功能单元（ALU, Branch, FPU, Vector, CSR）
│   │   │   └── datapath/   # 数据通路
│   │   ├── cache/          # 缓存（DCache, MMU/TLB）
│   │   ├── mem/            # 内存系统（LSQ, SBuffer, Prefetch）
│   │   └── transforms/     # FIRRTL 变换
│   ├── top/                # 顶层配置与生成器
│   ├── system/             # SoC 集成
│   ├── device/             # 虚拟/仿真设备
│   └── utils/              # 通用工具库
├── src/test/scala/         # 单元测试
├── build.sc                # Mill 构建定义
├── Makefile                # 构建/仿真入口
└── scripts/                # 开发与调试脚本
```

---

## IPD 核心思想在香山项目中的映射

### 1. 异步开发模式（Concurrent Development）

| IPD 概念 | 香山项目映射 |
|----------|-------------|
| 技术开发与产品开发分离 | Chisel 微架构参数化设计，技术平台独立于具体产品配置 |
| 技术预研 | 分支预测算法、预取算法、CHI 一致性协议的独立研究 |
| 异步验证 | difftest 框架独立于 RTL 开发，持续对齐参考模型（NEMU） |

###  2. 结构化流程（Structured Process）

| IPD 概念 | 香山项目映射 |
|----------|-------------|
| 阶段划分 | RTL 开发 → 仿真验证 → 综合 → 后端 → 流片 |
| 决策评审 | CI/CD 门禁（emu.yml CI 中的 verilog 检查、EMU 测试、性能测试） |
| 并行工程 | 前端/后端/存储团队并行开发，通过参数化接口对齐 |

### 3. 基于包层的模块化（Modular / Chiplink）

| IPD 概念 | 香山项目映射 |
|----------|-------------|
| 共用基础模块 CBB | rocket-chip（Diplomacy, TileLink, AXI4） |
| 异步接口 | Chisel Module 间标准化接口定义 |
| 版本管理 | Git Submodules（rocket-chip, difftest, huancun, fudian 等） |

### 4. 跨功能团队（Cross-Functional Team）

| IPD 团队角色 | 香山项目映射 |
|--------------|-------------|
| PDT 团队 | 香山项目组（架构、前端、后端、验证、物理设计） |
| 系统工程师 | 架构定义与配置参数（Configs.scala） |
| 硬件工程师 | 微架构实现（xiangshan/） |
| 验证工程师 | difftest 框架 + CI/CD |

---

## 使用指南

本档集面向以下读者：

1. **处理器架构师**：了解香山微架构的设计理念与实现细节
2. **RTL 工程师**：学习 Chisel 编写高性能 CPU 的最佳实践
3. **验证工程师**：掌握大规模硬件验证方法与 difftest 框架
4. **项目管理者**：理解 IPD 方法在硬件项目中的应用
5. **芯片设计学习者**：获取完整的开源高性能 CPU 设计参考

## 许可

本项目基于 Mulan PSL v2 许可证开源，详见 [LICENSE](../LICENSE)。

---

*文档生成日期：2026-10-19*
*对应代码分支：kunminghu-v2*
*作者：基于香山开源项目代码分析*
