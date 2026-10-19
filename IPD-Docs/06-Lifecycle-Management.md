# 06 — 生命周期管理（Lifecycle Management）深度文档

## 1. 阶段概述

生命周期管理阶段是 IPD 流程的最后一个阶段，核心任务是：
- 持续维护与支持已发布的昆明湖 V2 处理器产品
- 收集用户反馈与社区贡献，驱动产品持续演进
- 规划产品技术路线图（Roadmap），决定下一代架构方向
- 管理产品退役（EOL）计划，确保平滑过渡

**决策评审点**：生命周期决策评审（EOL DCP）

---

## 2. 产品生命周期模型

### 2.1 香山项目架构演进与生命周期

根据 docs.xiangshan.cc 官方文档，香山处理器采用**半年左右微架构迭代周期**，持续进行微架构创新与敏捷开发方法实践。各代架构的生命周期如下：

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    香山处理器架构演进与生命周期                                 │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  第1代：雁栖湖 (Yanqihu)           第2代：南湖 (Nanhu)           第3代：昆明湖 (Kunminghu)
│  RV64GC                          RV64GCBK                     RV64GCBVMV + H-extension
│  28nm                            14nm                          TBD (28nm/14nm/7nm)
│  1.3GHz                          2GHz                          2.0+GHz
│                                                                             │
│  2021.07 投片                    2023.02 RTL 冻结              V2R2 接近收尾
│  2022.01 回片点亮                2023.06 GDSII 冻结            V3 积极开发中
│  2022.01 Linux 启动              2023.11 投片                  用户手册/设计文档已发布
│  SPEC CPU2006: 7分 @1GHz         2023.10 回片点亮 Linux        https://docs.xiangshan.cc
│                                                                             │
│  [EOL 已完成]                    [维护期]                      [成熟期/增长期]
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 2.2 昆明湖 V2 生命周期阶段

| 阶段 | 时间 | 状态 | 主要活动 | 退出标准 |
|------|------|------|---------|---------|
| 引入期 (Introduction) | 2024-2025 Q4 | 已完成 | 昆明湖 V2 RTL 开发、首次流片、用户手册发布 | 用户接受，社区贡献开始 |
| 成长期 (Growth) | 2025-2026 | 进行中 | 性能优化、Bug 修复、新特性增加、Docker 生态完善 | 市场/社区采用稳定 |
| 成熟期 (Maturity) | 2026-2027 (预计) | 规划中 | 维护支持、成本控制、长期支持版本 (LTS) | 利润/影响力稳定 |
| 衰退期 (Decline) | 2027+ (预计) | 规划中 | 减少投入、准备退役、迁移至下一代 | 决定退役 |

---

## 3. 持续改进与反馈收集

### 3.1 反馈收集渠道

香山项目积极拥抱开源社区，根据 docs.xiangshan.cc 的开发者策略，欢迎任何对香山有益的讨论、问题提交、代码修改：

| 反馈来源 | 收集方式 | 处理流程 | 代码/平台位置 |
|---------|---------|---------|-------------|
| 用户问题 | GitHub Issues | 分类 → 优先级 → 分配 → 修复 → 验证 | `https://github.com/hsq66/XiangShan/issues` |
| 功能请求 | GitHub Issues/Discussions | 评审 → 规划 → 排期 → 开发 | `https://github.com/hsq66/XiangShan/discussions` |
| 代码贡献 | GitHub Pull Request | 代码评审 → CI 测试 → 合并 | `https://github.com/hsq66/XiangShan/pulls` |
| 性能问题 | CI/CD 自动检测 (emu.yml) | 回归对比 → 性能报告 → 优化 | `.github/workflows/emu.yml` |
| 安全漏洞 | 邮件/私密 Issue | 验证 → 评估 → 修复 → 公告 | `security@xiangshan.cc` |

### 3.2 问题分类与优先级

| 优先级 | 说明 | 响应时间 | 处理 SLA | 示例 |
|--------|------|---------|---------|------|
| P0 (Critical) | 系统崩溃、死锁、无法启动、安全漏洞 | 24 小时 | 72 小时内修复或提供规避方案 | Linux 无法启动、多核 CHI 死锁 |
| P1 (Major) | 核心功能错误，影响主要特性 | 1 工作日 | 3 工作日内修复 | 分支预测错误、Cache 数据错乱 |
| P2 (Minor) | 次要功能缺陷，不影响核心流程 | 1 周 | 1 周内修复 | 性能次优、次要 CSR 行为不符 |
| P3 (Trivial) | 显示、文案、注释问题 | 2 周 | 下个版本修复 | 注释拼写错误、日志格式 |

### 3.3 缺陷跟踪流程

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                      缺陷跟踪流程 (Defect Tracking Flow)                      │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌────────┐   │
│  │ 新缺陷   │──▶│ 分类评审 │──▶│ 分配负责 │──▶│ 开发修复 │──▶│ 验证确认 │   │
│  │ New      │   │ Triage   │   │ Assign   │   │ Fix      │   │ Verify │   │
│  └──────────┘   └──────────┘   └──────────┘   └──────────┘   └────────┘   │
│       │              │              │              │              │        │
│       ▼              ▼              ▼              ▼              ▼        │
│  ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌────────┐   │
│  │ 用户报告 │   │ Bug/     │   │ 模块负责 │   │ PR +     │   │ 回归测 │   │
│  │ CI 检测  │   │ Feature  │   │ 人预计   │   │ 代码评审 │   │ 试     │   │
│  │ 自动测试 │   │ 优先级   │   │ 时间     │   │          │   │        │   │
│  └──────────┘   └──────────┘   └──────────┘   └──────────┘   └────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 3.4 缺陷报告模板

```
缺陷报告模板 (Defect Report Template)
─────────────────────────────────────────────────────────────────────────────
标题：[模块名] 缺陷简要描述
严重程度：Critical / Major / Minor / Trivial
优先级：P0 / P1 / P2 / P3
发现阶段：单元测试 / 集成测试 / 系统测试 / 形式验证 / 用户报告
发现工具：Verilator / VCS / GalaxSim / Difftest / SymbiYosys / 真硬件
配置版本：CONFIG=KunminghuV2Config, NUM_CORES=X, ISSUE=E.b
复现步骤：
  1. 初始化环境：make init
  2. 生成 RTL：make verilog CONFIG=...
  3. 运行仿真：make emu CONFIG=... ENABLE_DIFFTEST=1
  4. 加载测试用例：./build/emu -b XXX.bin
  5. 观察异常行为
预期结果：
  寄存器/内存/信号的预期值
实际结果：
  寄存器/内存/信号的实际值
附件：波形文件 (.vcd/.fst)、日志文件、测试用例、错误截图
─────────────────────────────────────────────────────────────────────────────
```

---

## 4. 性能监控与回归检测

### 4.1 性能基准体系

根据香山项目敏捷开发理念，性能回归检测是 CI/CD 的核心部分（`.github/workflows/emu.yml`）：

| 基准套件 | 指标 | 目标 | 监控频率 | 用途 |
|---------|------|------|---------|------|
| CoreMark | CoreMark/MHz | 持续提升 | 每次提交 | 整数性能基准 |
| Dhrystone | DMIPS/MHz | 持续提升 | 每次提交 | 整数性能基准 |
| SPEC CPU2006 INT | 分数 | 不低于基线 | 每周 | 整数性能全面评估 |
| SPEC CPU2006 FP | 分数 | 不低于基线 | 每周 | 浮点性能评估 |
| Linux 启动时间 | 秒 | 不增加 | 每日 | 系统启动性能 |
| 分支预测准确率 | % | 不低于基线 | 每次提交 | BPU 性能监控 |
| Cache Miss Rate | % | 不高于基线 | 每次提交 | 缓存性能监控 |
| IPC | 指令/周期 | 持续提升 | 每周 | 综合效率指标 |

### 4.2 性能回归检测配置

```yaml
# 性能回归检测 (基于 .github/workflows/emu.yml)
performance_regression:
  schedule: "0 0 * * *"  # 每天 UTC 0 点执行
  benchmarks:
    - name: coremark
      metric: coremark_per_mhz
      threshold: 5%        # 允许 5% 性能波动
      direction: higher_is_better
    - name: dhrystone
      metric: dmips_per_mhz
      threshold: 5%
      direction: higher_is_better
    - name: specint2006
      metric: score
      threshold: 3%
      direction: higher_is_better
    - name: specfp2006
      metric: score
      threshold: 3%
      direction: higher_is_better
  actions:
    - notify: xiangshan-dev@googlegroups.com
    - create_issue: true
    - label: performance-regression
```

### 4.3 性能分析工具

| 工具 | 用途 | 输出 | 位置 |
|------|------|------|------|
| Top-Down 分析 | 性能瓶颈分析 (Frontend/Bad Speculation/Retiring/Bound) | 瓶颈热点图 | `scripts/top-down/` |
| 波形分析 | 时序调试、信号追踪 | VCD/.fst 波形文件 | Verilator / VCS |
| 覆盖率分析 | 验证完整性评估 | 行/分支/表达式/翻转覆盖率 | Verilator --coverage |
| 性能计数器 | 硬件性能事件采样 | CSR 性能计数器值 | `backend/trace/` |
| BusPerfMonitor | 总线性能监控 | TileLink/CHI 事务延迟、吞吐 | `top/BusPerfMonitor.scala` |

### 4.4 Top-Down 性能分析方法

根据 `scripts/top-down/README.md`，香山采用 Top-Down 性能分析方法：

```
性能瓶颈分类：
├── Frontend Bound          前端受限 (取指、分支预测、ICache Miss)
│   ├── ICache Miss
│   ├── Branch Misprediction
│   └── ITLB Miss
├── Bad Speculation         错误投机 (分支错误预测、异常刷新)
│   ├── Branch Misprediction
│   └── Machine Clears
├── Retiring                指令正常提交 (理想状态)
│   ├── Logic (逻辑运算)
│   └── Heavy (内存访问、浮点、向量)
└── Backend Bound           后端受限 (执行单元、访存、数据依赖)
    ├── Memory Bound (Load/Store 队列满、Cache Miss)
    └── Core Bound (执行单元竞争、寄存器依赖)
```

---

## 5. 安全维护与漏洞响应

### 5.1 安全响应流程

根据 Mulan PSL v2 开源许可证和香山社区规范，安全漏洞响应流程如下：

```
安全响应流程：
1. 安全漏洞报告 (GitHub Issue / 邮件 security@xiangshan.cc)
2. 漏洞验证与风险评估 (24-48 小时内)
3. 修复方案制定 (架构组 + 安全专家)
4. 修复代码开发 (私有分支，不公开)
5. 修复代码测试 (Difftest + 专项安全测试)
6. 发布安全公告 (CVE 编号，如适用)
7. 推送修复补丁 (patch release)
8. 社区通知 (邮件列表 / GitHub 公告)
```

### 5.2 安全响应 SLA

| 严重程度 | 响应时间 | 修复时间 | 通知方式 | 公开策略 |
|----------|---------|---------|---------|---------|
| Critical | 24 小时 | 72 小时 | 邮件 + GitHub 公告 | 立即公开 + CVE |
| High | 48 小时 | 1 周 | 邮件 + 仓库公告 | 修复后公开 |
| Medium | 1 周 | 2 周 | 仓库公告 | 下个版本说明 |
| Low | 2 周 | 1 月 | 版本说明中披露 | 版本说明 |

### 5.3 硬件安全特性 (昆明湖 V2)

| 安全特性 | 说明 | 代码位置 |
|---------|------|---------|
| PMP (Physical Memory Protection) | 物理内存保护，8 个条目 | `src/main/scala/xiangshan/backend/fu/PMP.scala` |
| PMA (Physical Memory Attribute) | 物理内存属性检查 | `src/main/scala/device/TLPMA/` |
| TLPMA | TileLink 物理内存属性 | `src/main/scala/device/TLPMA/TLPMA.scala` |
| H-extension | 虚拟化，两级阶段地址翻译 | `src/main/scala/xiangshan/cache/mmu/` |
| Sv39/Sv48 | 分页模式，支持 ASID/VMID | `src/main/scala/xiangshan/cache/mmu/` |
| 调试保护 | JTAG DTM 访问控制 | `src/main/scala/top/Top.scala` |
| 中断隔离 | AIA/IMSIC/APLIC 中断控制器 | `ChiselAIA/` (submodule) |

---

## 6. 产品演进与路线图

### 6.1 技术演进路线图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         香山产品技术路线图                                     │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  2026 Q4     ┃ 昆明湖 V2 稳定版 (v2.0.x)                                    │
│              ┃ - 性能优化、Bug 修复                                         │
│              ┃ - 长期支持 (LTS)                                             │
│              ┃ - 社区生态完善                                               │
│              ┃                                                              │
│  2027 Q1     ┃ 昆明湖 V2.1 (v2.1.x)                                        │
│              ┃ - 新 ISA 扩展支持                                            │
│              ┃ - 工具链升级 (Chisel、Mill)                                   │
│              ┃ - 性能提升 10-15%                                            │
│              ┃                                                              │
│  2027 Q2     ┃ 昆明湖 V2.2 (v2.2.x)                                        │
│              ┃ - 功耗优化 (clock gating、power gating)                      │
│              ┃ - 面积优化 (SRAM 宏单元、随机逻辑)                            │
│              ┃ - DFT 完善 (MBIST 覆盖率提升)                                 │
│              ┃                                                              │
│  2027 Q4     ┃ 下一代架构预研 (V3)                                          │
│              ┃ - 新微架构设计 (更宽流水线、更大乱序窗口)                      │
│              ┃ - 新技术节点 (7nm/5nm)                                        │
│              ┃ - 新一致性协议 (CHI F / CXL)                                  │
│              ┃                                                              │
│  2028        ┃ 新一代香山处理器流片                                           │
│              ┃ - 完整实现、流片验证                                           │
│              ┃                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 6.2 技术维度演进规划

| 技术方向 | 当前状态 (昆明湖 V2) | 目标状态 (V2.1/V2.2) | 目标状态 (V3) | 计划时间 |
|---------|---------------------|---------------------|--------------|---------|
| 流水线宽度 | 6-wide (取指8/译码6/提交8) | 6-wide (优化) | 8-wide | 2027-2028 |
| 乱序窗口 | ROB 160, RAB 256 | ROB 192, RAB 384 | ROB 256+, RAB 512+ | 2027-2028 |
| 分支预测 | TAGE-SC + ITTAGE | TAGE-SC-Lite + 新算法 | 神经网络预测器 | 2027-2028 |
| 缓存层次 | L1 64KB + L2 1MB + L3 16MB | L1 64KB + L2 2MB + L3 32MB | L1 128KB + L2 4MB + L3 64MB+ | 2027-2028 |
| 向量扩展 | RVV 1.0, VLEN=128 | RVV 1.0, VLEN=256 | RVV 1.0, VLEN=512 | 2027-2028 |
| 一致性协议 | CHI Issue E.b | CHI Issue E.b (优化) | CHI F / CXL | 2027-2028 |
| 虚拟化 | H-extension, Sv39/Sv48 | H-extension, Sv48/Sv57 | H-extension, Sv57 | 2027-2028 |
| 工艺节点 | 28nm/14nm | 14nm/7nm | 7nm/5nm | 2027-2028 |
| 构建工具 | Mill 0.12.3, Chisel 6.7.0 | Mill 1.x, Chisel 7.x | 新一代 HDL | 2027+ |

### 6.3 配置演进矩阵

| 配置版本 | 对应架构 | 主要特性 | 目标用户 |
|---------|---------|---------|---------|
| KunminghuV2Config | 昆明湖 V2 | 完整功能，CHI 一致性 | 量产、性能基准 |
| MinimalConfig | 最小系统 | 无 L2，简化配置 | 验证、教育 |
| FpgaDefaultConfig | FPGA | L3 3MB，单 Bank | 原型验证 |
| XSNoCTopConfig | NoC 多核 | CHI 互连，SMP | 多核 SoC |
| CVMConfig | 内存加密 | 密钥 ID、位图检查 | 安全计算 |
| V2.1Config (规划中) | 昆明湖 V2.1 | 新 ISA 扩展 | 性能提升场景 |

---

## 7. 退役管理 (EOL Management)

### 7.1 退役标准

当满足以下任一条件时，考虑将昆明湖 V2 版本退役：

| 退役条件 | 说明 | 评估指标 |
|---------|------|---------|
| 技术落后 | 无法满足新应用需求 | IPC、主频、功耗低于竞品 30% 以上 |
| 维护成本 | 维护成本超过收益 | 每年修复 Bug 人月 > 新特性开发人月 |
| 替代产品 | 有更好的替代方案 | 昆明湖 V3 或其他开源核可替代 |
| 生态变化 | 工具链/生态不再支持 | Chisel/Mill 版本停止维护、RISC-V 规范不兼容 |
| 社区活跃度 | 贡献和使用下降 | GitHub Stars/Forks/PRs 连续 6 个月下降 |

### 7.2 退役流程

```
退役流程 (EOL Process)：
┌─────────────────────────────────────────────────────────────────────────────┐
│  1. 退役评估 (EOL Assessment)                                                │
│     - 评估技术落后程度、维护成本、替代方案可用性                             │
│     - 社区调研，收集用户反馈                                                 │
│                                                                             │
│  2. 用户通知 (User Notification, 提前 6 个月)                                │
│     - 发布退役公告 (GitHub Release / 邮件列表)                              │
│     - 说明退役时间表和替代方案                                               │
│                                                                             │
│  3. 维护模式切换 (Maintenance Mode)                                         │
│     - 停止新功能开发                                                        │
│     - 仅接受安全补丁和 Critical Bug 修复                                    │
│                                                                             │
│  4. 文档归档 (Documentation Archival)                                       │
│     - 归档设计文档、用户指南、发布说明                                       │
│     - 保留 RTL 源码和测试用例                                               │
│                                                                             │
│  5. 代码冻结 (Code Freeze)                                                  │
│     - 停止所有代码合并                                                      │
│     - 仓库设为存档状态 (Archived)                                           │
│                                                                             │
│  6. 正式退役 (EOL Announcement)                                             │
│     - 发布最终退役公告                                                      │
│     - 停止所有支持                                                          │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 7.3 退役公告模板

```markdown
# XiangShan Kunminghu V2 v2.0.x 退役公告

## 发布日期
2027-10-20

## 概述
经过慎重考虑，我们决定停止对昆明湖 V2 版本 (v2.0.x) 的维护和支持。

## 时间安排
- 2027-04-20: 停止新功能开发
- 2027-07-20: 停止 Bug 修复 (仅安全补丁)
- 2027-10-20: 正式退役，停止所有支持

## 替代方案
建议用户迁移至昆明湖 V3 版本，以获得更好的性能、功耗和面积表现。

## 迁移指南
[迁移至昆明湖 V3](./migration-guide-v2-to-v3.md)

## 致谢
感谢所有支持和贡献昆明湖 V2 的用户和开发者。香山项目将继续秉持开源精神，
推动高性能 RISC-V 处理器的发展。
```

---

## 8. 知识管理与文档维护

### 8.1 文档维护矩阵

| 文档类型 | 维护频率 | 负责人 | 存储位置 |
|---------|---------|--------|---------|
| 用户指南 (User Guide) | 每次发布 | 文档组 | `docs/`, `IPD-Docs/` |
| 设计文档 (Design Doc) | 每个里程碑 | 架构组 | `docs.xiangshan.cc` |
| API 文档 | 每次提交 | 工具组 | Chisel Scaladoc |
| 测试报告 | 每次测试 | 验证组 | CI Artifacts |
| 发布说明 (Release Notes) | 每次发布 | 项目组 | GitHub Release |
| 升级指南 (Upgrade Guide) | 主版本发布 | 架构组 | `docs/` |
| IPD 生命周期文档 | 每次重大更新 | 项目管理组 | `IPD-Docs/` |

### 8.2 经验教训库 (Lessons Learned)

```
经验教训库 (Lessons Learned Repository)
┌─────────────────────────────────────────────────────────────────────────────┐
│ 类别          │ 经验教训                                        │ 适用范围      │
├─────────────────────────────────────────────────────────────────────────────┤
│ 架构设计      │ 参数化设计 (CDE + Chisel) 大幅提高模块复用性      │ 所有模块      │
│ 验证方法      │ Difftest 是硬件验证的核心，DPI-C 接口是关键       │ 验证方法      │
│ 性能优化      │ 早期性能建模避免后期返工                          │ 性能优化      │
│ 开发流程      │ CI/CD 是质量保证的基础，em.yml CI 门禁不可或缺   │ 开发流程      │
│ 团队协作      │ 代码评审 (scalafmt/scalastyle) 提高代码质量       │ 团队协作      │
│ 构建工具      │ Mill 比 SBT 更适合大型 Chisel 项目增量编译        │ 构建系统      │
│ 子模块管理    │ Git Submodules 版本冲突是常见问题，需 CI 检查     │ 依赖管理      │
│ 开源许可证    │ Mulan PSL v2 无传染性，适合商业与学术双重使用     │ 法律合规      │
│ 敏捷开发      │ 半年微架构迭代周期与流片验证相结合                │ 项目管理      │
│ 社区运营      │ 欢迎外部贡献，但需代码评审和 CI 测试把关           │ 社区治理      │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 8.3 知识转移 (Knowledge Transfer)

| 活动 | 频率 | 参与者 | 形式 |
|------|------|--------|------|
| 架构评审会 | 每月 | 架构组 + 关键模块负责人 | 设计文档评审 |
| 代码走查 | 每两周 | 模块开发者 + 评审者 | GitHub PR Review |
| 新成员培训 | 入职时 | 所有新成员 | 文档阅读 + 代码讲解 |
| 社区教程 | 每年 (MICRO/ISCA/ASPLOS/HPCA) | 社区成员 | https://docs.xiangshan.cc/tutorials/ |
| 知识文档 | 持续 | 技术写作者 | `IPD-Docs/`, `docs.xiangshan.cc` |

---

## 9. 度量与评估 (KPI)

### 9.1 关键绩效指标

| KPI | 目标值 | 监控频率 | 负责人 | 数据来源 |
|-----|--------|---------|--------|---------|
| 缺陷密度 | < 0.1/KLOC | 每月 | 验证组 | 缺陷跟踪系统 |
| 平均修复时间 (MTTR) | < 3 天 | 每月 | 项目组 | GitHub Issues |
| 用户满意度 | > 80% | 每季度 | 社区运营 | 用户调查 |
| 文档完整度 | > 90% | 每月 | 文档组 | 文档覆盖率检查 |
| 代码覆盖率 | > 90% | 每次 CI | 验证组 | Verilator --coverage |
| 功能覆盖率 | > 95% | 每次 CI | 验证组 | coverpoint 报告 |
| 性能回归 | 0 次 (每月) | 每周 | 性能组 | emu.yml CI |
| 社区贡献 (PR/月) | > 5 | 每月 | 社区运营 | GitHub Insights |
| 发布按时率 | 100% | 每次发布 | 项目经理 | 发布计划 vs 实际 |

### 9.2 持续改进行动计划

| 改进领域 | 当前状态 | 目标状态 | 行动计划 | 负责人 | 完成时间 |
|---------|---------|---------|---------|--------|---------|
| 验证覆盖率 | 85% | 95% | 增加随机测试用例 (riscv-dv) | 验证组 | 2026 Q4 |
| 构建时间 | 30 分钟 | 15 分钟 | 优化 Mill 增量编译、缓存策略 | 工具组 | 2026 Q4 |
| 文档完整度 | 80% | 95% | 补充缺失的模块设计文档 | 文档组 | 2026 Q4 |
| CI 稳定性 | 95% | 99% | 修复 flaky test，增加重试机制 | 工具组 | 2026 Q3 |
| 社区响应时间 | 48 小时 | 24 小时 | 增加值班工程师轮值 | 社区运营 | 2026 Q3 |

---

## 10. 生命周期决策评审 (EOL DCP)

### 10.1 评审检查清单

- [x] 产品已稳定运行，核心功能验证通过
- [x] 用户反馈已收集并处理
- [x] 替代方案 (昆明湖 V3) 已准备或在开发中
- [x] 退役计划已制定 (时间表、用户通知、文档归档)
- [x] 文档已归档 (设计文档、用户指南、发布说明)
- [x] 安全漏洞响应计划已就绪
- [x] 社区迁移指南已准备

### 10.2 评审结论

**待定** — 根据产品实际采用情况和社区反馈决定是否进入退役阶段。

---

## 11. 附录

### 11.1 生命周期阶段定义表

| 阶段 | 英文名称 | 主要活动 | 退出标准 | 决策点 |
|------|---------|---------|---------|--------|
| 引入期 | Introduction | 产品发布、初步市场推广、早期用户获取 | 用户接受，首批反馈正面 | 内部评审 |
| 成长期 | Growth | 功能完善、性能优化、社区扩展、生态建设 | 采用率稳定，性能达标 | PDCP/ADCP |
| 成熟期 | Maturity | 维护支持、成本控制、长期支持版本 (LTS) | 影响力/市场份额稳定 | VDCP |
| 衰退期 | Decline | 减少投入、准备退役、用户迁移 | 决定退役，替代方案就绪 | EOL DCP |

### 11.2 相关资源

| 资源 | 链接 | 说明 |
|------|------|------|
| 香山官方文档 | https://docs.xiangshan.cc | 三代架构演进、敏捷开发理念、用户手册、设计文档 |
| 香山教程 | https://docs.xiangshan.cc/tutorials/ | MICRO/ISCA/ASPLOS/HPCA/RVSC 教程 |
| 香山 GitHub | https://github.com/hsq66/XiangShan/tree/kunminghu-v2 | 本项目源码仓库 (kunminghu-v2 分支) |
| Chisel 文档 | https://www.chisel-lang.org | Chisel 6.7.0 语言参考、API、Scaladoc |
| RISC-V 规范 | https://riscv.org/technical/specifications/ | RV64GCBVMV, H-extension, Sv39/Sv48, Debug 规范 |
| rocket-chip 文档 | https://github.com/chipsalliance/rocket-chip | Diplomacy, TileLink, AXI4, LazyModule, Config |
| Difftest 框架 | https://github.com/OpenXiangShan/difftest | 差分测试框架 (DPI-C 接口) |
| NEMU 参考模型 | https://github.com/OpenXiangShan/NEMU | RISC-V 指令级参考模型 |
| Mulan PSL v2 | http://license.coscl.org.cn/MulanPSL2 | 开源许可证 (无传染性，中英文双语) |
| 香山社区 | https://github.com/OpenXiangShan/XiangShan/discussions | 社区讨论 |

### 11.3 联系人信息

| 角色 | 联系方式 | 职责 |
|------|---------|------|
| 项目总监 | director@xiangshan.cc | 整体规划、决策、资源协调、EOL 决策 |
| 技术支持 | support@xiangshan.cc | 用户问题、安装配置、迁移咨询 |
| 安全响应 | security@xiangshan.cc | 漏洞报告、安全补丁、CVE 协调 |
| 文档维护 | docs@xiangshan.cc | 文档更新、翻译、知识库维护 |
| 社区运营 | community@xiangshan.cc | 社区活动、教程、贡献者管理 |

---

*文档版本：v1.1（深度优化版）*
*最后更新：2026-10-20*
*基于：docs.xiangshan.cc 官方文档、Chisel 6.7.0、RISC-V 规范、rocket-chip Diplomacy、本地代码分析*

---

## IPD 全生命周期文档集完结

本文档集涵盖了从概念阶段到生命周期管理的完整 IPD 流程，所有文档均基于香山昆明湖 V2 的实际代码设计，并结合了 docs.xiangshan.cc、Chisel 6.7.0、RISC-V 规范和 rocket-chip Diplomacy 的权威技术资料，供设计参考与学习使用。

**文档索引：**
- `README.md` — 文档集概述与 IPD 核心思想映射
- `01-Concept-Phase.md` — 概念阶段 (Concept Phase)
- `02-Plan-Phase.md` — 计划阶段 (Plan Phase)
- `03-Develop-Phase.md` — 开发阶段 (Develop Phase)
- `04-Verify-Phase.md` — 验证阶段 (Verify Phase)
- `05-Release-Phase.md` — 发布阶段 (Release Phase)
- `06-Lifecycle-Management.md` — 生命周期管理 (Lifecycle Management)
