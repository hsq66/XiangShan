# 香山昆明湖 V2 工具使用与编译指导说明书

## 文档概述

本文档提供香山昆明湖 V2 项目的完整工具使用指南和编译流程说明，包括 Mill 构建命令、Verilog RTL 生成、仿真运行、测试执行等所有开发工作流。

---

## 1. 核心构建命令 (Mill)

Mill 是香山项目的主要构建系统，所有命令都在项目根目录的 Git Bash 中执行。

### 1.1 常用 Mill 命令速查表

| 命令 | 说明 | 示例 |
|------|------|------|
| `./mill --version` | 查看 Mill 版本 | `./mill --version` |
| `./mill __.printLogo` | 显示 Mill Logo | `./mill __.printLogo` |
| `./mill xiangshan.compile` | 编译主源代码 | `./mill xiangshan.compile` |
| `./mill xiangshan.test.compile` | 编译测试代码 | `./mill xiangshan.test.compile` |
| `./mill xiangshan.runMain top.TopMain` | 运行顶层生成器 | `./mill xiangshan.runMain top.TopMain --help` |
| `./mill xiangshan.test.testOnly` | 运行特定测试 | `./mill xiangshan.test.testOnly xiangshan.DecodeTest` |
| `./mill __.clean` | 清理构建输出 | `./mill __.clean` |

### 1.2 生成 Verilog RTL

生成可综合的 SystemVerilog 文件：

```bash
# 基本命令格式
./mill xiangshan.runMain top.TopMain \
  --target-dir <输出目录> \
  --config <配置类名> \
  --issue <CHI Issue 版本> \
  --num-cores <核心数量> \
  --firtool-opt "<firtool 优化选项>" \
  --split-verilog --dump-fir

# 示例：生成 KunminghuV2Config 配置的 RTL
./mill -Djvm-xmx=32G -Djvm-xss=256m xiangshan.runMain top.TopMain \
  --target-dir build/rtl \
  --config KunminghuV2Config \
  --issue E.b \
  --num-cores 1 \
  --firtool-opt "-O=release --disable-annotation-unknown --lowering-options=explicitBitcast,disallowLocalVariables,disallowPortDeclSharing,locationInfoStyle=none" \
  --split-verilog --dump-fir
```

### 1.3 生成仿真专用 RTL

```bash
./mill xiangshan.test.runMain top.XiangShanSim \
  --target-dir build/rtl \
  --config KunminghuV2Config \
  --issue E.b \
  --num-cores 1
```

---

## 2. Makefile 构建目标

项目根目录的 Makefile 提供了便捷的构建入口。

### 2.1 主要构建目标

| 目标 | 说明 | 命令示例 |
|------|------|---------|
| `verilog` | 生成 SystemVerilog RTL | `make verilog CONFIG=KunminghuV2Config` |
| `sim-verilog` | 生成仿真专用 Verilog | `make sim-verilog CONFIG=KunminghuV2Config` |
| `emu` | 编译并运行 Verilator 仿真 | `make emu CONFIG=KunminghuV2Config` |
| `gsim` | GSIM Difftest 仿真 | `make gsim CONFIG=KunminghuV2Config` |
| `simv` | 编译 VCS 仿真器 | `make simv CONFIG=KunminghuV2Config` |
| `simv-run` | 运行 VCS 仿真 | `make simv-run CONFIG=KunminghuV2Config` |
| `xsim` | 编译 GalaxSim 仿真器 | `make xsim CONFIG=KunminghuV2Config` |
| `pldm-build` | Palladium 硬件仿真构建 | `make pldm-build CONFIG=KunminghuV2Config` |
| `comp` | 仅编译检查（不生成 RTL） | `make comp` |
| `clean` | 清理构建目录 | `make clean` |
| `init` | 初始化 Git 子模块 | `make init` |
| `check-format` | 代码格式检查 | `make check-format` |
| `reformat` | 代码格式化 | `make reformat` |

### 2.2 Makefile 环境变量

| 变量 | 默认值 | 说明 |
|------|--------|------|
| `CONFIG` | `DefaultConfig` | Chisel 配置类名 |
| `NUM_CORES` | `1` | 处理器核心数量 |
| `ISSUE` | `E.b` | CHI Issue 版本 (B/C/E.b) |
| `CHISEL_TARGET` | `systemverilog` | 目标语言 |
| `JVM_XMX` | `40G` | JVM 堆内存 |
| `JVM_XSS` | `256m` | JVM 栈内存 |
| `RTL_SUFFIX` | `sv` | RTL 文件后缀 |

### 2.3 使用示例

```bash
# 生成 RTL (默认配置)
make verilog CONFIG=KunminghuV2Config NUM_CORES=1

# 生成最小配置 RTL (快速测试)
make verilog CONFIG=MinimalConfig

# 生成 FPGA 配置 RTL
make verilog CONFIG=FpgaDefaultConfig

# 编译检查
make comp

# 运行 Verilator 仿真
make emu CONFIG=KunminghuV2Config
```

---

## 3. 配置系统说明

### 3.1 主要配置类

| 配置类 | 说明 | 适用场景 |
|--------|------|---------|
| `KunminghuV2Config` | 昆明湖 V2 完整配置 (CHI 一致性) | 量产、性能基准 |
| `MinimalConfig` | 最小可综合配置 (无 L2) | 验证基础功能、快速仿真 |
| `MinimalSimConfig` | 仿真专用配置 (无 DCache, soft PTW) | 快速功能验证 |
| `DefaultConfig` | 默认配置 (L3 16MB + L2 1MB) | 通用配置 |
| `XSNoCTopConfig` | NoC 顶层配置 | 多核 SoC |
| `FpgaDefaultConfig` | FPGA 开发板配置 | 原型验证 |
| `MediumConfig` | 中等规模 (L3 4MB + L2 512KB) | 性能与面积平衡 |
| `CVMConfig` | 内存加密配置 | 安全计算场景 |

### 3.2 配置参数示例 (KunminghuV2Config)

```scala
class KunminghuV2Config(n: Int = 1) extends Config(
  L2CacheConfig("1MB", inclusive = true, banks = 4, tp = false)
    ++ new DefaultConfig(n)
    ++ new WithCHI
)
```

关键参数：
- `FetchWidth = 8`, `DecodeWidth = 6`, `RenameWidth = 6`, `CommitWidth = 8`
- `RobSize = 160`, `RabSize = 256`, `FtqSize = 64`
- `intPreg.numEntries = 224`, `fpPreg.numEntries = 192`, `vfPreg.numEntries = 128`
- `VLEN = 128`, `ELEN = 64` (RVV 1.0)
- L1 I/D Cache: 64KB, 4-way, SECDED
- L2: 1MB, 8-way, Inclusive, 4-banks
- L3: 16MB, 16-way, Non-inclusive, 4-banks

---

## 4. 仿真运行

### 4.1 Verilator 仿真 (开源，推荐)

```bash
# 1. 生成仿真 RTL
make sim-verilog CONFIG=KunminghuV2Config

# 2. 编译 Verilator 仿真器
make emu-mk CONFIG=KunminghuV2Config

# 3. 运行仿真
make emu CONFIG=KunminghuV2Config

# 4. 运行 Difftest (与 NEMU 参考模型对比)
make emu CONFIG=KunminghuV2Config ENABLE_DIFFTEST=1
```

### 4.2 运行特定测试用例

```bash
# 运行 ISA 测试
./build/emu -b isa-tests/rv64mi-p-csr.bin

# 运行 CoreMark
./build/emu -b ready-to-run/coremark-2-iteration.bin

# 运行 Linux
./build/emu -b ready-to-run/linux.bin
```

### 4.3 VCS 仿真 (商业，高性能)

```bash
make simv CONFIG=KunminghuV2Config
make simv-run CONFIG=KunminghuV2Config
```

### 4.4 GalaxSim 仿真

```bash
make xsim CONFIG=KunminghuV2Config
make xsim-run CONFIG=KunminghuV2Config
```

---

## 5. 测试执行

### 5.1 单元测试

```bash
# 运行特定测试类
./mill xiangshan.test.testOnly xiangshan.DecodeTest

# 运行所有测试
make test

# 运行特定模块测试
make test-DecodeUnit
make verilog-decode
```

### 5.2 ISA 测试套件

| 测试目录 | 说明 |
|----------|------|
| `isa-tests/rv64mi/` | 机器模式指令测试 |
| `isa-tests/rv64si/` | 监管者模式指令测试 |
| `isa-tests/rv64ua/` | 原子指令测试 |
| `isa-tests/rv64uf/` | 单精度浮点测试 |
| `isa-tests/rv64ud/` | 双精度浮点测试 |
| `isa-tests/rv64uv/` | 向量扩展测试 (RVV 1.0) |

---

## 6. 代码质量检查

```bash
# 代码格式检查 (scalafmt)
make check-format

# 代码格式化
make reformat

# 风格检查 (scalastyle)
mill xiangshan.checkFormat
```

---

## 7. Docker 环境使用 (推荐)

如果本机工具链有问题，建议使用 Docker 容器：

```bash
# 构建 Docker 镜像
docker build -t xiangshan:kunminghu-v2 -f Dockerfile .

# 运行容器
docker run -it --rm \
  -v $(pwd):/work \
  xiangshan:kunminghu-v2

# 在容器内构建
make init
make verilog CONFIG=KunminghuV2Config
```

---

## 8. 输出文件说明

| 输出路径 | 说明 |
|----------|------|
| `build/rtl/XSTop.sv` | 顶层 SystemVerilog RTL |
| `build/rtl/SimTop.sv` | 仿真专用 SystemVerilog RTL |
| `build/rtl/*.conf` | SRAM 配置文件 |
| `out/` | Mill 编译输出目录 |
| `build/emu` | Verilator 仿真器二进制 |
| `build/simv` | VCS 仿真器二进制 |

---

## 9. 故障排除

| 问题 | 原因 | 解决方案 |
|------|------|---------|
| Mill 编译缓慢 | 首次下载依赖 | 耐心等待，后续增量编译很快 |
| Scala 编译错误 | 子模块分支不匹配 | 检查 yunsuan 是否在 master 分支 |
| Verilog 生成失败 | Chisel QMCMinimizer 算法爆炸 | 使用 Docker 环境或优化 JVM 内存 |
| 仿真器找不到 | 未运行 `make emu-mk` | 先运行 `make sim-verilog` 再运行 `make emu-mk` |
| Difftest 失败 | NEMU 参考模型不匹配 | 检查 difftest 子模块是否初始化 |

---

*文档生成日期：2026-10-20*
*环境：Windows 11 + WSL2 Ubuntu-24.04*
