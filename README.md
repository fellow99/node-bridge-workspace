# Node Bridge

[**English**](README.md) · [中文](README_zh.md)

Embed a **real Node.js runtime inside your mobile app's own process** — not a WebView, not a cloud
function, not a subprocess, not a shipped binary. A JNI/NAPI bootstrap calls
`dlopen("libnode.so")` + `node::Start` in-process, with a bidirectional JSON bridge between
JavaScript and the host language (Kotlin on Android, ArkTS on HarmonyOS).

This workspace is the umbrella for **ten repositories** — on each platform the work is split into
two independently published packages: a **bridge** (API + loader, its own semver) and a
**runtime** (the engine binary, versioned exactly like Node.js). Android also ships a companion
**node-pty** addon, versioned with the engine, plus a family of **`*-cli` tool binaries**
(`node`, `python`, `git`, `uv`, `pnpm`) packaged as AARs and launched through a pseudoterminal.
A [project website](node-bridge-website) sits alongside them.

## The ten repositories

### Android · engine family (Maven Central)

| Repository | Registry | Package | Ships |
|---|---|---|---|
| [aosp-node-bridge](aosp-node-bridge) | Maven Central | `io.github.fellow99:node-bridge:0.2.0` | Kotlin API, JNI bootstrap (`libnode_bridge.so`), NAPI addon (`libnode_host_client.so`), JS payload |
| [aosp-node-runtime](aosp-node-runtime) | Maven Central | `io.github.fellow99:node-runtime:26.13.0` | `libnode.so` + `libc++_shared.so` only — engine version **equals** the Node.js version |
| [aosp-node-pty](aosp-node-pty) | Maven Central | `io.github.fellow99:node-pty:26.13.0` | `libpty_node.so` only — cross-compiled [node-pty](https://www.npmjs.com/package/node-pty) addon, `DT_NEEDED libnode.so`; paired 1:1 with the runtime |

### Android · CLI tools (GitHub Release; Maven Central pending)

| Repository | GitHub Release | Package | Ships |
|---|---|---|---|
| [aosp-node-cli](aosp-node-cli) | v26.13.0 | `io.github.fellow99:node-cli` | `libnode_cli.so` (exec-form PIE `node`), npm/npx/corepack shell shims, full-icu |
| [aosp-python-cli](aosp-python-cli) | v3.14.10 | `io.github.fellow99:python-cli` | `libpython_cli.so` (PEP 738), stdlib + ensurepip + cacert.pem assets |
| [aosp-git-cli](aosp-git-cli) | v2.56.0 | `io.github.fellow99:git-cli` | `libgit_cli.so`, zlib/libcurl/OpenSSL statically linked, https out of the box |
| [aosp-uv-cli](aosp-uv-cli) | v0.13.6 | `io.github.fellow99:uv-cli` | `libuv_cli.so` (Rust, `aarch64-linux-android`), rustls TLS |
| [aosp-pnpm-cli](aosp-pnpm-cli) | v12.16.1 | `io.github.fellow99:pnpm-cli` | `libpnpm_cli.so` (Rust), needs a `node` on `PATH` (aosp-node-cli) at runtime |

> ℹ️ All five CLI packages currently ship via **GitHub Release tarballs** only; Maven Central
> publication (same coordinates) is pending. See each repository's README for gates and
> caveats — e.g. python sdist cannot be compiled on device, and pnpm lifecycle scripts require a
> `node` executable on `PATH`.

### HarmonyOS · ohpm

| Repository | Registry | Package | Ships |
| [ohos-node-bridge](ohos-node-bridge) | [ohpm](https://ohpm.openharmony.cn/) | `@fellow99/node-bridge@0.2.0` | ArkTS API + type declarations, NAPI bootstrap `.so`, crash forensics |
|---|---|---|---|
| [ohos-node-runtime](ohos-node-runtime) | [ohpm](https://ohpm.openharmony.cn/) | `@fellow99/node-runtime@26.13.0` | `libnode.so` (arm64-v8a) + `libc++_shared.so`, Node.js LICENSE, version constants |

The [project website](node-bridge-website) (EN/中文, light/dark) is also part of this workspace.

## Releases

> ⚠️ **Every artifact is published for AARCH64 (arm64-v8a) only** — no x86 / x86_64 / arm32 builds.

### Android · Maven Central (`io.github.fellow99:*`)

| Package | Version | OS | Arch | Link |
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

### Android · CLI tools (GitHub Release)

| Package | Release | OS | Arch | Link |
|---|---|---|---|---|
| `node-cli` | v26.13.0 | Android | arm64-v8a | https://github.com/fellow99/aosp-node-cli/releases/tag/v26.13.0 |
| `python-cli` | v3.14.10 | Android | arm64-v8a | https://github.com/fellow99/aosp-python-cli/releases/tag/v3.14.10 |
| `git-cli` | v2.56.0 | Android | arm64-v8a | https://github.com/fellow99/aosp-git-cli/releases/tag/v2.56.0 |
| `uv-cli` | v0.13.6 | Android | arm64-v8a | https://github.com/fellow99/aosp-uv-cli/releases/tag/v0.13.6 |
| `pnpm-cli` | v12.16.1 | Android | arm64-v8a | https://github.com/fellow99/aosp-pnpm-cli/releases/tag/v12.16.1 |

### HarmonyOS · ohpm (`@fellow99/*`)

| Package | Version | OS | Arch | Link |
|---|---|---|---|---|
| `node-bridge` | 0.1.0 | OpenHarmony | arm64-v8a | https://ohpm.openharmony.cn/#/cn/detail/@fellow99%2Fnode-bridge/v/0.1.0 |
| `node-bridge` | 0.2.0 | OpenHarmony | arm64-v8a | https://ohpm.openharmony.cn/#/cn/detail/@fellow99%2Fnode-bridge/v/0.2.0 |
| `node-runtime` | 24.2.0 | OpenHarmony | arm64-v8a | https://ohpm.openharmony.cn/#/cn/detail/@fellow99%2Fnode-runtime/v/24.2.0 |
| `node-runtime` | 22.23.4 | OpenHarmony | arm64-v8a | https://ohpm.openharmony.cn/#/cn/detail/@fellow99%2Fnode-runtime/v/22.23.4 |
| `node-runtime` | 24.22.1 | OpenHarmony | arm64-v8a | https://ohpm.openharmony.cn/#/cn/detail/@fellow99%2Fnode-runtime/v/24.22.1 |
| `node-runtime` | 26.13.0 | OpenHarmony | arm64-v8a | https://ohpm.openharmony.cn/#/cn/detail/@fellow99%2Fnode-runtime/v/26.13.0 |

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
gates and provenance. The CLI tools follow the same discipline with their own `*-cli.lock.json`.

## Quick start

### Android — in-process Node (Maven Central)

```kotlin
// build.gradle.kts
dependencies {
    implementation("io.github.fellow99:node-bridge:0.2.0")
    runtimeOnly("io.github.fellow99:node-runtime:26.13.0") // engine, pinned to Node
    // optional: cross-compiled node-pty addon, same version as the runtime
    runtimeOnly("io.github.fellow99:node-pty:26.13.0")
}
```

```kotlin
// Register host capabilities FIRST, then start (order matters).
val rt = NodeRuntime.getInstance(context)
rt.host.register("fs.pickFolder") { """{"ok":true,"path":"/sdcard/Download"}""" }
rt.start(NodeRuntimeOptions(entry = "bootstrap.js", appEntry = "selfcheck.js"))
rt.state.collect { state -> /* Idle → Starting → Running → Exited/Failed */ }
```

### Android — CLI tools (GitHub Release)

```kotlin
// build.gradle.kts — drop the tarball into your repo, or follow per-repo install docs
dependencies {
    runtimeOnly("io.github.fellow99:node-runtime:26.13.0") // provides libc++_shared.so
}
```

Each `*-cli` ships an exec-form `lib*_cli.so` plus assets. Point `PATH` at
`nativeLibraryDir` (or a symlink farm under `$HOME/bin`) and launch the tool through a
pseudoterminal — e.g. `pty.spawn(nativeDir + "/libnode_cli.so", ["-e", "console.log(1)"])`.
`useLegacyPackaging=true` is required so the binaries land on disk. Some tools stack:
python + uv give a full Python toolchain, pnpm needs `node` on `PATH` (aosp-node-cli).

### HarmonyOS (ohpm)

```bash
ohpm install @fellow99/node-bridge
ohpm install @fellow99/node-runtime@26.13.0
```

```ts
import { NodeRuntime } from '@fellow99/node-bridge';
NodeRuntime.getInstance().start(ctx, { scriptPath: '<sandbox>/payload/bootstrap.js' });
```

V8 JIT additionally requires the `ohos.permission.kernel.ALLOW_WRITABLE_CODE_MEMORY` ACL declared in
your `module.json5`; `--jitless` is the fallback (WebAssembly unavailable).

## Cloning

All ten repositories plus the website are wired in as git submodules:

```bash
git clone --recurse-submodules https://github.com/fellow99/node-bridge-workspace.git
# or, after a plain clone:
git submodule update --init --recursive
```

```
node-bridge-workspace/
├── aosp-node-bridge/      # Android bridge — Kotlin API + JNI/NAPI bootstrap (Maven)
├── aosp-node-runtime/     # Android engine — cross-compiled libnode.so AAR (Maven)
├── aosp-node-pty/         # Android node-pty addon — cross-compiled libpty_node.so AAR (Maven)
├── aosp-node-cli/         # Android `node` CLI — exec-form libnode_cli.so + npm/npx/corepack (Release)
├── aosp-python-cli/       # Android `python` CLI — PEP 738 libpython_cli.so + stdlib (Release)
├── aosp-git-cli/          # Android `git` CLI — libgit_cli.so, https via static curl/openssl (Release)
├── aosp-uv-cli/           # Android `uv` CLI — Rust libuv_cli.so (Release)
├── aosp-pnpm-cli/         # Android `pnpm` CLI — Rust libpnpm_cli.so (Release)
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
- **CLI tools are device binaries, not SDKs** — several platform limits are real and reported,
  not hidden: python sdist cannot be compiled on device; `uv python install` is rejected (C2 red
  line); exec requires `useLegacyPackaging=true`; pnpm lifecycle needs a `node` on `PATH`.

## License

All repositories are [MIT](LICENSE). The bundled Node.js engine is covered by the upstream
Node.js MIT license (including V8/OpenSSL/ICU third-party notices, shipped with each runtime
package); the CLI AARs ship their own THIRD-PARTY-NOTICES.md (e.g. git GPLv2 — source tarball
is included with the release archive).
