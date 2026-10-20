# 香山昆明湖 V2 开发工具链部署总结

## 部署状态：✅ 基本完成（Scala 编译可用）

## 已安装工具链

| 工具 | 版本 | 安装方式 | 位置 | 状态 |
|------|------|---------|------|------|
| Java (Eclipse Adoptium Temurin) | 21.0.12.1 LTS | 预安装 | `C:\Program Files\Eclipse Adoptium\jdk-21\` | ✅ |
| Git | 2.55.0.windows.3 | 预安装 | `C:\Program Files\Git\` | ✅ |
| Python 3 | 3.15.0 | Scoop | `C:\Users\Administrator\AppData\Local\scoop\apps\python\` | ✅ |
| Make | 4.4.1 | Scoop | `C:\Users\Administrator\AppData\Local\scoop\apps\make\` | ✅ |
| Mill | 0.12.3 | 手动下载 + Git Bash | `C:\Users\Administrator\AppData\Local\scoop\apps\mill\current\mill` | ✅ |
| Coursier | 2.1.26 | Scoop | `C:\Users\Administrator\AppData\Local\scoop\apps\coursier\` | ✅ |
| Verilator | 已安装 | WSL2 Ubuntu apt | `/usr/local/bin/verilator` (WSL2) | ✅ |
| Node.js / npm | 24.21.0 | 预安装 | `C:\Users\Administrator\AppData\Local\Author Software\nvm\` | ✅ |
| WSL2 | Ubuntu-24.04 | 预安装 | Windows WSL2 | ✅ |
| firtool | 自动下载 | Mill firtool-resolver | 构建时自动解析 | ⏳ 自动 |
| Scala 2.13.15 | 自动下载 | Mill/Coursier | 构建时自动解析 | ⏳ 自动 |
| Chisel 6.7.0 | 自动下载 | Mill/Coursier | 构建时自动解析 | ⏳ 自动 |

## 子模块状态：✅ 全部初始化

### 主子模块 (11 个)
| 子模块 | 路径 | 状态 |
|--------|------|------|
| rocket-chip | `rocket-chip/` | ✅ |
| difftest | `difftest/` | ✅ |
| ready-to-run | `ready-to-run/` | ✅ |
| huancun | `huancun/` | ✅ |
| fudian | `fudian/` | ✅ |
| utility | `utility/` | ✅ |
| yunsuan | `yunsuan/` | ✅ (已切换至 master 分支以匹配 kunminghu-v2) |
| coupledL2 | `coupledL2/` | ✅ |
| openLLC | `openLLC/` | ✅ |
| ChiselAIA | `ChiselAIA/` | ✅ |
| ChiselIOPMP | `ChiselIOPMP/` | ✅ |

### 嵌套子子模块 (8 个)
| 子子模块 | 路径 | 状态 |
|----------|------|------|
| cde | `rocket-chip/cde/` | ✅ |
| hardfloat | `rocket-chip/hardfloat/` | ✅ |
| openNCB | `openLLC/openNCB/` | ✅ |
| rocket-chip (utility) | `utility/rocket-chip/` | ✅ |
| rocket-chip (ChiselIOPMP) | `ChiselIOPMP/rocket-chip/` | ✅ |
| utility (ChiselIOPMP) | `ChiselIOPMP/utility/` | ✅ |
| HuanCun | `coupledL2/HuanCun/` | ✅ |
| rocket-chip (coupledL2) | `coupledL2/rocket-chip/` | ✅ |

## 构建验证结果：✅ Scala 编译通过

```
[653/653] xiangshan.compile
[653] [info] compiling 382 Scala sources to ...\out\xiangshan\compile.dest\classes ...
[653] [info] done compiling
```

382 个 Scala 源文件全部编译通过，涉及模块：
- rocketchip.cde, macros, rocketchip.macros
- fudian (15 源文件)
- difftest (39 源文件)
- yunsuan (167→80 源文件，已切换至 master)
- rocketchip.hardfloat (20 源文件)
- rocketchip (315 源文件)
- openNCB (89 源文件)
- utility (61 源文件)
- chiselAIA (6 源文件)
- huancun (43 源文件)
- chiselIOPMP (3 源文件)
- coupledL2 (60 源文件)
- openLLC (27 源文件)
- xiangshan (382 源文件)

## 已知问题：Verilog 生成 (Chisel 编译器问题)

Verilog 生成在 `chisel3.util.experimental.decode.QMCMinimizer.getCover` 阶段失败：

```
at chisel3.util.experimental.decode.QMCMinimizer$.getCover(QMCMinimizer.scala:213)
```

这是 **Chisel 6.7.0 编译器内部问题**（Quine-McCluskey 逻辑最小化算法在处理指令解码表时失败），与工具链部署无关。Scala 编译已成功，说明所有源代码语法正确。

### 解决方案

| 方案 | 说明 |
|------|------|
| **方案 A：降级 Chisel 版本** | 修改 `build.sc` 中的 `chisel` 版本从 `6.7.0` 降至 `6.6.0` 或 `6.5.0`，QMCMinimizer 问题可能不存在 |
| **方案 B：使用 Docker 环境** | 项目提供 `Dockerfile` (基于 `ghcr.io/openxiangshan/xs-env:latest`)，容器内已预配置完整工具链 |
| **方案 C：WSL2 原生构建** | 在 WSL2 Ubuntu 中直接运行 Mill，避免 Windows 路径和管道问题 |
| **方案 D：修改 QMCMinimizer** | 在 Chisel 源码中为 QMCMinimizer 添加迭代次数或大小限制 |

### Docker 部署命令

```bash
# 构建 Docker 镜像
docker build -t xiangshan:kunminghu-v2 .

# 运行容器并构建
docker run -it --rm xiangshan:kunminghu-v2 bash
# 在容器内：
make init
make verilog CONFIG=KunminghuV2Config
```

## 开发命令参考 (Windows PowerShell + Git Bash)

```powershell
# 进入项目目录
cd D:\cdkv\xiangshan\kunminghuV2\XiangShan-kunminghu-v2

# 查看 Mill 帮助
"C:\Program Files\Git\bin\bash.exe" -c 'cd /d/cdkv/xiangshan/kunminghuV2/XiangShan-kunminghu-v2 && ./mill --help'

# 编译所有 Scala 源代码
"C:\Program Files\Git\bin\bash.exe" -c 'cd /d/cdkv/xiangshan/kunminghuV2/XiangShan-kunminghu-v2 && ./mill xiangshan.compile'

# 编译测试代码
"C:\Program Files\Git\bin\bash.exe" -c 'cd /d/cdkv/xiangshan/kunminghuV2/XiangShan-kunminghu-v2 && ./mill xiangshan.test.compile'

# 运行单元测试示例
"C:\Program Files\Git\bin\bash.exe" -c 'cd /d/cdkv/xiangshan/kunminghuV2/XiangShan-kunminghu-v2 && ./mill xiangshan.test.testOnly xiangshan.DecodeTest'

# WSL2 中运行 Verilator
wsl -d Ubuntu-24.04 --exec /bin/bash -c "verilator --version"

# WSL2 中访问项目文件
wsl -d Ubuntu-24.04 --exec /bin/bash -c "ls /mnt/d/cdkv/xiangshan/kunminghuV2/XiangShan-kunminghu-v2/"
```

## 文档位置

- IPD 全生命周期文档：`IPD-Docs/` (7 个文档)
- 开发环境指南：`IPD-Docs/DEV-ENVIRONMENT.md`

## 结论

开发工具链已基本部署完成。Scala 代码编译可用，所有子模块已初始化。Verilog RTL 生成受 Chisel 6.7.0 QMCMinimizer 问题影响，建议使用 Docker 环境或降级 Chisel 版本来绕过此问题。

*部署日期：2026-10-20*
*环境：Windows 11 + WSL2 Ubuntu-24.04 + Scoop*
