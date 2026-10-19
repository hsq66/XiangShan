# 05 — 发布阶段（Release Phase）深度文档

## 1. 阶段概述

发布阶段的核心任务是：
- 完成最终验证与确认，确保设计满足所有发布标准
- 准备产品发布文档（Release Notes、用户指南、升级指南、已知问题列表）
- 构建发布版本（RTL 源码、仿真器、Docker 镜像、配置文件、测试用例）
- 执行发布决策评审（RDCP），正式发布产品并交付用户

**决策评审点**：发布决策评审（RDCP）

---

## 2. 发布准备与检查清单

### 2.1 发布前功能完整性检查

基于昆明湖 V2 的实际微架构特性，发布前必须验证以下功能模块：

| 功能模块 | 检查项 | 验证方法 | 状态 |
|---------|--------|---------|------|
| 指令集架构 | RV64GCBVMV 完整指令集 | riscv-tests 套件 (rv64mi/si/ua/uf/ud/uv) | [x] |
| 浮点扩展 | F/D 扩展，NaN Boxing | rv64uf/rv64ud 测试 | [x] |
| 向量扩展 | RVV 1.0, VLEN=128 | rv64uv 测试套件 | [x] |
| 虚拟化 | H-extension, Sv39/Sv48 | 页表遍历测试、两级 TLB | [x] |
| 原子指令 | A 扩展, LR/SC, AMO | rv64ua 测试 | [x] |
| 压缩指令 | C 扩展 (16-bit) | 压缩指令测试 | [x] |
| 分支预测 | FTB + TAGE-SC + ITTAGE + RAS + UBTB | 分支密集型基准 | [x] |
| 缓存层次 | L1 I/D + L2 (Inclusive) + L3/LLC | Cache 压力测试 | [x] |
| 一致性协议 | CHI Issue E.b, TileLink | SMP Difftest, NoC 压力测试 | [x] |
| 调试支持 | JTAG DTM, Trace 接口 | 调试访问测试 | [x] |
| DFT 支持 | MBIST, SRAM_CTL | DFT 测试模式 | [x] |
| 中断控制 | AIA/IMSIC/APLIC (ChiselAIA) | 中断分发测试 | [x] |
| 内存保护 | PMP, PMA, TLPMA | 访存权限测试 | [x] |

### 2.2 性能指标检查

| 指标 | 目标值 | 测试方法 | 验证依据 |
|------|--------|---------|---------|
| 主频 | ≥ 2.0 GHz (28nm/14nm) | 综合时序报告 | 南湖架构 14nm 2GHz 参考 |
| IPC | ≥ 1.5 (SPECint) | SPEC CPU2006/2017 | 香山官方性能基准 |
| 核心面积 | ≤ 3mm² (28nm) | 综合面积报告 | 雁栖湖 28nm 实测 |
| 功耗 | ≤ 1W @ 2GHz | 功耗仿真 | 28nm 工艺估算 |
| 流水线级数 | 13+ 级 | 微架构设计文档 | 前端→后端→提交 |
| 乱序窗口 | ROB 160, RAB 256 | 微架构参数 | Parameters.scala |

### 2.3 验证状态检查

| 检查项 | 标准 | 验证工具 | 状态 |
|--------|------|---------|------|
| 单元测试覆盖率 | > 90% | ChiselTest + Verilator --coverage | [x] |
| 集成测试 | Difftest 全部通过 | difftest + NEMU | [x] |
| 系统测试 | Linux 启动成功 | 全芯片仿真 | [x] |
| 回归测试 | 100% 通过 | CI/CD (emu.yml) | [x] |
| 形式验证 | 关键属性通过 | SymbiYosys | [x] |
| 性能回归 | 不低于基线 | SPEC/CoreMark | [x] |

---

## 3. 发布版本命名与分支策略

### 3.1 版本命名规范

```
版本格式：vX.Y.Z[-suffix]

X - 主版本号：重大架构变更（如 雁栖湖→南湖→昆明湖）
Y - 次版本号：功能新增/增强（如 V2→V2R2→V3）
Z - 修订号：Bug 修复/性能优化/安全补丁
suffix：-rc1, -rc2 (候选版本), -beta (测试版本)

示例：
  v1.0.0    - 雁栖湖第一代发布版本
  v2.0.0    - 南湖第一代发布版本
  v2.1.0    - 南湖 V2 (增加 MBIST)
  v2.0.0    - 昆明湖 V2 首个正式版本 (本项目)
  v2.0.1    - 昆明湖 V2 修订版本
  v2.1.0-rc1 - 昆明湖 V2.1 候选版本
```

### 3.2 发布分支策略

```
main/master (稳定版本，对应 tags)
│
├── v1.0.0  (tag) - 雁栖湖 v1.0
│   └── RTL 源码、文档、测试用例
│
├── v2.0.0  (tag) - 南湖 v2.0
│   └── RTL 源码、设计文档、用户手册
│
├── v2.1.0  (tag) - 南湖 V2 (MBIST)
│   └── DFT 改进、P&R 优化
│
├── v2.0.0  (tag) - 昆明湖 V2 (本分支 kunminghu-v2)
│   ├── KunminghuV2Config (默认配置)
│   ├── MinimalConfig (最小可综合配置)
│   ├── FpgaConfig (FPGA 开发板配置)
│   └── XSNoCTopConfig (NoC 顶层配置)
│
├── v2.0.1  (tag) - 昆明湖 V2 修订版
│   └── Bug 修复、性能优化
│
├── v2.1.0  (分支) - 昆明湖 V2.1 功能增强
│   └── 新 ISA 扩展、工具链升级
│
├── develop  (开发分支，持续集成)
├── feature/*  (功能分支)
├── bugfix/*  (Bug 修复分支)
├── release/*  (发布准备分支)
├── hotfix/*  (热修复分支)
└── kunminghu-v2  (当前分支，昆明湖 V2)
```

### 3.3 配置发布矩阵

| 配置类 | 配置名 | 用途 | 适用场景 |
|--------|--------|------|---------|
| 默认 | `KunminghuV2Config` | 完整功能配置 | 量产、性能基准 |
| 最小 | `MinimalConfig` | 最小可综合配置 | 验证基础功能、快速仿真 |
| 仿真 | `MinimalSimConfig` | 仿真专用 (无 DCache, soft PTW) | 快速功能验证 |
| FPGA | `FpgaDefaultConfig` | FPGA 开发板配置 | 原型验证、板卡测试 |
| FPGA Difftest | `FpgaDiffDefaultConfig` | FPGA Difftest 配置 | 原型验证 + Difftest |
| NoC | `XSNoCTopConfig` | NoC 顶层配置 | 多核 SoC |
| 中等 | `MediumConfig` | 中等规模配置 (L3 4MB + L2 512KB) | 平衡性能与面积 |
| 内存加密 | `CVMConfig / CVMTestConfig` | 内存加密配置 | 安全计算场景 |
| 模糊测试 | `FuzzConfig` | 模糊测试配置 | 安全测试、覆盖率提升 |

---

## 4. 发布文档

### 4.1 发布说明 (Release Notes)

```markdown
# XiangShan Kunminghu V2 v2.0.0 Release Notes

## 发布信息
- 版本：v2.0.0
- 发布日期：2026-10-20
- 分支：kunminghu-v2
- 许可证：Mulan PSL v2
- 基础架构：香山第三代 (昆明湖)
- 代码仓库：https://github.com/hsq66/XiangShan/tree/kunminghu-v2

## 概述
昆明湖 V2 是香山开源高性能 RISC-V 处理器的第三代微架构重要版本，
采用 Chisel 6.7.0 (Scala 2.13.15) 编写，基于 rocket-chip Diplomacy 框架，
生成可综合的 SystemVerilog RTL，支持 6-wide 乱序执行、RVV 1.0 向量扩展、
H-extension 虚拟化、CHI Issue E.b 一致性协议。

## 新特性
### 架构特性
- 6-wide 超标量乱序执行流水线 (13+ 级)
- 取指 8 / 译码 6 / 重命名 6 / 发射 6 / 提交 8
- ROB 160 条目，RAB 256 条目
- 物理寄存器：224 (Int) + 192 (FP) + 128 (VF)

### 分支预测
- FTB (2048 entries, 4-way)
- TAGE-SC (4 tables, 4096 entries each)
- ITTAGE (5 tables)
- RAS (16 entries, Spec 32)
- UBTB (256 entries)
- 3 级预测流水线 (S1/S2/S3)

### 缓存与存储
- L1 I-Cache: 64KB, 4-way, SECDED, 2-cycle
- L1 D-Cache: 64KB, 4-way, SECDED, 2-cycle
- L2: 1MB, 8-way, Inclusive, 4-banks
- L3: 16MB, 16-way, Non-inclusive, 4-banks (可选)
- Load Queue: 72 entries, Store Queue: 56 entries
- Store Buffer: 16 entries
- TLB: L1 (48 entries × 5) + L2 (48 entries)

### 指令集支持
- RV64GCBVMV (基础 + 乘除法 + 原子 + 浮点 + 压缩 + 位操作 + 向量)
- H-extension (虚拟化)
- Sv39 / Sv48 (分页模式)
- NaN Boxing (浮点窄值处理)

### 一致性与互连
- CHI Issue E.b 一致性协议
- TileLink 片上总线 (rocket-chip Diplomacy)
- OpenLLC (Last Level Cache, OpenNCB)
- NoC 网络-on-Chip (多核 SMP)

### 调试与可测试性
- JTAG DTM (Debug Transport Module)
- Trace 接口 (RISC-V Trace 规范)
- MBIST (SRAM 自测试)
- SRAM_CTL (SRAM 控制)

## 已知问题
- 无 (首个正式发布版本)

## 升级指南
### 从南湖 V2 (v2.1.0) 迁移至昆明湖 V2 (v2.0.0)
1. 配置变更：KunminghuV2Config 替代 NanhuV2Config
2. CHI 协议：启用 EnableCHI，配置 OpenLLC
3. 向量支持：新增 RVV 1.0 支持 (VLEN=128)
4. 虚拟化：新增 H-extension, Sv48 支持
5. 构建工具：Mill 0.12.3 + Chisel 6.7.0

## 致谢
- 中国科学院计算技术研究所 (ICT-CAS)
- 北京开源芯片研究院 (BOSC)
- 鹏城实验室 (PCL)
- 香山开源社区贡献者

## 相关链接
- 香山官方文档：https://docs.xiangshan.cc
- Chisel 文档：https://www.chisel-lang.org
- RISC-V 规范：https://riscv.org
- rocket-chip 文档：https://github.com/chipsalliance/rocket-chip
- Mulan PSL v2：http://license.coscl.org.cn/MulanPSL2
```

### 4.2 用户指南 (User Guide)

#### 4.2.1 快速开始

```bash
# 1. 克隆仓库 (kunminghu-v2 分支)
git clone -b kunminghu-v2 https://github.com/hsq66/XiangShan.git
cd XiangShan

# 2. 初始化子模块 (rocket-chip, difftest, huancun, fudian, yunsuan 等)
make init

# 3. 生成 SystemVerilog RTL
make verilog CONFIG=KunminghuV2Config NUM_CORES=1
# 输出：build/rtl/XSTop.sv, build/rtl/SimTop.sv

# 4. 编译 Verilator 仿真器
make emu CONFIG=KunminghuV2Config

# 5. 运行 Difftest (与 NEMU 参考模型对比)
make emu CONFIG=KunminghuV2Config ENABLE_DIFFTEST=1

# 6. 运行特定 ISA 测试
./build/emu -b isa-tests/rv64mi-p-csr.bin

# 7. 运行 VCS 仿真 (商业仿真器)
make simv CONFIG=KunminghuV2Config
make simv-run

# 8. 查看代码覆盖率
make verilog FIRRTL_COVER=line,branch,expr,toggle
```

#### 4.2.2 构建参数说明

| 参数 | 默认值 | 可选值 | 说明 |
|------|--------|--------|------|
| CONFIG | DefaultConfig | KunminghuV2Config, MinimalConfig, FpgaDefaultConfig, ... | 配置类名 |
| NUM_CORES | 1 | 1, 2, 4, ... | 核心数量 (SMP) |
| ISSUE | E.b | B, C, E.b | CHI Issue 版本 |
| CHISEL_TARGET | systemverilog | systemverilog, firrtl | 目标语言 |
| JVM_XMX | 40G | 可调整 | JVM 堆内存 |
| JVM_XSS | 256m | 可调整 | JVM 栈内存 |
| RTL_SUFFIX | sv | sv | RTL 文件后缀 |

#### 4.2.3 常用 Makefile 目标

| 目标 | 说明 | 示例命令 |
|------|------|---------|
| verilog | 生成 SystemVerilog RTL | `make verilog CONFIG=KunminghuV2Config` |
| sim-verilog | 生成仿真专用 Verilog | `make sim-verilog` |
| emu | 编译并运行 Verilator 仿真 | `make emu CONFIG=KunminghuV2Config` |
| gsim | GSIM Difftest 仿真 | `make gsim CONFIG=KunminghuV2Config` |
| simv | 编译 VCS 仿真器 | `make simv CONFIG=KunminghuV2Config` |
| simv-run | 运行 VCS 仿真 | `make simv-run` |
| xsim | 编译 GalaxSim 仿真器 | `make xsim CONFIG=KunminghuV2Config` |
| pldm-build | Palladium 硬件仿真构建 | `make pldm-build CONFIG=KunminghuV2Config` |
| pldm-run | 运行 Palladium 仿真 | `make pldm-run` |
| comp | 编译检查 (不生成 RTL) | `make comp` |
| jar | 打包测试 JAR | `make jar` |
| clean | 清理构建目录 | `make clean` |
| init | 初始化 Git 子模块 | `make init` |
| init-force | 强制初始化子模块 | `make init-force` |
| check-format | 代码格式检查 (scalafmt) | `make check-format` |
| reformat | 代码格式化 | `make reformat` |
| bsp | 生成 BSP (Build Server Protocol) | `make bsp` |
| idea | 生成 IntelliJ IDEA 项目 | `make idea` |

#### 4.2.4 Docker 部署

```dockerfile
# Dockerfile (基于 ghcr.io/openxiangshan/xs-env:latest)
FROM ghcr.io/openxiangshan/xs-env:latest

# 设置工作目录
WORKDIR /xiangshan

# 复制源码
COPY . /xiangshan

# 初始化子模块
RUN make init

# 生成 RTL
RUN make verilog CONFIG=KunminghuV2Config

# 默认命令
CMD ["bash"]
```

```bash
# 构建 Docker 镜像
docker build -t xiangshan:kunminghu-v2-v2.0.0 .

# 运行容器
docker run -it --rm xiangshan:kunminghu-v2-v2.0.0

# 推送镜像 (需要权限)
docker push ghcr.io/hsq66/xiangshan:kunminghu-v2-v2.0.0
```

#### 4.2.5 FPGA 部署

```bash
# 1. 生成 FPGA 配置的 RTL
make verilog CONFIG=FpgaDefaultConfig

# 2. 综合 (以 Xilinx Vivado 为例)
vivado -mode tcl -script scripts/fpga_synth.tcl

# 3. 实现 (布局布线)
vivado -mode tcl -script scripts/fpga_impl.tcl

# 4. 生成比特流
vivado -mode tcl -script scripts/fpga_bitstream.tcl

# 5. 烧写 FPGA
xc3sprog -c ft2232 build/fpga.bit
```

### 4.3 硬件实现指南

#### 4.3.1 综合流程 (Synopsys Design Compiler)

```tcl
# 读取 RTL
read_verilog build/rtl/XSTop.sv
read_verilog build/rtl/SimTop.sv

# 设置约束
create_clock -name clk -period 0.5 [get_ports clock]  ;# 2GHz
set_input_delay 0.1 -clock clk [all_inputs]
set_output_delay 0.1 -clock clk [all_outputs]

# 设置库
set_link_path * [link_path]
set_target_library [list "typical.db"]

# 综合
compile_ultra

# 报告
report_timing > reports/timing.rpt
report_area > reports/area.rpt
report_power > reports/power.rpt
report_constraint -all_violators > reports/constraint.rpt
```

#### 4.3.2 物理设计流程 (Cadence Innovus/ ICC2)

| 步骤 | 工具 | 输出 | 关键检查点 |
|------|------|------|-----------|
| Floorplan | ICC2/Innovus | 布局规划 | 核心区域、IO 位置、宏单元布局 |
| Placement | ICC2/Innovus | 标准单元布局 | 时序滞后、拥塞度 |
| CTS | ICC2/Innovus | 时钟树综合 | 时钟 skew、插入延迟 |
| Routing | ICC2/Innovus | 布线 | 违规、拥塞、时序 |
| Signoff | Calibre/ICC2 | 签核验证 | DRC、LVS、时序、功耗 |

#### 4.3.3 工艺节点选择

| 工艺 | 应用场景 | 主频目标 | 面积目标 | 功耗目标 |
|------|---------|---------|---------|---------|
| 28nm | 验证节点、低成本量产 | 2.0 GHz | ≤ 3mm² | ≤ 1W |
| 14nm | 高性能量产 | 2.5 GHz | ≤ 2mm² | ≤ 0.8W |
| 7nm | 顶级性能 | 3.0 GHz | ≤ 1.5mm² | ≤ 0.6W |

---

## 5. 发布流程

### 5.1 发布流程图

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        发布流程 (Release Process)                        │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────┐ │
│  │ 代码冻结 │──▶│ 最终验证 │──▶│ 构建发布 │──▶│ 文档完成 │──▶│ 发布 │ │
│  │ Code     │   │ Final    │   │ Build    │   │ Docs     │   │ Eval │ │
│  │ Freeze   │   │ Verify   │   │ Release  │   │ Complete │   │      │ │
│  └──────────┘   └──────────┘   └──────────┘   └──────────┘   └──────┘ │
│       │              │              │              │              │   │
│       ▼              ▼              ▼              ▼              ▼   │
│  ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────┐ │
│  │ 停止合并 │   │ 全量回归 │   │ Verilog  │   │ Release  │   │ GitHub │ │
│  │ 新功能   │   │ 测试     │   │ 文档     │   │ Notes    │   │ Release│ │
│  │          │   │          │   │ Docker   │   │ 用户指南 │   │ 页面   │ │
│  │ T-7 天   │   │ T-5 天   │   │ T-3 天   │   │ T-3 天   │   │ T      │ │
│  └──────────┘   └──────────┘   └──────────┘   └──────────┘   └──────┘ │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 5.2 发布步骤与时间表

| 步骤 | 任务 | 负责人 | 时间节点 | 交付物 |
|------|------|--------|---------|--------|
| 1 | 代码冻结 | 项目经理 | T-7 天 | 无新功能合并 |
| 2 | 最终验证 | 验证组 | T-5 天 | 全量回归通过 |
| 3 | 构建发布版本 | 工具组 | T-3 天 | Verilog、Docker 镜像、JAR |
| 4 | 生成文档 | 文档组 | T-3 天 | Release Notes、用户指南 |
| 5 | 发布评审 | 项目组 | T-1 天 | RDCP 评审通过 |
| 6 | 正式发布 | 项目经理 | T | GitHub Release、文档发布 |

### 5.3 发布物清单

| 发布物 | 格式 | 位置 | 说明 |
|--------|------|------|------|
| RTL 源码 | SystemVerilog | `build/rtl/` | 可综合 RTL (XSTop.sv, SimTop.sv) |
| 构建脚本 | Makefile | 根目录 | 构建、仿真、综合入口 |
| 配置文件 | YAML/Scala | `src/main/resources/config/`, `src/main/scala/top/` | SoC 配置、微架构参数 |
| 测试用例 | 二进制 | `isa-tests/`, `src/test/scala/` | ISA 测试、单元测试 |
| Docker 镜像 | Docker | `ghcr.io/hsq66/xiangshan` | 开发环境镜像 (xsdev) |
| 用户指南 | Markdown/PDF | `IPD-Docs/`, `docs/` | 安装、使用、配置指南 |
| 发布说明 | Markdown | 仓库 Release 页面 | 新特性、已知问题、升级指南 |
| 许可证 | 文本 | `LICENSE` | Mulan PSL v2 |

---

## 6. 交付与部署

### 6.1 交付物目录结构

```
release/v2.0.0/
├── rtl/
│   ├── XSTop.sv
│   ├── SimTop.sv
│   └── *.conf (SRAM 配置)
├── src/
│   ├── main/scala/          # Chisel 源码
│   └── resources/config/    # 配置文件
├── docs/
│   ├── Release-Notes-v2.0.0.md
│   ├── User-Guide.md
│   └── Upgrade-Guide.md
├── tests/
│   ├── isa-tests/           # ISA 测试套件
│   └── unit-tests/          # 单元测试
├── docker/
│   └── Dockerfile           # 开发环境镜像
├── scripts/
│   ├── build.sh             # 构建脚本
│   └── sim.sh               # 仿真脚本
└── LICENSE                  # Mulan PSL v2
```

### 6.2 GitHub Release 发布

```bash
# 创建 Release 标签
git tag -a v2.0.0 -m "XiangShan Kunminghu V2 v2.0.0 Release"

# 推送标签
git push origin v2.0.0

# 在 GitHub 网页创建 Release，上传：
# - RTL 源码包 (tar.gz)
# - 文档包 (tar.gz)
# - Docker 镜像 (如适用)
```

### 6.3 社区支持渠道

| 渠道 | 用途 | 链接 |
|------|------|------|
| GitHub Issues | Bug 报告、功能请求 | https://github.com/hsq66/XiangShan/issues |
| GitHub Discussions | 社区讨论 | https://github.com/hsq66/XiangShan/discussions |
| 文档站 | 产品文档 | https://docs.xiangshan.cc |
| 邮件列表 | 项目通知 | xiangshan-dev@googlegroups.com |
| 微信/社群 | 中文社区交流 | 香山开源社区 |

---

## 7. 发布后支持

### 7.1 支持策略

| 版本类型 | 支持周期 | 更新内容 | 示例 |
|----------|---------|---------|------|
| 稳定版本 (Stable) | 长期支持 | Bug 修复、安全补丁 | v2.0.x |
| 开发版本 (Development) | 短期支持 | 新特性、性能优化 | develop 分支 |
| EOL 版本 | 停止支持 | 无更新 | v1.x |

### 7.2 版本支持矩阵

| 版本 | 状态 | 支持截止 | 备注 |
|------|------|---------|------|
| v2.0.x | Active (活跃) | 2027-10 | 当前版本，昆明湖 V2 |
| v2.1.x | Planning (规划中) | 2028-04 | 新特性开发中 |
| v1.x | Maintenance (维护) | 2026-12 | 南湖架构，仅安全补丁 |
| v0.x | EOL (停止支持) | - | 雁栖湖架构，已停止 |

### 7.3 安全响应流程

| 严重程度 | 响应时间 | 修复时间 | 通知方式 |
|----------|---------|---------|---------|
| Critical | 24 小时 | 72 小时 | 邮件 + GitHub 公告 |
| High | 48 小时 | 1 周 | 邮件 + 仓库公告 |
| Medium | 1 周 | 2 周 | 仓库公告 |
| Low | 2 周 | 1 月 | 版本说明中披露 |

**安全更新流程：**
1. 安全漏洞报告 (GitHub Issue 或邮件)
2. 漏洞验证与风险评估
3. 修复方案制定
4. 修复代码开发 + 测试
5. 发布安全公告 (CVE 如适用)
6. 推送修复补丁 (patch release)

---

## 8. 发布阶段决策评审 (RDCP)

### 8.1 评审检查清单

- [x] 代码冻结 (无新功能合并)
- [x] 最终验证通过 (单元/集成/系统/性能)
- [x] 发布版本构建成功 (Verilog、Docker、JAR)
- [x] 文档完成 (Release Notes、用户指南、升级指南)
- [x] 已知问题列表完成
- [x] 发布评审通过 (RDCP)
- [x] 交付物准备就绪 (RTL、文档、测试用例)
- [x] 支持渠道就绪 (Issues、Discussions、文档站)

### 8.2 评审结论

**通过** — 项目正式发布。

---

## 9. 附录

### 9.1 发布检查表

```
发布检查表 (Release Checklist)
─────────────────────────────────────────────────────────────────────────
□ 代码冻结完成
□ 单元测试通过 (>90% 覆盖率)
□ 集成测试通过 (Difftest 100%)
□ 系统测试通过 (Linux 启动)
□ 性能测试通过 (不低于基线)
□ 回归测试通过 (100%)
□ 代码格式检查通过 (scalafmt)
□ 代码风格检查通过 (scalastyle)
□ Verilog 生成成功
□ 仿真器构建成功 (Verilator/VCS)
□ Docker 镜像构建成功
□ 发布说明 (Release Notes) 完成
□ 用户指南 (User Guide) 完成
□ 升级指南 (Upgrade Guide) 完成
□ 已知问题列表完成
□ 贡献者名单更新
□ 许可证文件 (Mulan PSL v2) 就绪
□ GitHub Release 页面创建
□ 支持渠道就绪 (Issues/Discussions/文档)
─────────────────────────────────────────────────────────────────────────
```

### 9.2 联系人信息

| 角色 | 联系方式 | 职责 |
|------|---------|------|
| 项目总监 | director@xiangshan.cc | 整体规划、决策、资源协调 |
| 技术支持 | support@xiangshan.cc | 用户问题、安装配置 |
| 安全响应 | security@xiangshan.cc | 漏洞报告、安全补丁 |
| 文档维护 | docs@xiangshan.cc | 文档更新、翻译 |

### 9.3 相关资源

| 资源 | 链接 | 说明 |
|------|------|------|
| 香山官方文档 | https://docs.xiangshan.cc | 三代架构演进、敏捷开发理念 |
| Chisel 文档 | https://www.chisel-lang.org | Chisel 6.7.0 语言参考、API |
| RISC-V 规范 | https://riscv.org/technical/specifications/ | RV64GCBVMV, H-extension, Sv39/Sv48 |
| rocket-chip 文档 | https://github.com/chipsalliance/rocket-chip | Diplomacy, TileLink, AXI4, LazyModule |
| 香山 GitHub | https://github.com/hsq66/XiangShan/tree/kunminghu-v2 | 本项目源码仓库 |
| Mulan PSL v2 | http://license.coscl.org.cn/MulanPSL2 | 开源许可证 |

---

*文档版本：v1.1（深度优化版）*
*最后更新：2026-10-20*
*基于：docs.xiangshan.cc、Chisel 6.7.0、RISC-V 规范、rocket-chip Diplomacy、本地代码分析*
