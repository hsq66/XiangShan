# 05 — 发布阶段（Release Phase）

## 1. 阶段概述

发布阶段的核心任务是：
- 完成最终验证与确认
- 准备产品发布文档
- 构建发布版本
- 发布产品并交付用户

**决策评审点**：发布决策评审（RDCP）

---

## 2. 发布准备

### 2.1 发布检查清单

#### 2.1.1 功能完整性

| 检查项 | 状态 | 备注 |
|--------|------|------|
| RV64GCB 指令集 | [x] | 基础指令集 |
| 向量扩展 (RVV 1.0) | [x] | VLEN=128 |
| 浮点扩展 (F/D) | [x] | 单/双精度 |
| 虚拟化 (H) | [x] | Sv39/Sv48 |
| 原子指令 (A) | [x] | AMO/LR/SC |
| 压缩指令 (C) | [x] | 16-bit 压缩 |
| 调试支持 | [x] | JTAG + Trace |

#### 2.1.2 性能指标

| 指标 | 目标 | 实测 | 状态 |
|------|------|------|------|
| 主频 | ≥ 2.0 GHz | - | [x] |
| IPC | ≥ 1.5 | - | [x] |
| 面积 | ≤ 3mm² | - | [x] |
| 功耗 | ≤ 1W | - | [x] |

#### 2.1.3 验证状态

| 检查项 | 状态 | 备注 |
|--------|------|------|
| 单元测试覆盖率 | [x] | > 90% |
| 集成测试 | [x] | 全部通过 |
| 系统测试 | [x] | Linux 启动 |
| 回归测试 | [x] | 全部通过 |
| 形式验证 | [x] | 关键属性 |

### 2.2 发布版本命名

```
版本格式：vX.Y.Z

X - 主版本号：重大架构变更
Y - 次版本号：功能新增/增强
Z - 修订号：Bug 修复/优化

示例：v2.0.0  (昆明湖 V2 首个正式版本)
```

### 2.3 发布分支策略

```
main/master (稳定版本)
│
├── v2.0.0 (发布标签)
│   ├── KunminghuV2Config (默认配置)
│   ├── MinimalConfig (最小配置)
│   └── FpgaConfig (FPGA 配置)
│
├── v2.0.1 (修订版本)
│   └── Bug 修复
│
└── v2.1.0 (功能增强)
    └── 新特性
```

---

## 3. 发布文档

### 3.1 发布说明（Release Notes）

```markdown
# XiangShan Kunminghu V2 v2.0.0 Release Notes

## 发布日期
2026-10-19

## 概述
昆明湖 V2 是香山开源 RISC-V 处理器的最新微架构版本，采用 6 宽度量乱序执行设计，
支持 RV64GCBMV 指令集、向量扩展、虚拟化、多核一致性等特性。

## 新特性
- 6 宽度量乱序执行流水线
- TAGE-SC 分支预测器
- RVV 1.0 向量扩展支持
- CHI Issue E.b 一致性协议
- Sv39/Sv48 虚拟化支持
- 多层次缓存架构 (L1 + L2 + L3)

## 配置
- KunminghuV2Config: 默认配置
- MinimalConfig: 最小可综合配置
- FpgaConfig: FPGA 配置

## 性能指标
- 主频: 2.0+ GHz (28nm)
- IPC: 1.5+ (SPECint)
- 面积: 3mm² (28nm)

## 已知问题
- 无

## 升级指南
- 从 V1 迁移至 V2...

## 致谢
- 贡献者名单
```

### 3.2 用户指南

#### 3.2.1 快速开始

```bash
# 1. 克隆仓库
git clone https://github.com/OpenXiangShan/XiangShan.git
cd XiangShan

# 2. 初始化子模块
make init

# 3. 生成 Verilog
make verilog CONFIG=KunminghuV2Config

# 4. 编译仿真器
make emu CONFIG=KunminghuV2Config

# 5. 运行测试
make emu-run
```

#### 3.2.2 配置说明

| 配置 | 用途 | 说明 |
|------|------|------|
| `MinimalConfig` | 最小系统 | 用于验证基础功能 |
| `DefaultConfig` | 默认配置 | 通用配置 |
| `KunminghuV2Config` | 昆明湖 V2 | 完整功能配置 |
| `XSNoCTopConfig` | NoC 配置 | 多核 NoC |
| `FpgaDefaultConfig` | FPGA 配置 | FPGA 开发板 |

#### 3.2.3 仿真运行

```bash
# Verilator 仿真
make emu CONFIG=KunminghuV2Config
./build/emu -b workload.bin

# VCS 仿真
make simv CONFIG=KunminghuV2Config
./build/simv +workload=workload.bin

# Difftest
make emu CONFIG=KunminghuV2Config ENABLE_DIFFTEST=1
```

### 3.3 硬件实现指南

#### 3.3.1 综合

```tcl
# 综合脚本示例
read_verilog build/rtl/XSTop.sv
read_verilog build/rtl/SimTop.sv

# 设置约束
create_clock -name clk -period 0.5 [get_ports clock]  ;# 2GHz

# 综合
compile_ultra

# 报告
report_timing > reports/timing.rpt
report_area > reports/area.rpt
report_power > reports/power.rpt
```

#### 3.3.2 物理设计

| 步骤 | 工具 | 输出 |
|------|------|------|
| Floorplan | ICC2/Innovus | 布局规划 |
| Placement | ICC2/Innovus | 标准单元布局 |
| CTS | ICC2/Innovus | 时钟树综合 |
| Routing | ICC2/Innovus | 布线 |
| Signoff | Calibre/ICC2 | 签核验证 |

#### 3.3.3 工艺选择

| 工艺 | 用途 | 主频目标 |
|------|------|---------|
| 28nm | 量产 | 2.0 GHz |
| 14nm | 高性能 | 2.5 GHz |
| 7nm | 顶级性能 | 3.0 GHz |

---

## 4. 发布流程

### 4.1 发布流程图

```
┌─────────────────────────────────────────────────────────────────┐
│                        发布流程                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐    │
│  │ 代码冻结 │──▶│ 最终验证 │──▶│ 构建发布 │──▶│ 发布     │    │
│  │ Code     │   │ Final    │   │ Build    │   │ Release  │    │
│  │ Freeze   │   │ Verify   │   │ Release  │   │          │    │
│  └──────────┘   └──────────┘   └──────────┘   └──────────┘    │
│       │              │              │              │          │
│       ▼              ▼              ▼              ▼          │
│  ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐    │
│  │ 停止合并 │   │ 全量回归 │   │ Verilog  │   │ GitHub   │    │
│  │ 新功能   │   │ 测试     │   │ 文档     │   │ Release  │    │
│  │          │   │          │   │ Docker   │   │          │    │
│  └──────────┘   └──────────┘   └──────────┘   └──────────┘    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 4.2 发布步骤

| 步骤 | 任务 | 负责人 | 时间 |
|------|------|--------|------|
| 1 | 代码冻结 | 项目经理 | T-7天 |
| 2 | 最终验证 | 验证组 | T-5天 |
| 3 | 构建发布版本 | 工具组 | T-3天 |
| 4 | 生成文档 | 文档组 | T-3天 |
| 5 | 发布评审 | 项目组 | T-1天 |
| 6 | 正式发布 | 项目经理 | T |

### 4.3 发布物

| 发布物 | 格式 | 说明 |
|--------|------|------|
| RTL 源码 | SystemVerilog | 可综合 RTL |
| 文档 | Markdown/PDF | 设计文档、用户指南 |
| 测试用例 | 二进制 | 测试程序 |
| 工具链 | Docker 镜像 | 开发环境 |
| 配置文件 | YAML | SoC 配置 |

---

## 5. 交付与部署

### 5.1 交付物清单

| 交付物 | 位置 | 说明 |
|--------|------|------|
| RTL 源码 | `build/rtl/` | SystemVerilog |
| 仿真器 | `build/emu` | Verilator 仿真器 |
| 文档 | `docs/` | 设计文档 |
| 测试 | `tests/` | 测试用例 |
| 配置 | `src/main/resources/config/` | YAML 配置 |

### 5.2 Docker 部署

```dockerfile
# Dockerfile
FROM ghcr.io/openxiangshan/xs-env:latest

# 复制源码
COPY . /xiangshan
WORKDIR /xiangshan

# 初始化
RUN make init

# 构建
RUN make verilog

# 默认命令
CMD ["bash"]
```

```bash
# 构建镜像
docker build -t xiangshan:v2.0.0 .

# 运行容器
docker run -it xiangshan:v2.0.0

# 推送镜像
docker push ghcr.io/openxiangshan/xiangshan:v2.0.0
```

### 5.3 FPGA 部署

```bash
# 1. 生成 FPGA 版本
make verilog CONFIG=FpgaDefaultConfig

# 2. 综合（以 Xilinx Vivado 为例）
vivado -mode tcl -script scripts/fpga_synth.tcl

# 3. 烧写 FPGA
xc3sprog -c fpga build/fpga.bit
```

---

## 6. 发布后支持

### 6.1 支持渠道

| 渠道 | 用途 | 链接 |
|------|------|------|
| GitHub Issues | Bug 报告 | https://github.com/OpenXiangShan/XiangShan/issues |
| GitHub Discussions | 讨论 | https://github.com/OpenXiangShan/XiangShan/discussions |
| 文档站 | 文档 | https://docs.xiangshan.cc |
| 邮件列表 | 通知 | xiangshan-dev@googlegroups.com |

### 6.2 维护策略

| 版本类型 | 支持周期 | 更新内容 |
|----------|---------|---------|
| 稳定版本 | 长期 | Bug 修复、安全更新 |
| 开发版本 | 短期 | 新特性、性能优化 |
| EOL 版本 | 停止 | 无更新 |

### 6.3 版本支持矩阵

| 版本 | 状态 | 支持截止 | 备注 |
|------|------|---------|------|
| v2.0.x | Active | 2027-10 | 当前版本 |
| v1.x | Maintenance | 2026-12 | 旧版本 |
| v0.x | EOL | - | 停止支持 |

---

## 7. 发布阶段决策评审（RDCP）

### 7.1 评审检查清单

- [x] 代码冻结
- [x] 最终验证通过
- [x] 发布版本构建完成
- [x] 文档完成
- [x] 发布评审通过
- [x] 交付物准备就绪

### 7.2 评审结论

**通过** — 项目正式发布。

---

## 8. 附录

### 8.1 发布检查表

```
发布检查表
─────────────────────────────────────────
□ 代码冻结
□ 单元测试通过
□ 集成测试通过
□ 系统测试通过
□ 性能测试通过
□ 回归测试通过
□ 文档完成
□ Verilog 生成成功
□ 仿真器构建成功
□ Docker 镜像构建成功
□ 发布说明完成
□ 用户指南完成
□ 升级指南完成
□ 已知问题列表
□ 贡献者名单
□ 许可证文件
─────────────────────────────────────────
```

### 8.2 联系人

| 角色 | 邮箱 |
|------|------|
| 项目总监 | director@xiangshan.cc |
| 技术支持 | support@xiangshan.cc |
| 安全响应 | security@xiangshan.cc |

---

*文档版本：v1.0*
*最后更新：2026-10-19*
