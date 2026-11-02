# AOSP · node-pty 调用常用程序（node/python/git 等）运行方案分析

> 状态：**技术途径分析（立项前输入，非实现基线）· §9 制品线与 submodule 划分已拍板（2026-10-30）**
> 日期：2026-10-30 · 平台：Android（arm64-v8a，minSdk 24）
> 定位：回答一个问题——在安卓环境下，进程内 Node 运行时（[`aosp-node-runtime`](../aosp-node-runtime/README_zh.md)）
> + node-pty 原生件（[`aosp-node-pty`](../aosp-node-pty/README_zh.md)）之外，如何让 PTY 终端里
> **还能调用 node（含 npm/pnpm）、python（含 pip/uv）、git 等常用程序**。
> 阅读顺序：先 §1 问题本质，再 §2 平台硬约束（决定一切），后按需查 §3–§9。
> 关联文档：[`aosp-node-bridge/docs/000-工程规划.md`](../aosp-node-bridge/docs/000-工程规划.md)（§4.9 路线取舍）、
> [`aosp-node-pty/docs/000-工程规划.md`](../aosp-node-pty/docs/000-工程规划.md)（§7.4 运行期事实）。

---

## 1. 问题本质

node-pty 本身**不挑程序**——`forkpty` 之后 `exec` 的是任意可执行文件
（`unlockpt/grantpt → setsid → TIOCSCTTY → exec <目标>`，由 node-pty 完成，
见 pty 规划 §7.4）。真正的问题是：

> **原生 Android 上根本不存在 node、python、git 这些程序**——`/system/bin` 里只有
> `mksh`（系统 shell）和 `toybox`（busybox 式核心工具集，无 xz/make/gcc）。
> 所以要回答的三件事是：
> **① 可执行文件从哪来；② 放在哪才能被 exec；③ 生态依赖（npm/pip 装的东西）怎么办。**

### 1.1 生态级限制（贯穿全篇，必须如实上报）

**设备上没有编译器**：无 gcc/make，toybox 不含完整工具链。因此——

- npm 装**带 C/C++ 原生扩展的包**：设备上无法 `node-gyp`（这正是 pty 仓立项理由之一）；
- pip 装**源码分发（sdist）的包**：无法本地编译；
- 唯一可行域：**纯 JS / 纯 Python 的包** + **随 APK 预置的原生件**。

这不是可以"修好"的问题，而是能力边界，应写入消费方文档的能力标志，绝不静默降级。

---

## 2. 平台硬约束（决定方案形态的 6 条）

| # | 约束 | 依据 / 影响 |
|---|---|---|
| **C1** | **targetSdk≥29 的 SELinux 分域**：应用可写数据目录（`filesDir`）**不可 `exec`、不可 `dlopen`**（`neverallow execute:app_data_file`） | pty 规划 §3.1 已记录。⇒ 工具二进制**只能放进 APK 的 `nativeLibraryDir`**，且必须以 `lib*.so` 命名 |
| **C2** | **运行期下载可执行代码不可行**：既是 Play 政策红线，也违反本工作区既定纪律（"不得运行期解包 `.so` 再加载"，bridge 规划 §4.10） | ⇒ **所有工具必须随 APK 预置**；"apt/pkg 式运行时安装可执行件"这条路封死 |
| **C3** | **`useLegacyPackaging=false`（= `extractNativeLibs=false`）时库不落盘**：库直接从 APK mmap，`nativeLibraryDir` 可能是空目录 | ⚠️ 与 pty README 现有表述（"APK 解包后二者同在 nativeLibraryDir"）有出入，**必须真机核验**（§7 探针 1）。**exec 型工具要求 `useLegacyPackaging=true`**，否则 `execve` 直接 ENOENT |
| **C4** | **exec 型二进制是另一种 ELF 形态**：PIE + `PT_INTERP=/system/bin/linker64`（合法且**必需**） | 现有"无 PT_INTERP"门禁只适用于 dlopen 库；工具线需要**新的门禁 profile**（§4.2），不能照搬 `pty.lock.json` 门禁集 |
| **C5** | **JIT/execmem 与 targetSdk 联动**：CLI 子进程与宿主 App 同 SELinux 域，P0-1 的 W^X 问题同样作用于 CLI node 的 V8 JIT | 已实测 Android 15 真机 JIT 可用（bridge `docs/002-真机验证记录.md` R2：hotLoop 15–18ms）。新 targetSdk 档位的 execmem 收紧趋势需跟踪；保留 `--jitless` 兜底 |
| **C6** | **`node::Start` 每进程一次** 不构成障碍 | CLI 版 node 是**独立进程**，与进程内引擎互不冲突——这是 CLI 路线的天然优势 |

补充两条次级约束：

- **exec 出的子进程走默认 linker namespace**：不会自动看到 App 的库目录。
  依赖 `libc++_shared.so` 的工具须经 `LD_LIBRARY_PATH=<nativeLibraryDir>` 注入，或静态固化 C++（`-static-libstdc++`）。
- **脚本文件的 exec 语义**：`filesDir` 下的脚本不可直接 exec（C1）；`nativeLibraryDir` 下的
  `lib*.so` 壳脚本（sh 脚本改扩展名）**可以**经 shebang 被内核正常解释执行（§3.1 的命令入口技巧）。

---

## 3. 逐工具技术途径

### 3.1 node（含 npm / pnpm / npx / corepack）

| 方案 | 做法 | 先例 / 可行性 | 代价 |
|---|---|---|---|
| **A1：交叉编译 node CLI（推荐主线）** | 与 aosp-node-runtime **同一棵源码树**，去掉 `--shared`（`scripts/patch-node-source.py` 当前正是加 `--shared`），默认构建目标即产出 `node` PIE 可执行；改名 `libnode_cli.so`（**不能叫 `libnode.so`**——避免与引擎 soname 同名冲突，同目录两文件不可重名） | 与 runtime 同一条 CI 流水线即可复现；bridge 规划 §2 调研中的 Magisk `node_on_android` 已证明"官方 node PIE 二进制在 Android 可跑" | 体积 ≈60–90MB（`--with-intl=none` + strip 后）；引擎升版时同步重编 |
| **A2：socket 转发复用进程内引擎（备选）** | 不带 node 二进制；进程内 Node 起一个 Unix socket 服务，nativeLibraryDir 放一个几 KB 的转发器，把 argv/env/stdio 转进引擎执行 | 省下 ≈90MB | 终端 REPL 与业务引擎**共享全局状态与生命周期**（一次崩溃全挂、`node::Start` 不可重启）；交互语义需自建；复杂度高。仅当体积压力实测不可接受时评估 |

**npm / npx / corepack**：全是纯 JS，**不需要 exec 位**——

- node 源码树自带 npm；corepack 为纯 JS。

> ⚠️ **pnpm 例外（2026-10-30 拍板）**：pnpm 自 12 起为 **Rust 独立二进制**，不再适用"纯 JS 随包"方案——Rust 版走独立制品线 `aosp-pnpm-cli`（决策与遗留探针见 §9.3）；本节的 JS 分发仅作为 **M2 基线**（corepack 或钉版 JS 树）兜底。
- 分发形态：assets → 首启幂等物化到 `filesDir`（**复用 bridge 现有 payload 物化机制**，assets 不是文件系统路径，必须物化）。
- 命令入口："**壳脚本作 lib**"技巧，例如 `libnpm.so` 的文件内容就是：

```sh
#!/system/bin/sh
exec "$NODE_NATIVE_DIR/libnode_cli.so" "$NODE_TOOLS_DIR/npm/bin/npm-cli.js" "$@"
```

  放进 jniLibs 一样会被解包到 `nativeLibraryDir` 并可执行（内核只看 shebang；路径经 shell profile 的环境变量注入）。

**已知限制（必须写进消费方文档）**：`npm i -g` 装出来的 CLI 脚本落在 `filesDir`——
受 C1 不能直接 exec；且其 `#!/usr/bin/env node` shebang 在 Android 上无 `/usr/bin/env` 可寻。
只能：经 `npx` 调用、以 `node <路径>` 显式调用、或安装期做 shebang 改写（Termux 的做法）。

### 3.2 python（含 pip / uv）

| 项 | 途径 | 说明 |
|---|---|---|
| **CPython 本体** | **CPython 3.13+ 的 PEP 738 官方 Android 支持**：`./configure --host=aarch64-linux-android`，NDK 交叉编译 | 官方支持（不像 node 是 unsupported）；Termux / Chaquopy / pydroid 均为成熟先例。体积 ≈15–30MB |
| **关键决策：扩展模块静态化** | 配置为**静态 libpython + 全静态扩展模块**（`Modules/Setup.local` 的 `*static*`），只把 `Lib/` 纯 py stdlib 经 assets→filesDir 物化 | 规避 C1 的唯一干净办法：动态扩展 `.so` 若放 filesDir 会因不可 `dlopen` 全废。静态化后 = 一个 python 可执行 + 纯数据 stdlib，全程不触碰 dlopen 禁令。（Chaquopy 的动态方案需自定义 importer + jniLibs 特殊命名，复杂一个量级） |
| **ssl** | OpenSSL 交叉编译（Termux 先例）静态链入 `_ssl`；**CA bundle 必须随包**（Android 无 `/etc/ssl/certs`），经 `SSL_CERT_FILE` 注入 | 否则 pip 连不上 PyPI |
| **pip** | `ensurepip` 随包（CPython 源码内嵌 wheel），纯 py，无 exec 需求 | |
| **uv** | Rust `aarch64-linux-android` target 交叉编译（cargo-ndk）；TLS 走 rustls 自带根证书；依赖 ring/zstd 等 NDK 下可编 | 无官方 Android 产物，按本工作区"CI 交叉编译 + lock + 门禁"模式自建。体积 ≈25–40MB |

限制同 §1.1：pip 装 sdist 包无法本地编译；纯 py wheel（含 uv 管理的 python 环境）可用。

### 3.3 git

- **NDK 交叉编译真实 git**，Termux 是完整先例。配置要点：
  `NO_TCLTK`、`NO_GETTEXT`、`NO_ICONV`（bionic 无 iconv）、`NO_REGEX`（用自带 compat/regex）、
  zlib 静态；https 传输需 **libcurl + OpenSSL 静态链入**（体积增量 ≈5–8MB）。产物本体 ≈2–5MB。
- 环境注入：`GIT_CONFIG_GLOBAL=$HOME/.gitconfig`、`GIT_TEMPLATE_DIR` 指向物化后的模板目录。
- ssh 远程（`git@…`）需另带 dropbear/openssh——建议二期。
- 备选不推荐：isomorphic-git（纯 JS，可在进程内引擎跑，但无法以 `git` 命令形态出现在 PTY，协议覆盖不全）。

### 3.4 bash（顺带）

`/system/bin/sh`（mksh）已可用（pty 探针判据即它）。真 bash 也很容易：
configure 类项目里 bash 是最好交叉编的一档，同样以 `libbash.so` 随包。

---

## 4. 公共基础设施（无论选哪些工具都必须建）

### 4.1 新制品线（镜像 pty 的"伴生件"模式立项）

**按工具分仓（已拍板，仓库清单与命名约定见 §9）**：每工具一条
`fetch-source（pin + sha256）→ 交叉编译 → 门禁 → GitHub Release → Maven` 流水线，
与 runtime / pty 完全同构（fetch 不改源码、lock 唯一事实源、二进制不入库、CI 产物人工回写）。

版本纪律（已定，详见 §9.2）：

- `node-cli`：恒等于 Node 版本（与 runtime/pty 配对，镜像 pty 纪律）；
- python / git / uv / pnpm：跟随各自上游版本，独立 semver（与 Node ABI 无关，无 NODE_MODULE_VERSION 约束）。

### 4.2 exec 型门禁 profile（新，区别于 dlopen 库门禁）

| 门禁 | 判据 | 与 pty 门禁的差异 |
|---|---|---|
| ELF 形态 | `ET_DYN`（PIE） | 同 |
| 解释器 | **`PT_INTERP = /system/bin/linker64`（允许且必需）** | **相反**：pty 门禁要求无 PT_INTERP（dlopen 库） |
| 依赖白名单 | `DT_NEEDED` 仅限系统库（libc/libm/libdl/libz）+ 明确登记的随包库 | 新增"白名单"概念 |
| 页对齐 | 16KB（`-Wl,-z,max-page-size=16384`） | 同 |
| 出处 | sha256 / size / 源版本 / NDK / flags | 同 |
| 命名 | `lib*.so` 前缀强制；**严禁与 `libnode.so` 同名** | 新增 |

### 4.3 环境引导层（bridge 载荷或 Kotlin 侧）

- **路径与环境变量**：`HOME/TMPDIR` → filesDir；`PATH=$NLDIR:$HOME/bin:/system/bin`；
  `LD_LIBRARY_PATH=$NLDIR`；`SSL_CERT_FILE`；`GIT_CONFIG_GLOBAL`；`PYTHONHOME/PYTHONPATH` 等。
- **assets→filesDir 幂等物化**：扩展到工具数据（npm 树、python stdlib、git 模板、CA bundle）——
  复用 bridge 现有 payload 物化机制与 marker 纪律。
- **命令符号链接农场**：在 `$HOME/bin` 建 `node → $NLDIR/libnode_cli.so` 等 symlink；
  exec 符号链接时 SELinux 只查目标文件标签（`apk_data_file` 域，放行），路径名即为命令名。
- **PTY 会话管理**：`pty.spawn(绝对路径, args, { env })`；消费方先跑现有
  `/system/bin/sh` 回程探针，再逐工具探针。

---

## 5. 整体路线对比与推荐

| 路线 | 形态 | 优点 | 缺点 | 判断 |
|---|---|---|---|---|
| **R1：随包 CLI（推荐主线）** | 全部工具交叉编译后以 `lib*.so` 进 AAR，exec 于 nativeLibraryDir | 真终端语义（每命令独立进程、独立生命周期）；完全复用现有 CI/lock/门禁纪律；无运行期下载 | APK 体积（node ≈60–90MB、uv ≈30MB、python+stdlib ≈30MB、git ≈10MB）；每工具一条构建线 | ✅ **主线**。体积可用 Play 动态特性模块按需分发缓解（split APK 的 lib 目录同样合规可 exec） |
| **R2：CLI 薄 + 复用进程内引擎** | 只带 git/python 等；node 走 socket 转发（§3.1 A2） | 省 ≈90MB | 共享状态/生命周期、实现复杂、REPL 体验打折 | ⏸ 备选，体积压力实测不可接受时再评估 |
| **R3：proot + 完整 Linux 发行版** | glibc rootfs（Ubuntu/Alpine）+ proot | 唯一能让 glibc 生态（含 npm/pip 装的可执行二进制包）完整工作的形态 | **C1 封死**：rootfs 二进制在 filesDir，targetSdk≥29 不可 exec；仅 targetSdk≤28 侧载或 root 可行；与本工作区"合规、不静默降级"纪律冲突 | ❌ 仅登记，不推荐（Termux 也是靠 targetSdk 28 才做到） |
| **R4：跨 App 依赖 Termux（RUN_COMMAND）** | 用户另装 Termux，跨 App 调用 | 零体积 | 非自包含、依赖第三方 App、体验断裂 | ❌ 不符合本工作区"进程内自足"定位 |

---

## 6. 与既有规划的关系（立项时须显式衔接）

[aosp-node-bridge 规划 §4.9 路线 C](../aosp-node-bridge/docs/000-工程规划.md)
当时否决过"子进程 exec 自带 node 二进制"——但那是在**"选谁当引擎"**的语境下否决的
（理由：体积翻倍、FAQ 权限问题、Play 政策）。本方案语境不同，三点澄清：

1. **PTY 天然就是多进程形态**——终端里每个命令本来就该是独立进程。
   R1 不与"进程内引擎"主线竞争，而是其**伴生能力**（正如 node-pty 之于 node-runtime）；
2. 否决理由中的"noexec"针对**数据目录**；`nativeLibraryDir` 不受影响（C1 的准确边界）；
3. "FAQ 说 spawn 有权限问题"在 pty 工程中已被事实推进：`/system/bin/sh` spawn 可用是探针判据
   （pty 规划 §7.4）；剩余未决项只是"exec **自带**二进制"这一档（原 P0-7 的延伸）。

⇒ 立项文档需写明这层关系，避免与 §4.9-C 的历史结论打架；建议同步在 bridge 规划 §4.9-C
补一行"（注：该否决针对引擎选型；终端工具场景见 aosp+node+python+git 运行方案）"。

---

## 7. 真机验证清单（P0 级探针，立项后第一件事）

| # | 探针 | 判据 | 对应约束 |
|---|---|---|---|
| 1 | **`nativeLibraryDir` 落盘核验**：消费方 App 分别用 `useLegacyPackaging=false/true` 构建，`ls nativeLibraryDir` + `execve` | false 时文件是否存在；true 时 exec 是否成功 | **C3（全方案前置）**；顺带修正 pty README 表述 |
| 2 | **exec 自带二进制**：`libhello.so`（hello-world PIE）exec 回显 | stdout 收到回显 | C1/C4 |
| 3 | **symlink exec**：`$HOME/bin/node → $NLDIR/libnode_cli.so` | 经 symlink 执行成功 | §4.3 |
| 4 | **壳脚本 exec**：`libnpm.so`（sh 脚本）shebang 运行 | 脚本逻辑执行成功 | §3.1 |
| 5 | **PTY × CLI**：`pty.spawn($NLDIR/libnode_cli.so, ['-e','console.log(1)'])` | PTY 回程收到输出、exit/close 事件齐全 | C4 |
| 6 | **fork/clone 于 App 域** | 记录性探针（pty 已隐含验证） | — |
| 7 | **CLI node 的 JIT**（Android 15 真机） | hotLoop 时间量级 + `--jitless` 兜底可用 | C5 |
| 8 | **16KB 页设备**（Pixel 16KB 模拟器） | 全部 exec 件加载与执行正常 | C4 |
| 9 | **网络面**：随包 CA bundle 下 `git ls-remote` / pip index | TLS 握手成功 | §3.2/§3.3 |

---

## 8. 落地步骤建议

| 里程碑 | 内容 | 出口判据 |
|---|---|---|
| **M0 立项** | 五仓 scaffold（§9.2）+ exec 门禁 profile + lock 纪律；写明与 bridge §4.9-C 的关系 | 规划文档评审通过 |
| **M1 最小链路** | 先拿 **git**（最小、依赖少、先例最硬）跑通"fetch → 交叉编译 → 门禁 → Release → AAR → exec → PTY 回程"全流程；顺带产出 §7 探针 1 的结论 | 真机终端里 `git --version` 回程成功 |
| **M2 node 线** | node CLI + npm/corepack 壳 + 环境引导层（物化 / profile / symlink 农场）；pnpm 先走 corepack/JS 基线（§9.3） | `node -v`、`npm -v`、`pnpm -v`（JS 基线）在终端可用 |
| **M3 python 线** | python（PEP 738 + 静态扩展）+ pip（ensurepip + CA）；uv（Rust 交叉编译线） | `pip install <纯py包>` 成功；`uv venv` 可用 |
| **M3.5 pnpm 线** | `aosp-pnpm-cli`（Rust 独立二进制；先跑 §9.3 的 ⬜ 依赖可编性试编） | 终端里 `pnpm install`（Rust 版 12+）真实可用 |
| **M4 治理与扩展** | 体积治理（动态特性分发）、bash/openssh 等扩展、跟随机制（runtime 升版 → node CLI 同版重编）、`aosp-node-shell` 立项（M1 后条件触发，见 §9.4） | 跟随 runbook 成立 |

---

## 9. 制品线与 submodule 划分（已拍板，2026-10-30）

### 9.1 命名语义约定

| 后缀 | 含义 | 制品形态 | 加载语义 | 立项条件 |
|---|---|---|---|---|
| **`-runtime`** | 进程内 dlopen 的引擎共享库（先例：`libnode.so`） | ET_DYN、无 PT_INTERP、16KB 门禁 | 被桥 `dlopen`，addon 以 `DT_NEEDED` 挂靠 | **有桥消费方才建** |
| **`-cli`** | exec 型命令行可执行 | PIE + `PT_INTERP=/system/bin/linker64`、静态依赖、16KB | 被 `pty.spawn`/`execve` 执行 | 终端工具需求即可建 |

- 仓名 = `aosp-<工具>-<角色>`；坐标 = `io.github.fellow99:<仓名去 aosp- 前缀>:<版本>`；
- 二进制名 = `lib<工具>_cli.so`（exec 型统一带 `_cli` 后缀）；命名禁区：`libnode.so`（引擎占用）、`libuv.so`（与 libuv 库混淆）；
- **一仓 = 一版本线 = 一 lock = 一条 tag 流**。

### 9.2 首发五仓

| # | submodule | Maven 坐标 | 版本规则 | AAR 内容 | 构建线 | 里程碑 |
|---|---|---|---|---|---|---|
| 1 | `aosp-node-cli` | `io.github.fellow99:node-cli:<Node版本>` | 恒等于 Node 版本（与 runtime/pty 配对） | jniLibs：`libnode_cli.so`；assets：npm/corepack 树（pnpm JS 钉版可选） | Node 源码树去掉 `--shared` | M2 |
| 2 | `aosp-python-cli` | `io.github.fellow99:python-cli:<CPython版本>` | = CPython 版本 | jniLibs：`libpython_cli.so`（静态扩展）；assets：stdlib + ensurepip + CA bundle | PEP 738 官方 Android 构建 | M3 |
| 3 | `aosp-git-cli` | `io.github.fellow99:git-cli:<git版本>` | 跟 git 上游，独立 semver | jniLibs：`libgit_cli.so`（静态链 zlib/curl/openssl） | git + vendored deps | M1 |
| 4 | `aosp-uv-cli` | `io.github.fellow99:uv-cli:<uv版本>` | 跟 uv 上游，独立 semver | jniLibs：`libuv_cli.so` | Rust `aarch64-linux-android`（cargo-ndk） | M3 |
| 5 | `aosp-pnpm-cli` | `io.github.fellow99:pnpm-cli:<pnpm版本>` | 跟 pnpm 上游，独立 semver | jniLibs：`libpnpm_cli.so` | Rust 独立二进制（pnpm 12+） | M3.5 |

### 9.3 `aosp-pnpm-cli` 决策与遗留探针

- **决策（2026-10-30）**：建仓，按**纯 Rust 独立二进制**对待（pnpm 12 起）。
- **运行期依赖**：`node` 必须在 PATH（package lifecycle scripts 是 JS，由 node 执行）⇒ 对 `aosp-node-cli` 有运行时依赖，不是孤岛；其 README 必须声明。
- **遗留 ⬜ 探针**（不阻塞建仓，阻塞 M3.5 出口）：
  1. 上游是否发布 `android-arm64` 平台二进制（esbuild/biome 式 optionalDependencies 分发大概率没有）——无论如何 CI 自建（C2 纪律），不下载上游二进制；
  2. Rust 依赖（napi-rs？ring？tokio？）在 `aarch64-linux-android` 下的可编性试编；
  3. 若探针推翻前提（实际是 NAPI 混合形态、由 node 加载）：**冻结本仓计划并重新评审**——彼时应按 pty 式伴生件处理（NODE_MODULE_VERSION 配对 + `node-cli` 进程内 napi 符号解析问题），属另一条制品线。
- M2 基线不受影响：终端 `pnpm` 命令先经 corepack / JS 壳脚本可用。

### 9.4 延后 / 条件仓（现在不建）

| 仓 | 触发条件 | 说明 |
|---|---|---|
| `aosp-node-shell` | **M1（git 最小链路）跑通后、M2 前立项** | 会话/引导层：pty 加载垫片 + 环境引导（PATH/profile/物化/symlink 农场）+ 会话管理；即 pty 规划 §10-2 预留的"统一加载垫片" |
| `aosp-python-runtime` | 出现真实"进程内嵌 Python"需求（如 `aosp-python-bridge`）时 | `libpython3.x.so` 动态扩展形态，与已存在的 `python-cli` 构成 node 同款两线；版本同为 CPython 版本 |
| bash / openssh / dropbear | M4 再议 | bash 独立小仓即可；openssh 可为 git ssh 远程服务 |

### 9.5 刻意不建仓（归处已定）

| 东西 | 归处 | 原因 |
|---|---|---|
| npm / npx / corepack | `aosp-node-cli` 的 assets | 纯 JS、无 exec 位、随 node 源码树分发 |
| pnpm（JS 版） | `aosp-node-cli` assets（钉版）或 corepack | Rust 版独立成线后，JS 版仅作 M2 基线兜底 |
| pip | `aosp-python-cli`（ensurepip） | 随 CPython 源码内嵌，同一棵源码树 |
| 终端 UI / 协议封装 | 消费方 | pty 规划 §1.3 边界不变 |

### 9.6 否决方案记录

| 方案 | 否决理由 |
|---|---|
| `node-cli` 并入 `aosp-node-runtime` | 同源码树但需按非 `--shared` 二次配置（构建时间翻倍）；runtime 发布被捆绑 ≈90MB 资产、CI 双产物耦合；破坏 runtime 仓"只有引擎二进制"章程。**配对靠版本纪律，不靠同仓** |
| 单一 `aosp-node-tools` 大仓（多模块多版本线） | 破坏"一仓一版本线一 lock"；一仓多条 tag 流 + workflow 路由复杂化；镜像式 scaffold 已把单仓成本压低，大仓省不了事反而丢发布独立性 |
| 现在就建 `aosp-python-runtime` | 无 python-bridge 消费方（YAGNI）；可嵌入形态是**另一种构建配置**（动态扩展 + 自定义 importer + dlopen 门禁），与 CLI 形态（静态扩展）不可同仓 |

### 9.7 落地动作

```bash
git submodule add https://github.com/fellow99/aosp-node-cli.git
git submodule add https://github.com/fellow99/aosp-python-cli.git
git submodule add https://github.com/fellow99/aosp-git-cli.git
git submodule add https://github.com/fellow99/aosp-uv-cli.git
git submodule add https://github.com/fellow99/aosp-pnpm-cli.git
# aosp-node-shell 在 M1 结束后再 add
```

每仓镜像 `aosp-node-pty` scaffold（settings/publish/provision/fetch/lock/workflow/README×2/specs），两处差异：

1. 门禁换 **exec 型 profile**（§4.2）：`PT_INTERP=/system/bin/linker64` 允许、`DT_NEEDED` 白名单、16KB、`lib<工具>_cli.so` 命名门禁；
2. jniLibs 之外多 `assets/`（纯数据，无 ELF 门禁，只记 sha256）。

同步更新：根 README 仓库表、`.gitmodules`；发布节奏沿用 pty 同款（**GitHub Release 先行、
  Maven Central 随后**——Maven 阶段执行口径见 [`docs/发布到maven-central.md`](./发布到maven-central.md)，
  runbook §10 已登记五条 CLI 线复用）。

---

## 附：可行性先例索引

| 工具 | 先例 | 佐证点 |
|---|---|---|
| node on Android（PIE 可执行） | Magisk `node_on_android`；nodejs-mobile（同源码树、不同目标） | bridge 规划 §2 调研；本仓 runtime 交叉编译链已跑通 |
| git on Android | Termux `git` 包 | NDK 全套依赖（curl/openssl/zlib）交叉编译成熟 |
| CPython on Android | **PEP 738（3.13+ 官方支持）**；Termux；Chaquopy；pydroid | 官方平台支持 + 多家商业先例 |
| uv on Android | 无官方产物；Rust `aarch64-linux-android` 为官方 tier2 target | cargo-ndk 交叉编译常规操作 |
| pnpm on Android | pnpm 12+ 为 Rust 独立二进制（⬜ 上游 android-arm64 分发待探针核实，§9.3） | 与 uv 同走 Rust 交叉编译线 |
| exec 于 nativeLibraryDir | 大量 Play 在架应用（捆绑 ffmpeg/ssh 等） | `lib*.so` + extractNativeLibs=true 为公认合规形态 |
| proot 发行版 on Android | Termux proot-distro | 仅 targetSdk≤28 / root 可行（反面教材，见 R3） |

> 本文档只做**技术途径分析**，不构成任何实现承诺；所有体积/行为数字以真机探针与 CI 实测为准。
> 落地时按本工作区纪律：**先改规划文档，再改实现；未实测项一律标 ⬜，绝不当作已定事实引用。**
