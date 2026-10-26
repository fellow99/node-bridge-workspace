# Node Bridge

[**English**](README.md) · [中文](README_zh.md)

Embed a **real Node.js runtime inside your mobile app's own process** — not a WebView, not a cloud
function, not a subprocess, not a shipped binary. A JNI/NAPI bootstrap calls
`dlopen("libnode.so")` + `node::Start` in-process, with a bidirectional JSON bridge between
JavaScript and the host language (Kotlin on Android, ArkTS on HarmonyOS).

This workspace is the umbrella for **four repositories** — on each platform the work is split into
two independently published packages: a **bridge** (API + loader, its own semver) and a
**runtime** (the engine binary, versioned exactly like Node.js). A [project website](node-bridge-website)
sits alongside them.

## The four repositories

| Repository | Platform / registry | Package | Ships |
|---|---|---|---|
| [aosp-node-bridge](aosp-node-bridge) | Android · Maven Central | `io.github.fellow99:node-bridge:0.2.0` | Kotlin API, JNI bootstrap (`libnode_bridge.so`), NAPI addon (`libnode_host_client.so`), JS payload |
| [aosp-node-runtime](aosp-node-runtime) | Android · Maven Central | `io.github.fellow99:node-runtime:24.2.0` | `libnode.so` + `libc++_shared.so` only — engine version **equals** the Node.js version |
| [ohos-node-bridge](ohos-node-bridge) | HarmonyOS · [ohpm](https://ohpm.openharmony.cn/) | `@fellow99/node-bridge@0.2.0` | ArkTS API + type declarations, NAPI bootstrap `.so`, crash forensics |
| [ohos-node-runtime](ohos-node-runtime) | HarmonyOS · [ohpm](https://ohpm.openharmony.cn/) | `@fellow99/node-runtime@24.2.0` | `libnode.so` (arm64-v8a) + `libc++_shared.so`, Node.js LICENSE, version constants |
| [node-bridge-website](node-bridge-website) | — | — | Project website (EN/中文, light/dark) |

Engine: **Node.js v24.2.0** · ABI **arm64-v8a** · Android `minSdk 24` · HarmonyOS `2in1` / `tablet`
(no phone).

## Why two packages

The bridge and the engine are coupled by a **runtime symbol contract**, not a compile-time link:

- `libnode_bridge.so` links only `log`/`dl` — it never links `libnode`. It finds the engine with
  `dlopen` and `node::Start` with `dlsym`, so the bridge is decoupled from the Node version.
- The NAPI addon leaves its `napi_*` symbols undefined on purpose and resolves them at runtime
  against the already-loaded `libnode.so` (`RTLD_GLOBAL`), guaranteeing Node's own N-API is used.
- The only symbols the bridge depends on are `_ZN4node5StartEiPPc` (`node::Start(int, char**)`)
  and the stable N-API surface — **swapping Node versions needs no bridge recompile**.

```
┌──────────────────────────── App process ────────────────────────────┐
│                                                                     │
│   Kotlin / ArkTS   NodeRuntime · HostBridge                         │
│        ▲  result (JSON)              │ host requests (JSON)         │
│        ▼                              ▼                             │
│   libnode_bridge.so  ◀─ slot table ─▶ libnode_host_client.so (NAPI) │
│        ▲ dlopen + dlsym                   ▲ process.dlopen          │
│        └────────────┐        ┌───────────┘                          │
│                 libnode.so (Node.js + V8)  ◀── bootstrap/host.js    │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

The engine binary is **never committed and never built by hand**: cloud CI cross-compiles it from
the official Node.js tarball, asserts it is `ET_DYN`, has no `PT_INTERP` and exports `node::Start`,
then publishes it. `runtime.lock.json` is the single source of truth for version, SHA-256, ELF
gates and provenance.

## Quick start

### Android (Maven Central)

```kotlin
// build.gradle.kts
dependencies {
    implementation("io.github.fellow99:node-bridge:0.2.0")
    runtimeOnly("io.github.fellow99:node-runtime:24.2.0") // engine, pinned to Node
}
```

```kotlin
// Register host capabilities FIRST, then start (order matters).
val rt = NodeRuntime.getInstance(context)
rt.host.register("fs.pickFolder") { """{"ok":true,"path":"/sdcard/Download"}""" }
rt.start(NodeRuntimeOptions(entry = "bootstrap.js", appEntry = "selfcheck.js"))
rt.state.collect { state -> /* Idle → Starting → Running → Exited/Failed */ }
```

### HarmonyOS (ohpm)

```bash
ohpm install @fellow99/node-bridge
ohpm install @fellow99/node-runtime@24.2.0
```

```ts
import { NodeRuntime } from '@fellow99/node-bridge';
NodeRuntime.getInstance().start(ctx, { scriptPath: '<sandbox>/payload/bootstrap.js' });
```

V8 JIT additionally requires the `ohos.permission.kernel.ALLOW_WRITABLE_CODE_MEMORY` ACL declared in
your `module.json5`; `--jitless` is the fallback (WebAssembly unavailable).

## Cloning

All four repositories plus the website are wired in as git submodules:

```bash
git clone --recurse-submodules https://github.com/fellow99/node-bridge-workspace.git
# or, after a plain clone:
git submodule update --init --recursive
```

```
node-bridge-workspace/
├── aosp-node-bridge/      # Android bridge — Kotlin API + JNI/NAPI bootstrap (Maven)
├── aosp-node-runtime/     # Android engine — cross-compiled libnode.so AAR (Maven)
├── ohos-node-bridge/      # HarmonyOS bridge — ArkTS API + NAPI bootstrap (ohpm)
├── ohos-node-runtime/     # HarmonyOS engine — libnode.so HAR (ohpm)
└── node-bridge-website/   # Project website — plain HTML/CSS/JS, EN/中文, light/dark
```

## Constraints (reported, never silently degraded)

- **`node::Start` is once per process** — a crash/exit is terminal (`Exited`/`Failed`); for a
  restartable form, run Node in a dedicated `:node` process.
- **JIT must be verified on device** — W^X/`execmem` rules and HarmonyOS ACLs can block V8 JIT.
  JIT and WebAssembly are verified working on Android 15 and HarmonyOS 2in1 devices.
- **arm64-v8a only**, no other ABIs; HarmonyOS targets `2in1`/`tablet`, not phones.

## License

All repositories are [MIT](LICENSE). The bundled Node.js engine is covered by the upstream
Node.js MIT license (including V8/OpenSSL/ICU third-party notices, shipped with each runtime
package).
