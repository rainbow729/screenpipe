# Screenpipe 开发环境搭建与运行指南

> 基于 Windows 环境的完整实战指南，涵盖从零开始到成功运行的所有步骤。

---

## 目录

1. [项目核心组件与技术栈](#1-项目核心组件与技术栈)
2. [开发环境配置要求](#2-开发环境配置要求)
3. [Windows 完整安装步骤](#3-windows-完整安装步骤)
4. [启动不同组件的方法](#4-启动不同组件的方法)
5. [常见问题与解决方案](#5-常见问题与解决方案)
6. [验证项目是否成功运行](#6-验证项目是否成功运行)

---

## 1. 项目核心组件与技术栈

| 组件 | 技术栈 | 路径 | 说明 |
|------|--------|------|------|
| 核心引擎 | Rust | `crates/screenpipe-engine/` | 事件驱动屏幕/音频捕获、SQLite 存储、REST API |
| 屏幕捕获 | Rust | `crates/screenpipe-screen/` | 无障碍树 + OCR 回退 |
| 音频处理 | Rust + Whisper | `crates/screenpipe-audio/` | 本地语音转文字、说话人识别 |
| 数据库 | Rust + SQLite/SQLx | `crates/screenpipe-db/` | FTS5 全文搜索 |
| 桌面应用 | Tauri 2 + Next.js 15 | `apps/screenpipe-app-tauri/` | 前端 React 18 + 后端 Rust 嵌入引擎 |
| CLI 工具 | Node.js | `packages/cli/screenpipe/` | `npx screenpipe@latest record` |

**关键版本：**

- **Rust 1.93.1**（`rust-toolchain.toml` 自动指定）
- **Tauri 2.10**
- **Next.js 15.1.4**（静态导出模式 `output: 'export'`）
- **React 18** + **Zustand 5**
- **Bun 1.3.10**（JS/TS 包管理器，**禁止使用 npm/pnpm**）
- **SQLite + FTS5**（本地存储）

---

## 2. 开发环境配置要求

| 要求 | 最低值 | 推荐值 |
|------|--------|--------|
| RAM | 8 GB | 16 GB+ |
| 磁盘 | 10 GB | 20 GB+（含模型和缓存） |
| CPU | 现代 4 核 | 8 核+ |
| OS | Windows 10/11 | Windows 11 |
| GPU | 无要求 | DirectML 兼容 GPU（加速推理） |

**Windows 特有依赖：**

- Visual Studio 2022 Build Tools（含 MSVC + Windows SDK，**必须勾选"C++ 桌面开发"工作负载**）
- LLVM/Clang（Rust bindgen 需要）
- CMake
- 7-Zip + Wget（`pre_build.js` 下载 FFmpeg 时需要）
- FFmpeg（`pre_build.js` 会自动下载到 `src-tauri/ffmpeg/`）
- OpenBLAS（`pre_build.js` 会自动下载到 `src-tauri/openblas/`）

---

## 3. Windows 完整安装步骤

### 3.1 安装基础工具链

```powershell
# 通过 winget 安装所有必需工具
winget install -e --id Microsoft.VisualStudio.2022.BuildTools
winget install -e --id Rustlang.Rustup
winget install -e --id LLVM.LLVM
winget install -e --id Kitware.CMake
winget install -e --id GnuWin32.UnZip
winget install -e --id Git.Git
winget install -e --id JernejSimoncic.Wget
winget install -e --id 7zip.7zip

# 安装 Bun（JS 包管理器）
irm https://bun.sh/install.ps1 | iex
```

> ⚠️ **重要**：安装 Visual Studio Build Tools 时，必须在安装界面勾选 **"使用 C++ 的桌面开发"** 工作负载，否则 Rust 编译会找不到 `link.exe`。

### 3.2 设置环境变量

```powershell
# 设置 LIBCLANG_PATH（bindgen 编译 Rust 绑定代码时需要）
[System.Environment]::SetEnvironmentVariable('LIBCLANG_PATH', 'C:\Program Files\LLVM\bin', 'User')

# 将 GnuWin32 加入 PATH
[System.Environment]::SetEnvironmentVariable(
    'PATH',
    "$([System.Environment]::GetEnvironmentVariable('PATH', 'User'));C:\Program Files (x86)\GnuWin32\bin",
    'User'
)
```

> 设置后**必须重启终端**使环境变量生效。

### 3.3 克隆项目

```powershell
git clone https://github.com/screenpipe/screenpipe.git
cd screenpipe
```

### 3.4 配置 Cargo 镜像（网络受限环境）

如果访问 crates.io 或 GitHub 速度慢，编辑 `.cargo/config.toml`：

```toml
[source.crates-io]
replace-with = "rsproxy-sparse"

[source.rsproxy-sparse]
registry = "sparse+https://rsproxy.cn/index/"

[net]
git-fetch-with-cli = true
```

### 3.5 配置 Git 代理（如需）

```powershell
git config --global http.proxy http://127.0.0.1:代理端口
git config --global https.proxy http://127.0.0.1:代理端口
```

---

## 4. 启动不同组件的方法

### 4.1 方式一：仅运行 Rust 核心引擎（最快验证）

```powershell
# 在项目根目录编译（首次约 10~30 分钟）
cargo build --release

# 运行（默认端口 3030）
.\target\release\screenpipe.exe

# 使用不同端口/数据目录（避免与已安装版本冲突）
.\target\release\screenpipe.exe --port 3035 --data-dir C:\temp\sp
```

> Windows 上**不加** `--features metal,apple-intelligence`（那是 macOS 专用）。

### 4.2 方式二：开发模式启动桌面应用（推荐）

```powershell
cd apps\screenpipe-app-tauri

# 安装前端依赖
bun install

# 开发模式启动（自动执行 pre_build.js + 启动 Tauri）
bun tauri dev
```

`bun tauri dev` 的完整流程：
1. 执行 `pre_build.js`（自动下载 FFmpeg、OpenBLAS，复制 Bun 二进制，设置 VC++ 运行时）
2. 启动 Next.js 开发服务器（`http://localhost:1420`）
3. 编译 Rust 后端（**首次约 10~30 分钟，后续增量编译很快**）
4. 弹出 Tauri 桌面窗口

> ⚠️ 编译时看到 `Building [====>] 1406/1407: screenpipe-app(bin)` 卡在最后一步是**正常现象**，最后一个 binary 体量最大，需要等待 3~10 分钟。

### 4.3 方式三：构建安装包

```powershell
cd apps\screenpipe-app-tauri
bun install
bun tauri build
# 产物在 src-tauri/target/release/bundle/nsis/ 目录下
```

### 4.4 方式四：直接使用 CLI（无需编译）

```powershell
npx screenpipe@latest record
```

### 4.5 加速构建的 Profile

项目内置了 `release-dev` 配置，比完整 release 快 3~5 倍：

```powershell
cargo build --profile release-dev
```

---

## 5. 常见问题与解决方案

### 🔴 问题 1：FFmpeg 每次都重新下载

**原因**：`pre_build.js` 检查的是 `src-tauri/ffmpeg/` 目录（固定名称），如果你手动解压得到的是 `ffmpeg-8.0.1-full_build-shared/` 目录，名称不匹配，脚本每次都会认为 FFmpeg 不存在而重新下载。

**解决**：将目录重命名为 `ffmpeg`：

```powershell
cd apps\screenpipe-app-tauri\src-tauri
Rename-Item "ffmpeg-8.0.1-full_build-shared" "ffmpeg"
```

正确的目录结构：
```
src-tauri/ffmpeg/
├── bin/
│   ├── ffmpeg.exe
│   ├── ffprobe.exe
│   └── *.dll
└── lib/
    └── ...
```

### 🔴 问题 2：OpenBLAS 每次都重新下载

**原因**：同 FFmpeg，脚本检查的是 `src-tauri/openblas/` 目录，如果手动解压得到的是 `OpenBLAS-0.3.31-x64/`，名称不匹配。

**解决**：将目录重命名为 `openblas`：

```powershell
cd apps\screenpipe-app-tauri\src-tauri
Rename-Item "OpenBLAS-0.3.31-x64" "openblas"
```

正确的目录结构：
```
src-tauri/openblas/
├── bin/
│   └── libopenblas.dll
├── include/
│   └── cblas.h
└── lib/
    └── libopenblas.lib
```

### 🔴 问题 3：LIBCLANG_PATH 未设置导致 bindgen 编译失败

**错误信息**：`Unable to find libclang` 或 `the libclang binding was not found`

**解决**：
```powershell
# 确认 LLVM 已安装
"C:\Program Files\LLVM\bin\clang.exe" --version

# 设置环境变量（当前会话）
$env:LIBCLANG_PATH = "C:\Program Files\LLVM\bin"

# 永久设置
[System.Environment]::SetEnvironmentVariable('LIBCLANG_PATH', 'C:\Program Files\LLVM\bin', 'User')
```

### 🔴 问题 4：Visual Studio Build Tools 缺少 C++ 工具

**错误信息**：`link.exe not found` 或 MSVC 相关链接错误

**解决**：打开 **Visual Studio Installer** → 修改 Build Tools → 勾选 **"使用 C++ 的桌面开发"** → 确保安装了 Windows 10/11 SDK。

### 🔴 问题 5：端口 1420 被占用

**错误信息**：`listen EACCES: permission denied 0.0.0.0:1420` 或 `EADDRINUSE`

**解决方案 A**：找到并终止占用进程：
```powershell
netstat -ano | findstr :1420
# 找到 PID 后
taskkill /PID <PID> /F
# 或批量终止
taskkill /IM node.exe /F
taskkill /IM bun.exe /F
```

**解决方案 B**：检查 Windows 保留端口（os error -4092）：
```powershell
netsh interface ipv4 show excludedportrange protocol=tcp
```
如果 1420 在保留区间内，修改 `package.json` 的端口为 `1421`，同时修改 `src-tauri/tauri.conf.json` 中的 `devUrl` 为 `http://localhost:1421`（两处必须一致）。

### 🔴 问题 6：下载 NSIS 失败（`os error 11001`，DNS 解析失败）

**错误信息**：`failed to bundle project: io: 不知道这样的主机。 (os error 11001)`

**原因**：Tauri CLI 首次运行需要从 GitHub 下载 NSIS 打包工具，网络无法访问 `github.com`。

**解决方案 A**：配置代理：
```powershell
$env:HTTPS_PROXY = "http://127.0.0.1:代理端口"
$env:HTTP_PROXY  = "http://127.0.0.1:代理端口"
bun tauri dev
```

**解决方案 B**：手动下载放入缓存：
1. 用浏览器下载：`https://github.com/tauri-apps/binary-releases/releases/download/nsis-3.11/nsis-3.11.zip`
2. 创建缓存目录：
   ```powershell
   New-Item -ItemType Directory -Force "$env:LOCALAPPDATA\tauri\NSIS"
   ```
3. 解压到缓存目录：
   ```powershell
   Expand-Archive -Path "nsis-3.11.zip" -DestinationPath "$env:LOCALAPPDATA\tauri\NSIS" -Force
   ```

### 🔴 问题 7：GitHub 依赖下载失败（Cargo）

**错误信息**：`Failed to fetch` 或 `network error` 时拉取 GitHub 上的 crate

**解决**：为 Git 配置代理：
```powershell
git config --global http.proxy http://127.0.0.1:代理端口
git config --global https.proxy http://127.0.0.1:代理端口
```

同时在 `.cargo/config.toml` 中配置国内镜像：
```toml
[net]
git-fetch-with-cli = true
```

### 🟡 问题 8：音频设备持续报错（非致命）

**错误信息**：`device check error: No supported input configurations found`（每 2 秒重复）

**原因**：Intel Smart Sound Technology 麦克风阵列与 WASAPI 后端存在兼容性问题，`cpal` 音频库无法枚举其支持的输入配置。这是**非致命错误**，不影响应用的其他功能。

**解决**：
- 在系统声音设置里**禁用该麦克风阵列设备**，改用 USB 麦克风或耳机
- 或在应用设置里关闭麦克风捕获功能（如不需要音频转录）

### 🟡 问题 9：界面显示空白

**原因**：可能是 WebView2 缓存问题，或首次运行跳转到引导页面（`/onboarding`）是正常行为。

**解决**：
```powershell
# 清除 WebView2 缓存
Remove-Item -Recurse -Force "$env:APPDATA\screenpi.pe.dev" -ErrorAction SilentlyContinue
Remove-Item -Recurse -Force "$env:LOCALAPPDATA\screenpi.pe.dev" -ErrorAction SilentlyContinue
```

确认 WebView2 已安装：
```powershell
winget install -e --id Microsoft.EdgeWebView2Runtime
```

### 🟡 问题 10：数据库迁移错误

**错误信息**：`migration XXXXXXXXXX was previously applied but is missing`

**解决**：
```powershell
# 删除特定迁移记录
sqlite3 ~/.screenpipe/db.sqlite "DELETE FROM _sqlx_migrations WHERE version = XXXXXXXXXX;"

# 最坏情况：备份后重置
cp ~/.screenpipe/db.sqlite ~/.screenpipe/db.sqlite.backup
sqlite3 ~/.screenpipe/db.sqlite "DROP TABLE _sqlx_migrations;"
```

---

## 6. 验证项目是否成功运行

### 6.1 验证 Rust 核心引擎

```powershell
# 运行后检查 API 健康状态
curl http://localhost:3030/health

# 测试搜索接口
curl "http://localhost:3030/search?q=test&limit=5"
```

### 6.2 验证桌面应用

1. `bun tauri dev` 启动后桌面应出现 Tauri 窗口
2. 系统托盘区域应出现 screenpipe 图标
3. 首次运行会显示引导页 `/onboarding`（正常行为）
4. 访问 `http://localhost:1420` 应看到前端界面

### 6.3 运行测试

```powershell
# Rust 测试
cargo test

# 前端单元测试
cd apps\screenpipe-app-tauri
bun test
```

---

## 快速启动速查表

```powershell
# === 首次环境搭建 ===
winget install Microsoft.VisualStudio.2022.BuildTools Rustlang.Rustup LLVM.LLVM Kitware.CMake GnuWin32.UnZip Git.Git JernejSimoncic.Wget 7zip.7zip
irm https://bun.sh/install.ps1 | iex
[System.Environment]::SetEnvironmentVariable('LIBCLANG_PATH', 'C:\Program Files\LLVM\bin', 'User')
# 重启终端

# === 克隆项目 ===
git clone https://github.com/screenpipe/screenpipe.git
cd screenpipe

# === 方式 A：仅运行核心引擎 ===
cargo build --release
.\target\release\screenpipe.exe

# === 方式 B：运行完整桌面应用（开发模式）===
cd apps\screenpipe-app-tauri
bun install
bun tauri dev

# === 验证 ===
curl http://localhost:3030/health
```

---

*本文档基于项目实际构建经验整理，适用于 Windows 10/11 环境。*
