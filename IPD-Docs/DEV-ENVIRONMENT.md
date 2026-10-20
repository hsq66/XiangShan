# 香山昆明湖 V2 开发环境工具链部署指南

## 工具链状态总览

| 工具 | 版本 | 安装位置 | 状态 | 用途 |
|------|------|---------|------|------|
| Java (Eclipse Adoptium Temurin) | 21.0.12.1 LTS | `C:\Program Files\Eclipse Adoptium\jdk-21\` | ✅ 已安装 | Mill/Scala 运行时 |
| Git | 2.55.0.windows.3 | `C:\Program Files\Git\` | ✅ 已安装 | 版本控制、子模块 |
| Python 3 | 3.15.0 | `C:\Users\Administrator\AppData\Local\scoop\apps\python\` | ✅ 已安装 (Scoop) | 脚本工具 |
| Make | 4.4.1 | `C:\Users\Administrator\AppData\Local\scoop\apps\make\` | ✅ 已安装 (Scoop) | 构建入口 (Makefile) |
| Mill | 0.12.3 | `C:\Users\Administrator\AppData\Local\scoop\apps\mill\current\mill` | ✅ 已安装 | Chisel 构建系统 |
| Coursier | 2.1.26 | `C:\Users\Administrator\AppData\Local\scoop\apps\coursier\` | ✅ 已安装 (Scoop) | Scala 包管理 |
| Verilator | 已安装 | WSL2 Ubuntu `/usr/local/bin/verilator` | ✅ 已安装 (WSL2) | RTL 仿真真器 |
| Node.js | 24.21.0 | `C:\Users\Administrator\AppData\Local\Author Software\nvm\` | ✅ 已安装 | 前端工具 |
| npm | 包含在 Node | 同上 | ✅ 已安装 | 包管理 |
| WSL2 | Ubuntu-24.04 | Windows WSL2 | ✅ 已安装 | Linux 仿真环境 |
| firtool | 自动下载 | 由 Mill firtool-resolver 管理 | ⏳ 自动 | FIRRTL 编译器 |
| Scala | 自动下载 | 由 Mill/Coursier 管理 | ⏳ 自动 | 宿主语言 |
| Chisel | 自动下载 | 由 Mill/Coursier 管理 | ⏳ 自动 | 硬件描述语言 |

---

## 快速使用指南

### 1. 使用 Mill 生成 Verilog RTL (Windows PowerShell)

Mill 脚本需要在 Git Bash 中运行。在项目根目录下：

```powershell
# 进入项目目录
cd D:\cdkv\xiangshan\kunminghuV2\XiangShan-kunminghu-v2

# 使用 Git Bash 运行 Mill
"C:\Program Files\Git\bin\bash.exe" -c 'cd /d/cdkv/xiangshan/kunminghuV2/XiangShan-kunminghu-v2 && ./mill --version'

# 生成 Verilog (默认配置)
"C:\Program Files\Git\bin\bash.exe" -c 'cd /d/cdkv/xiangshan/kunminghuV2/XiangShan-kunminghu-v2 && ./mill xiangshan.runMain top.TopMain --target-dir build/rtl --config KunminghuV2Config --issue E.b --num-cores 1'

# 编译检查
"C:\Program Files\Git\bin\bash.exe" -c 'cd /d/cdkv/xiangshan/kunminghuV2/XiangShan-kunminghu-v2 && ./mill xiangshan.compile'
```

### 2. 使用 Verilator 运行仿真 (WSL2 Ubuntu)

Mill 生成的 Verilog 位于 Windows 文件系统，WSL2 可以通过 `/mnt/d/` 访问：

```bash
# 在 WSL2 Ubuntu 中运行
wsl -d Ubuntu-24.04 --exec /bin/bash -c "
  cd /mnt/d/cdkv/xiangshan/kunminghuV2/XiangShan-kunminghu-v2
  verilator --version
"
```

### 3. 使用 Makefile (Windows + WSL2 混合)

```powershell
# 在 Windows PowerShell 中
cd D:\cdkv\xiangshan\kunminghuV2\XiangShan-kunminghu-v2

# 生成 Verilog (通过 Mill)
make verilog CONFIG=KunminghuV2Config

# 或者使用 WSL2 中的 make 运行完整仿真
wsl -d Ubuntu-24.04 --exec /bin/bash -c "
  cd /mnt/d/cdkv/xiangshan/kunminghuV2/XiangShan-kunminghu-v2
  make emu CONFIG=KunminghuV2Config
"
```

---

## Mill 命令便利封装 (PowerShell)

在项目根目录创建 `mill.ps1` 以便在 PowerShell 中直接使用：

```powershell
# 项目根目录 mill.ps1
param([string[]]$Args)
& "C:\Program Files\Git\bin\bash.exe" -c "cd /d/cdkv/xiangshan/kunminghuV2/XiangShan-kunminghu-v2 && ./mill $($Args -join ' ')"
```

使用示例：
```powershell
.\mill.ps1 --version
.\mill.ps1 xiangshan.compile
.\mill.ps1 xiangshan.runMain top.TopMain --help
```

---

## 首次构建注意事项

1. **首次运行 Mill 会下载大量依赖** (Chisel 6.7.0, firtool, rocket-chip 等)，预计需要 10-30 分钟，具体取决于网络速度。
2. **Mill 构建日志** 位于 `build/` 目录。
3. **生成的 Verilog** 位于 `build/rtl/` 目录 (`XSTop.sv`, `SimTop.sv`)。
4. **Java 21 兼容性**：Mill 0.12.3 在 Java 21 上可运行，但首次启动可能有 stdin 相关警告（不影响功能）。

---

## 故障排除

| 问题 | 解决方案 |
|------|---------|
| Mill 首次启动很慢 | 正在下载依赖，请耐心等待 |
| Mill 报 Java 版本错误 | 确保 JAVA_HOME 指向 Java 21 |
| Verilator 找不到 | 在 WSL2 中使用：`wsl -d Ubuntu-24.04 --exec /bin/bash -c "verilator --version"` |
| Make 找不到 mill 脚本 | 确保项目根目录有 `mill` 文件 (已部署) |
| 子模块缺失 | 运行 `git submodule update --init` |

---

## Docker 替代方案

如果本机工具链有问题，项目提供了 Docker 环境：

```dockerfile
FROM ghcr.io/openxiangshan/xs-env:latest
WORKDIR /xiangshan
COPY . /xiangshan
RUN make init && make verilog CONFIG=KunminghuV2Config
```

---

*文档生成日期：2026-10-20*
*环境：Windows 11 + WSL2 Ubuntu-24.04*
