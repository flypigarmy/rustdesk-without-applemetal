# RustDesk without Apple Metal

Fork 自 [rustdesk/rustdesk](https://github.com/rustdesk/rustdesk)，移除 Apple Metal / VideoToolbox 依赖，使其能在 **macOS 10.14.6 (Mojave)** 上编译运行。

## 代码改动

仅修改 **2 个文件**，共 6 行：

### 1. Cargo.toml（删除 2 行）

移除 `hwcodec` 和 `vram` feature 声明，编译时不编译硬件编解码模块：

```diff
-hwcodec = ["scrap/hwcodec"]
-vram = ["scrap/vram"]
```

### 2. libs/scrap/build.rs（注释 7 行）

注释掉 VideoToolbox 等 5 个 Apple 框架的链接，避免在无 Metal 支持的系统上链接失败：

```diff
-    if target_os == "macos" || target_os == "ios" {
+/*     if target_os == "macos" || target_os == "ios" {
         println!("cargo:rustc-link-lib=framework=CoreFoundation");
         println!("cargo:rustc-link-lib=framework=CoreVideo");
         println!("cargo:rustc-link-lib=framework=CoreMedia");
         println!("cargo:rustc-link-lib=framework=VideoToolbox");
         println!("cargo:rustc-link-lib=framework=AVFoundation");
-    }
+    } */
```

**代价**：失去硬件编解码加速，远程桌面画面使用软件编解码。

---

## 构建踩坑全记录（macOS 10.14.6）

以下是在 Mac mini (macOS 10.14.6) 上从零构建 RustDesk 遇到的所有问题及解决方案。

### 一、依赖库缺失与编译

#### 1. vcpkg 只装了 opus，缺 libvpx

```bash
vcpkg install libvpx
```

**失败**：缺少 NASM 汇编器。

#### 2. Homebrew 安装 nasm 失败

macOS 10.14.6 太旧，Homebrew 已不支持，无法通过 `brew install nasm` 安装。

#### 3. 手动编译 libvpx（绕过汇编器）

利用 vcpkg 已下载的源码，`--target=generic-gnu` 绕过汇编器要求：

```bash
cd ~/vcpkg/downloads/libvpx-*
./configure --target=generic-gnu --disable-examples --disable-unit-tests --prefix=$VCPKG_ROOT/installed/x64-osx
make -j4
make install
```

编译产物 `libvpx.a`（~2MB）安装到 vcpkg installed 目录。

#### 4. 缺 libaom

aom 已编译在 `/usr/local/lib/`（不在 vcpkg 目录），需手动复制：

```bash
cp /usr/local/lib/libaom.a $VCPKG_ROOT/installed/x64-osx/lib/
cp -r /usr/local/include/aom $VCPKG_ROOT/installed/x64-osx/include/
```

#### 5. 缺 libyuv

libyuv 同样在 `/usr/local/lib/` 但不在 vcpkg，需手动复制：

```bash
cp /usr/local/lib/libyuv.a $VCPKG_ROOT/installed/x64-osx/lib/
cp -r /usr/local/include/libyuv $VCPKG_ROOT/installed/x64-osx/include/
```

> **教训**：vcpkg 不是唯一的库来源。如果系统已有 `/usr/local` 下的静态库，直接复制到 vcpkg installed 目录即可，不必重复编译。

#### 6. 缺少库文件清单

| 库 | 来源 | 安装方式 |
|----|------|----------|
| libvpx | vcpkg 源码手动编译 | `--target=generic-gnu` 绕过 NASM |
| libaom | `/usr/local/lib/` 已有 | 复制到 vcpkg installed |
| libyuv | `/usr/local/lib/` 已有 | 复制到 vcpkg installed |
| opus | vcpkg 正常安装 | `vcpkg install opus` |

---

### 二、打包 App Bundle 踩坑

#### 1. service 二进制未打包进 DMG ❌❌❌（最严重）

**现象**：安装后设备不在线，日志全是 `service: No such file or directory`。

**根因**：macOS 版 RustDesk 有两个进程：
- **UI 进程**：`RustDesk`（用户界面）
- **服务进程**：`service`（以 root 运行，通过 LaunchAgent `daemon.plist` 启动）

打包脚本只复制了 `rustdesk` 二进制，遗漏了 `target/release/service`（5.6MB）。服务进程从未启动 → IPC socket 不存在 → 服务端使用默认配置（rs-ny.rustdesk.com）→ `is_public()` 拦截心跳。

**解决**：打包时同时复制 `service` 二进制到 `RustDesk.app/Contents/MacOS/service`。

#### 2. src/ui/ 目录未复制 ❌❌

**现象**：点击"设置永久密码"按钮无反应/不显示。

**根因**：Sciter UI 版本的界面文件在 `src/ui/` 目录。旧版 UI 调用 `handler.permanent_password()` 获取密码，新版 Rust 已改用 `handler.is_local_permanent_password_set()`。打包脚本只替换了二进制，没复制 UI 文件 → 旧 UI 调新 Rust API 失败。

**解决**：打包时复制整个 `src/` 目录到 `RustDesk.app/Contents/MacOS/src/`。

#### 3. RustDesk.sh 启动脚本遗漏

**现象**：直接运行二进制无法正常启动。

**根因**：`RustDesk.app/Contents/MacOS/RustDesk.sh` 是 Sciter 版的启动入口脚本（设置环境变量后启动二进制），打包时遗漏。

**解决**：用旧 app bundle 作模板保留 `RustDesk.sh` 和 `Frameworks/libsciter.dylib`，只替换二进制和 UI 文件。

#### 4. macOS 大小写不敏感陷阱

**现象**：`ln -sf rustdesk RustDesk` 创建符号链接后，二进制文件被覆盖为 0 字节。

**根因**：macOS 文件系统默认大小写不敏感（APFS/HFS+ 均如此），`rustdesk` 和 `RustDesk` 指向同一个文件。`ln -sf rustdesk RustDesk` 不是创建符号链接，而是删除原文件再创建指向自己的链接。

**解决**：不要对大小写不同的同名文件创建符号链接，直接 `cp rustdesk RustDesk`。

---

### 三、签名踩坑

#### 1. 非交互 SSH 无法导入证书到 login keychain

`security import` 到 login keychain 需要用户交互确认（弹窗），SSH 非交互环境下无法完成。

**解决**：创建临时 keychain：

```bash
security create-keychain -p temp123 /tmp/sign.keychain
security unlock-keychain -p temp123 /tmp/sign.keychain
security import cert.p12 -k /tmp/sign.keychain -P password -T /usr/bin/codesign
security set-key-partition-list -S apple-tool:,apple: -k temp123 /tmp/sign.keychain
```

#### 2. 自签名证书无 codesign 权限

导入证书后 `security find-identity` 显示 0 个有效 codesigning identity，原因是证书缺少 codesigning 扩展。

**解决**：使用 `codesign --sign -`（ad-hoc 签名）或创建带 codesigning 扩展的证书。

---

### 四、Git 子模块损坏

**现象**：`libs/hbb_common` 目录为空，`.git` 丢失。

**解决**：重新 clone 整个仓库（`git clone --recurse-submodules`），旧仓库移至 `rustdesk_old`。

---

### 五、网络相关问题

#### 1. 企业防火墙拦截高端口

Mac mini 所在网络网关 `172.30.1.254` 对 TCP 高端口（21114-21117）出站做白名单，仅放行标准端口（22/80/443），但 UDP 21116 畅通。

**结果**：RustDesk UDP 注册正常（rendezvous），但 TCP 21114 HTTP API 心跳超时。

**解决**：服务器端修改 Linux 内核参数启用双 IP 跨网段宽松模式。

#### 2. Git 代理配置失效

Mac mini git 全局配置了 `http.proxy=http://192.168.100.106:10809`，但该代理不可用导致 `git push` 报 `Connection refused`。

**解决**：

```bash
git config --global --unset http.proxy
git config --global --unset https.proxy
```

---

### 六、远程操作注意事项

#### 1. PowerShell 变量展开

通过 PowerShell 发 SSH 命令时，`$HOME`、`$port` 等变量会被 PowerShell 先展开，导致远端命令异常。

**解决**：用单引号包裹，或用 base64 编码传输脚本：

```bash
echo SCRIPT_BASE64 | base64 -D > /tmp/script.sh && bash /tmp/script.sh
```

#### 2. SSH 长命令超时

SSH 会话执行长耗时命令（如编译、下载）容易被 SIGKILL 终止。

**解决**：用 `nohup` 后台运行 + 轮询结果文件：

```bash
nohup long_command > /tmp/output.log 2>&1 &
# 后续检查
cat /tmp/output.log
```

---

## 完整打包脚本要点

正确的打包流程应包含：

```
RustDesk.app/
├── Contents/
│   ├── Info.plist
│   ├── MacOS/
│   │   ├── RustDesk          ← 主二进制（25MB）
│   │   ├── service           ← 服务二进制（5.6MB）⚠ 必须包含
│   │   └── RustDesk.sh       ← Sciter 版启动脚本 ⚠ 必须包含
│   ├── Frameworks/
│   │   └── libsciter.dylib   ← Sciter UI 引擎 ⚠ 必须包含
│   ├── Resources/
│   │   └── entitlements.plist
│   └── src/
│       └── ui/               ← Sciter UI 文件 ⚠ 必须包含
│           ├── index.tis
│           ├── ...
```

## 构建环境

| 项目 | 版本/路径 |
|------|-----------|
| macOS | 10.14.6 (Mojave) |
| Xcode | 10.3 |
| Rust | stable (via rustup) |
| vcpkg | 2023.04.15 |
| 编译时间 | ~9 分钟（4 核） |
| 产物大小 | rustdesk 25MB + service 5.6MB |
| DMG 大小 | ~37-38MB |

## Fork 信息

- 上游仓库：[rustdesk/rustdesk](https://github.com/rustdesk/rustdesk)
- 本 Fork 仅用于在无 Apple Metal 支持的旧 macOS 上构建 RustDesk
- 代码改动最小化，仅移除编译依赖，不改变功能逻辑
