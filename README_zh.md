# Node Bridge

[English](README.md) · [**中文**](README_zh.md)

把**真正的 Node.js 运行时嵌进移动应用自己的进程**——不是 WebView、不是云函数、不是子进程、
也不是随包下发的独立二进制。JNI/NAPI 引导件在进程内调用
`dlopen("libnode.so")` + `node::Start`，并在 JavaScript 与宿主语言（Android 上是 Kotlin、
鸿蒙上是 ArkTS）之间建立双向 JSON 桥。

本工作区统辖**四个仓库**——每个平台都拆成两个独立发布的包：**桥**（API 与加载器，走自己的
semver）与**运行时**（引擎二进制，版本号与 Node.js 完全一致）。旁边还有一个[项目网站](node-bridge-website)
子工程。

## 四个仓库

| 仓库 | 平台 / 发布面 | 包坐标 | 装什么 |
|---|---|---|---|
| [aosp-node-bridge](aosp-node-bridge) | Android · Maven Central | `io.github.fellow99:node-bridge:0.2.0` | Kotlin API、JNI 引导件（`libnode_bridge.so`）、NAPI addon（`libnode_host_client.so`）、JS 载荷 |
| [aosp-node-runtime](aosp-node-runtime) | Android · Maven Central | `io.github.fellow99:node-runtime:24.2.0` | 只有 `libnode.so` + `libc++_shared.so`——版本号**等于** Node.js 版本 |
| [ohos-node-bridge](ohos-node-bridge) | 鸿蒙 · [ohpm](https://ohpm.openharmony.cn/) | `@fellow99/node-bridge@0.2.0` | ArkTS API + 类型 `.d.ets`、NAPI 引导件两个 `.so`、日志/崩溃取证 |
| [ohos-node-runtime](ohos-node-runtime) | 鸿蒙 · [ohpm](https://ohpm.openharmony.cn/) | `@fellow99/node-runtime@24.2.0` | `libnode.so`（arm64-v8a）+ `libc++_shared.so`、版本常量、Node 官方 LICENSE |
| [node-bridge-website](node-bridge-website) | — | — | 项目网站（中英双语、浅色/暗色） |

引擎：**Node.js v24.2.0** · ABI **arm64-v8a** · Android `minSdk 24` · 鸿蒙 `2in1`/`tablet`
（不做手机）。

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
来源的唯一事实源。

## 快速开始

### Android（Maven Central）

```kotlin
// build.gradle.kts
dependencies {
    implementation("io.github.fellow99:node-bridge:0.2.0")
    runtimeOnly("io.github.fellow99:node-runtime:24.2.0") // 引擎，按 Node 版本钉死
}
```

```kotlin
// 先注册宿主能力，再启动（顺序不能反）。
val rt = NodeRuntime.getInstance(context)
rt.host.register("fs.pickFolder") { """{"ok":true,"path":"/sdcard/Download"}""" }
rt.start(NodeRuntimeOptions(entry = "bootstrap.js", appEntry = "selfcheck.js"))
rt.state.collect { state -> /* Idle → Starting → Running → Exited/Failed */ }
```

### 鸿蒙（ohpm）

```bash
ohpm install @fellow99/node-bridge
ohpm install @fellow99/node-runtime@24.2.0
```

```ts
import { NodeRuntime } from '@fellow99/node-bridge';
NodeRuntime.getInstance().start(ctx, { scriptPath: '<沙箱绝对路径>/payload/bootstrap.js' });
```

另外，V8 JIT 需要在自己的 `module.json5` 按自己的包名申请 ACL：
`ohos.permission.kernel.ALLOW_WRITABLE_CODE_MEMORY`；没有它可走 `--jitless` 兜底，但
WebAssembly 不可用。

## 克隆本工作区

四个仓库与网站均以 git submodule 形式挂接：

```bash
git clone --recurse-submodules https://github.com/fellow99/node-bridge-workspace.git
# 或者普通克隆后补拉：
git submodule update --init --recursive
```

```
node-bridge-workspace/
├── aosp-node-bridge/      # Android 桥 —— Kotlin API + JNI/NAPI 引导件（Maven）
├── aosp-node-runtime/     # Android 引擎 —— 交叉编译的 libnode.so AAR（Maven）
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

## 许可

四个仓库均为 [MIT](LICENSE)。随包分发的 Node.js 引擎适用上游 Node.js MIT 许可（含
V8/OpenSSL/ICU 等第三方许可全文，随各运行时包附带）。
