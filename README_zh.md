# Node Bridge

[English](README.md) · [**中文**](README_zh.md)

把**真正的 Node.js 运行时嵌进移动应用自己的进程**——不是 WebView、不是云函数、不是子进程、
也不是随包下发的独立二进制。JNI/NAPI 引导件在进程内调用
`dlopen("libnode.so")` + `node::Start`，并在 JavaScript 与宿主语言（Android 上是 Kotlin、
鸿蒙上是 ArkTS）之间建立双向 JSON 桥。

本工作区统辖**十个仓库**——每个平台都拆成两个独立发布的包：**桥**（API 与加载器，走自己的
semver）与**运行时**（引擎二进制，版本号与 Node.js 完全一致）；Android 侧另有一个与引擎同版本的
**node-pty** 伴生件，以及一组 **`*-cli` 命令行工具**（`node`、`python`、`git`、`uv`、`pnpm`，
以 AAR 打包、经伪终端启动）。旁边还有一个[项目网站](node-bridge-website)子工程。

## 十个仓库

### Android · 引擎线（Maven Central）

| 仓库 | 发布面 | 包坐标 | 装什么 |
|---|---|---|---|
| [aosp-node-bridge](aosp-node-bridge) | Maven Central | `io.github.fellow99:node-bridge:0.2.0` | Kotlin API、JNI 引导件（`libnode_bridge.so`）、NAPI addon（`libnode_host_client.so`）、JS 载荷 |
| [aosp-node-runtime](aosp-node-runtime) | Maven Central | `io.github.fellow99:node-runtime:26.13.0` | 只有 `libnode.so` + `libc++_shared.so`——版本号**等于** Node.js 版本 |
| [aosp-node-pty](aosp-node-pty) | Maven Central | `io.github.fellow99:node-pty:26.13.0` | 只有 `libpty_node.so`——交叉编译的 [node-pty](https://www.npmjs.com/package/node-pty) 原生件，`DT_NEEDED libnode.so`，与运行时一对一配对 |

### Android · CLI 工具线（GitHub Release；Maven Central 待发）

| 仓库 | GitHub Release | 包坐标 | 装什么 |
|---|---|---|---|
| [aosp-node-cli](aosp-node-cli) | v26.13.0 | `io.github.fellow99:node-cli` | `libnode_cli.so`（exec 型 PIE `node`）、npm/npx/corepack 三壳、full-icu |
| [aosp-python-cli](aosp-python-cli) | v3.14.10 | `io.github.fellow99:python-cli` | `libpython_cli.so`（PEP 738）、stdlib + ensurepip + cacert.pem 资产 |
| [aosp-git-cli](aosp-git-cli) | v2.56.0 | `io.github.fellow99:git-cli` | `libgit_cli.so`，zlib/libcurl/OpenSSL 静态链入，https 开箱可用 |
| [aosp-uv-cli](aosp-uv-cli) | v0.13.6 | `io.github.fellow99:uv-cli` | `libuv_cli.so`（Rust，`aarch64-linux-android`），rustls TLS |
| [aosp-pnpm-cli](aosp-pnpm-cli) | v12.16.1 | `io.github.fellow99:pnpm-cli` | `libpnpm_cli.so`（Rust），运行期需 `PATH` 上有 `node`（aosp-node-cli） |

> ℹ️ 五个 CLI 包目前均只走 **GitHub Release 归档**；Maven Central（同坐标）待发。各仓 README
> 列有门禁与边界，例如 python sdist 不可设备编译、pnpm lifecycle 脚本需要 `PATH` 上有
> `node` 可执行。

### 鸿蒙 · ohpm

| 仓库 | 发布面 | 包坐标 | 装什么 |
|---|---|---|---|
| [ohos-node-bridge](ohos-node-bridge) | [ohpm](https://ohpm.openharmony.cn/) | `@fellow99/node-bridge@0.2.0` | ArkTS API + 类型 `.d.ets`、NAPI 引导件两个 `.so`、日志/崩溃取证 |
| [ohos-node-runtime](ohos-node-runtime) | [ohpm](https://ohpm.openharmony.cn/) | `@fellow99/node-runtime@26.13.0` | `libnode.so`（arm64-v8a）+ `libc++_shared.so`、版本常量、Node 官方 LICENSE |

引擎：Android **Node.js v26.13.0** · 鸿蒙 **Node.js v26.13.0** · ABI **arm64-v8a** · Android
`minSdk 24` · 鸿蒙 `2in1`/`tablet`（不做手机）。node-pty 原生件与 Android 运行时同版本
（NODE_MODULE_VERSION 147）。

[项目网站](node-bridge-website)（中英双语、浅色/暗色）也属于本工作区。

## 发布版本

> ⚠️ **所有制品只发布 AArch64（arm64-v8a）一个架构**——无 x86 / x86_64 / arm32 包。
### Android · Maven Central（`io.github.fellow99:*`）

| 包 | 版本 | 操作系统 | 架构 | 链接 |
|---|---|---|---|---|
| `node-bridge` | 0.1.0 | Android | arm64-v8a | https://repo1.maven.org/maven2/io/github/fellow99/node-bridge/0.1.0/ |
| `node-bridge` | 0.2.0 | Android | arm64-v8a | https://repo1.maven.org/maven2/io/github/fellow99/node-bridge/0.2.0/ |
| `node-runtime` | 18.20.4 | Android | arm64-v8a | https://repo1.maven.org/maven2/io/github/fellow99/node-runtime/18.20.4/ |
| `node-runtime` | 22.23.4 | Android | arm64-v8a | https://repo1.maven.org/maven2/io/github/fellow99/node-runtime/22.23.4/ |
| `node-runtime` | 24.2.0 | Android | arm64-v8a | https://repo1.maven.org/maven2/io/github/fellow99/node-runtime/24.2.0/ |
| `node-runtime` | 24.22.1 | Android | arm64-v8a | https://repo1.maven.org/maven2/io/github/fellow99/node-runtime/24.22.1/ |
| `node-runtime` | 26.13.0 | Android | arm64-v8a | https://repo1.maven.org/maven2/io/github/fellow99/node-runtime/26.13.0/ |
| `node-pty` | 22.23.4 | Android | arm64-v8a | https://repo1.maven.org/maven2/io/github/fellow99/node-pty/22.23.4/ |
| `node-pty` | 24.22.1 | Android | arm64-v8a | https://repo1.maven.org/maven2/io/github/fellow99/node-pty/24.22.1/ |
| `node-pty` | 26.13.0 | Android | arm64-v8a | https://repo1.maven.org/maven2/io/github/fellow99/node-pty/26.13.0/ |

### Android · CLI 工具（GitHub Release）

| 包 | 版本 | 操作系统 | 架构 | 链接 |
|---|---|---|---|---|
| `node-cli` | v26.13.0 | Android | arm64-v8a | https://github.com/fellow99/aosp-node-cli/releases/tag/v26.13.0 |
| `python-cli` | v3.14.10 | Android | arm64-v8a | https://github.com/fellow99/aosp-python-cli/releases/tag/v3.14.10 |
| `git-cli` | v2.56.0 | Android | arm64-v8a | https://github.com/fellow99/aosp-git-cli/releases/tag/v2.56.0 |
| `uv-cli` | v0.13.6 | Android | arm64-v8a | https://github.com/fellow99/aosp-uv-cli/releases/tag/v0.13.6 |
| `pnpm-cli` | v12.16.1 | Android | arm64-v8a | https://github.com/fellow99/aosp-pnpm-cli/releases/tag/v12.16.1 |

### 鸿蒙 · ohpm（`@fellow99/*`）

| 包 | 版本 | 操作系统 | 架构 | 链接 |
|---|---|---|---|---|
| `node-bridge` | 0.1.0 | OpenHarmony | arm64-v8a | https://ohpm.openharmony.cn/#/cn/detail/@fellow99%2Fnode-bridge/v/0.1.0 |
| `node-bridge` | 0.2.0 | OpenHarmony | arm64-v8a | https://ohpm.openharmony.cn/#/cn/detail/@fellow99%2Fnode-bridge/v/0.2.0 |
| `node-runtime` | 24.2.0 | OpenHarmony | arm64-v8a | https://ohpm.openharmony.cn/#/cn/detail/@fellow99%2Fnode-runtime/v/24.2.0 |
| `node-runtime` | 22.23.4 | OpenHarmony | arm64-v8a | https://ohpm.openharmony.cn/#/cn/detail/@fellow99%2Fnode-runtime/v/22.23.4 |
| `node-runtime` | 24.22.1 | OpenHarmony | arm64-v8a | https://ohpm.openharmony.cn/#/cn/detail/@fellow99%2Fnode-runtime/v/24.22.1 |
| `node-runtime` | 26.13.0 | OpenHarmony | arm64-v8a | https://ohpm.openharmony.cn/#/cn/detail/@fellow99%2Fnode-runtime/v/26.13.0 |

## 为什么拆两个包

桥与引擎靠**运行期符号契约**咬合，而非编译期链接：

- `libnode_bridge.so` 只链接 `log`/`dl`，**从不链接 `libnode`**——它用 `dlopen` 找引擎、用
  `dlsym` 找 `node::Start`，因此桥与 Node 版本解耦。
- NAPI addon 故意保留未定义的 `napi_*` 符号（`-z undefs`），运行期再对着已加载的
  `libnode.so`（`RTLD_GLOBAL`）解析——保证用的是 Node 自己的 NAPI 实现。
- 桥只依赖两个符号：`_ZN4node5StartEiPPc`（`node::Start(int, char**)`）与稳定的 N-API
  表面——**换 Node 版本不用重编桥**（已真机验证）。

```
┌──────────────────────────── App process ────────────────────────────┐
│                                                                     │
│   Kotlin / ArkTS   NodeRuntime · HostBridge                         │
│        ▲  result (JSON)              │ host requests (JSON)         │
│        ▼                              ▼                             │
│   libnode_bridge.so  ◀─ slot table ─→ libnode_host_client.so (NAPI) │
│        ▲ dlopen + dlsym                   ▲ process.dlopen          │
│        └────────────┐        ┌───────────┘                          │
│                 libnode.so (Node.js + V8)  ◀── bootstrap/host.js    │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

引擎二进制**不入库、也不手工构建**：云 CI 从官方 Node.js 源码包交叉编译，断言其为 `ET_DYN`、
无 `PT_INTERP`、导出 `node::Start` 后发布。`runtime.lock.json` 是版本、SHA-256、ELF 门禁与
来源的唯一事实源；CLI 工具同样以各自的 `*-cli.lock.json` 施行同一套纪律。

## 快速开始

### Android —— 进程内 Node（Maven Central）

```kotlin
// build.gradle.kts
dependencies {
    implementation("io.github.fellow99:node-bridge:0.2.0")
    runtimeOnly("io.github.fellow99:node-runtime:26.13.0") // 引擎，按 Node 版本钉死
    // 可选：交叉编译的 node-pty 原生件，与运行时同版本
    runtimeOnly("io.github.fellow99:node-pty:26.13.0")
}
```

```kotlin
// 先注册宿主能力，再启动（顺序不能反）。
val rt = NodeRuntime.getInstance(context)
rt.host.register("fs.pickFolder") { """{"ok":true,"path":"/sdcard/Download"}""" }
rt.start(NodeRuntimeOptions(entry = "bootstrap.js", appEntry = "selfcheck.js"))
rt.state.collect { state -> /* Idle → Starting → Running → Exited/Failed */ }
```

### Android —— CLI 工具（GitHub Release）

```kotlin
// build.gradle.kts —— 把归档放进你的仓库，或按各仓 README 的安装说明
dependencies {
    runtimeOnly("io.github.fellow99:node-runtime:26.13.0") // 提供 libc++_shared.so
}
```

每个 `*-cli` 交付 exec 型 `lib*_cli.so` + 资产。把 `PATH` 指向 `nativeLibraryDir`（或在
`$HOME/bin` 下做 symlink farm），经伪终端启动工具——例如
`pty.spawn(nativeDir + "/libnode_cli.so", ["-e", "console.log(1)"])`。
必须 `useLegacyPackaging=true` 让二进制物理落盘。部分工具可叠加：python + uv 组成完整
Python 工具链；pnpm 运行期需要 `PATH` 上有 `node`（aosp-node-cli）。

### 鸿蒙（ohpm）

```bash
ohpm install @fellow99/node-bridge
ohpm install @fellow99/node-runtime@26.13.0
```

```ts
import { NodeRuntime } from '@fellow99/node-bridge';
NodeRuntime.getInstance().start(ctx, { scriptPath: '<沙箱绝对路径>/payload/bootstrap.js' });
```

另外，V8 JIT 需要在自己的 `module.json5` 按自己的包名申请 ACL：
`ohos.permission.kernel.ALLOW_WRITABLE_CODE_MEMORY`；没有它可走 `--jitless` 兜底，但
WebAssembly 不可用。

## 克隆本工作区

十个仓库与网站均以 git submodule 形式挂接：

```bash
git clone --recurse-submodules https://github.com/fellow99/node-bridge-workspace.git
# 或者普通克隆后补拉：
git submodule update --init --recursive
```

```
node-bridge-workspace/
├── aosp-node-bridge/      # Android 桥 —— Kotlin API + JNI/NAPI 引导件（Maven）
├── aosp-node-runtime/     # Android 引擎 —— 交叉编译的 libnode.so AAR（Maven）
├── aosp-node-pty/         # Android node-pty 原生件 —— 交叉编译的 libpty_node.so AAR（Maven）
├── aosp-node-cli/         # Android `node` CLI —— exec 型 libnode_cli.so + npm/npx/corepack（Release）
├── aosp-python-cli/       # Android `python` CLI —— PEP 738 libpython_cli.so + stdlib（Release）
├── aosp-git-cli/          # Android `git` CLI —— libgit_cli.so，静态 curl/openssl 的 https（Release）
├── aosp-uv-cli/           # Android `uv` CLI —— Rust libuv_cli.so（Release）
├── aosp-pnpm-cli/         # Android `pnpm` CLI —— Rust libpnpm_cli.so（Release）
├── ohos-node-bridge/      # 鸿蒙桥 —— ArkTS API + NAPI 引导件（ohpm）
├── ohos-node-runtime/     # 鸿蒙引擎 —— libnode.so HAR（ohpm）
└── node-bridge-website/   # 项目网站 —— 纯 HTML/CSS/JS，中英双语、浅色暗色
```

## 已知限制（实测结论，绝不静默降级）

- **`node::Start` 每进程只能一次**——崩溃/退出即终态（`Exited`/`Failed`）；要可重启就把
  Node 放进独立的 `:node` 进程。
- **JIT 必须真机验证**——W^X/`execmem` 限制与鸿蒙 ACL 都可能挡住 V8 JIT。JIT 与
  WebAssembly 已在 Android 15 与鸿蒙 2in1 真机上验证通过。
- **只支持 arm64-v8a**，无其他 ABI；鸿蒙只面向 `2in1`/`tablet`，不做手机。
- **CLI 工具是设备二进制，不是 SDK**——若干平台限制真实存在并如实披露：python sdist
  不可设备编译；`uv python install` 被拒绝（C2 红线）；exec 必须 `useLegacyPackaging=true`；
  pnpm lifecycle 需要 `PATH` 上有 `node`。

## 许可

所有仓库均为 [MIT](LICENSE)。随包分发的 Node.js 引擎适用上游 Node.js MIT 许可（含
V8/OpenSSL/ICU 等第三方许可全文，随各运行时包附带）；CLI AAR 自带各自的
THIRD-PARTY-NOTICES.md（如 git 为 GPLv2——源码包随 Release 归档附带）。
