# 香山昆明湖 V2 开发工具链部署全过程文档

## 文档概述

本文档详细记录了在 Windows 11 + WSL2 环境下，为香山昆明湖 V2 (XiangShan Kunminghu V2) 项目部署完整开发工具链的全过程，包括所有工具的下载、安装、配置和验证步骤。

## 1. 环境前提

### 1.1 本机环境概览

| 项目 | 详情 |
|------|------|
| 操作系统 | Windows 11 |
| 架构 | x86_64 (amd64) |
| Shell | PowerShell 5.1 + Git Bash (MINGW64) |
| 容器平台 | Docker 29.8.0 + WSL2 (Ubuntu-24.04) |
| 网络 | 可访问 GitHub (github.com:443 连通) |

### 1.2 已预安装工具

| 工具 | 版本 | 安装位置 |
|------|------|---------|
| Java (Eclipse Adoptium Temurin) | 21.0.12.1 LTS | `C:\Program Files\Eclipse Adoptium\jdk-21\` |
| Git | 2.55.0.windows.3 | `C:\Program Files\Git\` |
| Node.js / npm | 24.21.0 | `C:\Users\Administrator\AppData\Local\Author Software\nvm\` |

---

## 2. 工具链安装步骤

### 2.1 安装 Scoop (Windows 包管理器)

Scoop 是 Windows 上轻量级的命令行包管理器，用于安装 Python、Make、Coursier 等开发工具。

```powershell
# 1. 设置执行策略
Set-ExecutionPolicy RemoteSigned -Scope CurrentUser -Force

# 2. 下载 Scoop 安装脚本
Invoke-WebRequest -Uri "https://get.scoop.sh" -OutFile "$env:TEMP\scoop-install.ps1" -TimeoutSec 30

# 3. 运行安装脚本（非管理员模式）
Set-ExecutionPolicy RemoteSigned -Scope CurrentUser -Force
& "$env:TEMP\scoop-install.ps1" -ScoopDir "$env:LOCALAPPDATA\scoop"

# 4. 添加 extras bucket（部分工具在 extras 中）
$env:SCOOP = "$env:LOCALAPPDATA\scoop"
$env:PATH = "$env:SCOOP\bin;$env:PATH"
& "$env:SCOOP\bin\scoop.ps1" bucket add extras

# 5. 验证安装
& "$env:SCOOP\bin\scoop.ps1" --version
```

### 2.2 安装基础构建工具

```powershell
# 安装 Python 3 和 Make（Scoop）
& "$env:LOCALAPPDATA\scoop\bin\scoop.ps1" install python make

# 安装 Coursier（Scala 包管理器）
& "$env:LOCALAPPDATA\scoop\bin\scoop.ps1" install coursier

# 验证安装
python3 --version
make --version
coursier --version
```

**预期输出：**
```
Python 3.15.0
GNU Make 4.4.1
coursier 2.1.26
```

### 2.3 安装 Mill 构建工具 (0.12.3)

Mill 是香山项目的构建系统，版本由 `.mill-version` 文件锁定为 `0.12.3`。

```powershell
# 1. 创建 Mill 安装目录
New-Item -ItemType Directory -Path "$env:LOCALAPPDATA\scoop\apps\mill\current" -Force

# 2. 下载 Mill 启动脚本
Invoke-WebRequest -Uri "https://github.com/lihaoyi/mill/releases/download/0.12.3/0.12.3" -OutFile "$env:LOCALAPPDATA\scoop\apps\mill\current\mill" -TimeoutSec 120

# 3. 创建 Mill PowerShell 包装器
@'
param([string[]]$Args)
java -jar "$env:LOCALAPPDATA\scoop\apps\mill\current\mill" @Args
'@ | Out-File -FilePath "$env:LOCALAPPDATA\scoop\apps\mill\current\mill.ps1" -Encoding utf8

# 4. 创建 Scoop shim（使 mill 命令可用）
@'
param([string[]]$Args)
& "$env:LOCALAPPDATA\scoop\apps\mill\current\mill.ps1" @Args
'@ | Out-File -FilePath "$env:SCOOP\shims\mill.ps1" -Encoding utf8

# 5. 将 Mill 添加到 PATH
$env:PATH = "$env:LOCALAPPDATA\scoop\bin;$env:PATH"

# 6. 验证安装（需要在 Git Bash 中运行，因为 Mill 是 shell 脚本）
"C:\Program Files\Git\bin\bash.exe" --version
```

> **注意**：Mill 是 shell 脚本，必须在 Git Bash 中运行。无法直接在 PowerShell 中执行。

### 2.4 将 Mill 部署到项目目录

```powershell
# 将 mill 脚本复制到项目根目录
Copy-Item "$env:LOCALAPPDATA\scoop\apps\mill\current\mill" "D:\cdkv\xiangshan\kunminghuV2\XiangShan-kunminghu-v2\mill" -Force

# 验证 Mill 版本
& "C:\Program Files\Git\bin\bash.exe" -c 'cd /d/cdkv/xiangshan/kunminghuV2/XiangShan-kunminghu-v2 && ./mill --version'
```

**预期输出（首次运行会下载 Mill 核心，耗时 1-2 分钟）：**
```
Downloading https://github.com/lihaoyi/mill/releases/download/0.12.3/...
...
Mill Build Tool version 0.12.3
Java version: 21.0.12.1, vendor: Eclipse Adoptium, runtime: C:\Program Files\Eclipse Adoptium\jdk-21.0.12.101-hotspot
Default locale: zh_CN, platform encoding: UTF-8
OS name: "Windows 11", version: 10.0, arch: amd64
```

---

## 3. Git 子模块初始化

香山项目依赖 11 个 Git 子模块，部分子模块还有嵌套子模块。

### 3.1 初始化所有子模块

```powershell
cd "D:\cdkv\xiangshan\kunminghuV2\XiangShan-kunminghu-v2"

# 初始化主子模块
git submodule update --init

# 初始化嵌套子子模块
cd rocket-chip && git submodule update --init cde hardfloat && cd ..
cd openLLC && git submodule update --init openNCB && cd ..
cd utility && git submodule update --init rocket-chip && cd ..
cd ChiselIOPMP && git submodule update --init rocket-chip utility && cd ..
cd coupledL2 && git submodule update --init HuanCun rocket-chip && cd ..
```

### 3.2 修复 yunsuan 子模块分支

yunsuan 子模块默认在 `kunminghu-v3` 分支，与主项目 `kunminghu-v2` 分支的 API 不兼容，需要切换到 `master`：

```powershell
cd "D:\cdkv\xiangshan\kunminghuV2\XiangShan-kunminghu-v2\yunsuan"
git checkout master
```

### 3.3 验证子模块状态

```powershell
git -C "D:\cdkv\xiangshan\kunminghuV2\XiangShan-kunminghu-v2" submodule status
```

**预期输出（所有子模块都有哈希值，未以 `-` 开头）：**
```
 13fddf2f ChiselAIA (native_axi-61-g13fddf2)
 cc1113c3 ChiselIOPMP (heads/master)
 55d8e69c coupledL2 (v0.9-325-g55d8e69c)
 268b88c6 difftest (heads/master)
 e1bd4695 fudian (heads/pipeline)
 65ef0773 huancun (heads/master)
 a62384a2 openLLC (heads/master)
 c4114ce3 ready-to-run (nemu-ci-workloads-20250913-16-gc4114ce)
 9b59a4b0 rocket-chip (heads/master)
 b3afd290 utility (heads/master)
 267ff736 yunsuan (heads/kunminghu-v3 -> 已切换至 master)
```

---

## 4. 工具链验证

### 4.1 Scala 编译测试

```powershell
& "C:\Program Files\Git\bin\bash.exe" -c 'cd /d/cdkv/xiangshan/kunminghuV2/XiangShan-kunminghu-v2 && ./mill xiangshan.compile'
```

**预期结果：**
- 382 个 Scala 源文件全部编译通过
- 涉及模块：rocketchip, fudian, difftest, yunsuan, hardfloat, openNCB, utility, chiselAIA, huancun, chiselIOPMP, coupledL2, openLLC, xiangshan
- 可能有 deprecation 警告，但不影响功能

### 4.2 Verilator 验证 (WSL2)

```bash
wsl -d Ubuntu-24.04 --exec /bin/bash -c "verilator --version"
```

**预期输出：**
```
Verilator 5.x.x
```

---

## 5. 工具链总览表

| 工具 | 版本 | 安装方式 | 位置 | 用途 |
|------|------|---------|------|------|
| Java | 21.0.12.1 LTS | 预安装 | `C:\Program Files\Eclipse Adoptium\jdk-21\` | Mill/Scala 运行时 |
| Git | 2.55.0 | 预安装 | `C:\Program Files\Git\` | 版本控制、子模块 |
| Python 3 | 3.15.0 | Scoop | `...\scoop\apps\python\` | 脚本工具 |
| Make | 4.4.1 | Scoop | `...\scoop\apps\make\` | 构建入口 |
| Mill | 0.12.3 | 手动下载 | `...\scoop\apps\mill\current\mill` | Chisel 构建系统 |
| Coursier | 2.1.26 | Scoop | `...\scoop\apps\coursier\` | Scala 包管理 |
| Verilator | 5.x.x | WSL2 apt | `/usr/local/bin/verilator` | RTL 仿真器 |
| firtool | 自动下载 | Mill resolver | 构建时自动解析 | FIRRTL 编译器 |
| Scala 2.13.15 | 自动下载 | Mill/Coursier | 构建时自动解析 | 宿主语言 |
| Chisel 6.7.0 | 自动下载 | Mill/Coursier | 构建时自动解析 | 硬件描述语言 |

---

## 6. 故障排除

| 问题 | 原因 | 解决方案 |
|------|------|---------|
| Mill 首次启动很慢 | 正在下载 Mill 核心和依赖 | 耐心等待，首次构建需 10-30 分钟 |
| `mill` 命令找不到 | 未创建 Scoop shim | 检查 `$env:SCOOP\shims\mill.ps1` 是否存在 |
| Java 版本不匹配 | Mill 0.12.3 需要 Java 8-17 | 确保 JAVA_HOME 指向 Java 21，或使用 Docker |
| 子模块为空 | 未运行 `git submodule update --init` | 重新运行子模块初始化命令 |
| yunsuan 编译错误 | 分支不匹配（kunminghu-v3 vs kunminghu-v2） | 将 yunsuan 切换到 master 分支 |
| Verilog 生成失败 | Chisel QMCMinimizer 算法爆炸 | 使用 Docker 环境（见优化文档） |

---

*文档生成日期：2026-10-20*
*环境：Windows 11 + WSL2 Ubuntu-24.04*
