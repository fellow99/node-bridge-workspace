# 发布到 ohpm（OpenHarmony 三方库中心仓）

> **本文是什么**：`@fellow99/node-bridge` 与 `@fellow99/node-runtime` 两个 HAR 包**发布到
> [ohpm.openharmony.cn](https://ohpm.openharmony.cn/) 的方法与可照抄的流程（runbook）。
>
> | 项 | 值 |
> |---|---|
> | 文档日期 | 2026-11-01 |
> | 覆盖仓库 | [`ohos-node-bridge`](../ohos-node-bridge)（桥接包）、[`ohos-node-runtime`](../ohos-node-runtime)（运行时包） |
> | 规则依据 | 中心仓帮助中心（`publishrequirefile` / `publishrules` / `guidename` / `createandpublish`）+ 本机 `ohpm` CLI 实测；详见 [`ohos-node-bridge/docs/提交到OpenHarmony三方库中心仓.md`](../ohos-node-bridge/docs/提交到OpenHarmony三方库中心仓.md)（规则基线、差距分析、审核口径） |
> | 工具依据 | `ohos-node-runtime/tools/{provision,verify-har,release}.mjs`（发布门禁与编排） |
>
> 本文只讲**发布面**：包形态、元数据、门禁、命令、审核与核验。架构与实现见各仓库 `README.md` 与 `docs/000-工程规划.md`。

---

## 0. 速览

### 0.1 两个包、两个仓库

| 包 | 仓库 | hvigor module（标识符 / 目录） | 产物 | 版本规则 |
|---|---|---|---|---|
| `@fellow99/node-bridge` | `ohos-node-bridge` | `library` / `library/` | `library/build/default/outputs/default/library.har` | **自身 semver**（当前 `0.2.0`） |
| `@fellow99/node-runtime` | `ohos-node-runtime` | `runtimehar` / `runtime-har/` | `runtime-har/build/default/outputs/default/runtimehar.har` | **= Node.js 版本**（当前 `24.2.0`） |

> ⚠️ 三个名字不要混：**module 标识符** `runtimehar`（给 `-p module=`）、**源码目录** `runtime-har/`、
> **HAR 文件名** `runtimehar.har`（hvigor schema 不允许中划线，故标识符去掉 `-`）。

两包由**运行期符号契约**（`dlopen` + `dlsym(node::Start)` + `napi_*`）咬合，**不做包依赖**：消费方必须显式装两个包。

### 0.2 中心仓现状（2026-11-01 实测）

用中心仓 detail 接口查（命令见 §8.2）：

| 包 | 中心仓已上架版本 | latest | 差异说明 |
|---|---|---|---|
| `@fellow99/node-bridge` | `0.1.0`、**`0.2.0`** | `0.2.0` | 两个版本均已上架；`0.2.0` 随运行时分拆后的事实源更新（详见 `library/CHANGELOG.md`） |
| `@fellow99/node-runtime` | `24.2.0`、`22.23.4`、`24.22.1`、**`26.13.0`** | `26.13.0` | 四个版本均已上架；版本 = Node 版本。`26.13.0` 为 ABI 147（Temporal 禁用）；均在 `runtime.lock.json` 锁定 SHA-256 |

> ⚠️ **只发布 AArch64（arm64-v8a）一个架构**，鸿蒙侧更只面向 `2in1`/`tablet`（不做手机、不可 x86_64）。

#### 已发布版本一览（一版本一链接，2026-11-01 实测在线）

| 包 | 版本 | 操作系统 | 架构 | 发布链接 |
|---|---|---|---|---|
| `@fellow99/node-bridge` | `0.1.0` | OpenHarmony | arm64-v8a | <https://ohpm.openharmony.cn/#/cn/detail/@fellow99%2Fnode-bridge/v/0.1.0> |
| `@fellow99/node-bridge` | `0.2.0` | OpenHarmony | arm64-v8a | <https://ohpm.openharmony.cn/#/cn/detail/@fellow99%2Fnode-bridge/v/0.2.0> |
| `@fellow99/node-runtime` | `24.2.0` | OpenHarmony | arm64-v8a | <https://ohpm.openharmony.cn/#/cn/detail/@fellow99%2Fnode-runtime/v/24.2.0> |
| `@fellow99/node-runtime` | `22.23.4` | OpenHarmony | arm64-v8a | <https://ohpm.openharmony.cn/#/cn/detail/@fellow99%2Fnode-runtime/v/22.23.4> |
| `@fellow99/node-runtime` | `24.22.1` | OpenHarmony | arm64-v8a | <https://ohpm.openharmony.cn/#/cn/detail/@fellow99%2Fnode-runtime/v/24.22.1> |
| `@fellow99/node-runtime` | `26.13.0` | OpenHarmony | arm64-v8a | <https://ohpm.openharmony.cn/#/cn/detail/@fellow99%2Fnode-runtime/v/26.13.0> |

### 0.3 三句话流程

1. **出包**：各自 `hvigorw assembleHar -p buildMode=release` → 得 `.har`（HAR 是 **gzip tar**，不是 zip）。
2. **门禁**：`node tools/verify-har.mjs <har>`（复刻服务端规则，硬失败）+ `ohpm prepublish <har>`（第二道）。
3. **发布**：`ohpm publish <har>`，**顺序不能反：先运行时包、后桥接包**；在**交互式终端**执行；成功后等人工审核上架。

---

## 1. 发布单元与版本纪律

### 1.1 版本号从哪来

| 包 | 唯一事实源 | 需要与之一致的其它位置 |
|---|---|---|
| `@fellow99/node-runtime` | `runtime-har/oh-package.json5` 的 `version` | ① `runtime.lock.json` 的 `active.node`；② `runtime-har/CHANGELOG.md` 首条 `## <version>`；③ `runtime-har/Index.ets` 的 `NODE_VERSION` / `NODE_MODULE_VERSION` / `LIB_SHA256` / `LIB_SIZE` |
| `@fellow99/node-bridge` | `library/oh-package.json5` 的 `version` | `library/CHANGELOG.md` 首条 |

### 1.2 两条硬纪律

| 纪律 | 后果 |
|---|---|
| **版本号不可重用**：同一版本发布后不能覆盖、不能重发 | 首发前必须把 §3/§6 全绿跑完；被审核驳回只能换版本号 |
| **包名不可改**：首次发布后 `packageName` 会写进包内 `module.json` | `@fellow99/node-bridge`、`@fellow99/node-runtime` 已定稿，别再改名 |

失败重试的具体选择（`-rc` 预发布 / `+build` / 改 patch）见 §9。

---

## 2. 一次性前置（只做一次）

### 2.1 账号、组织、密钥、发布码

| # | 事项 | 位置 / 命令 |
|---|---|---|
| 1 | 注册并登录中心仓 | <https://ohpm.openharmony.cn> |
| 2 | 创建**个人组织** `fellow99`（组织名必须 = scope `@fellow99`；创建者自动为成员） | 【个人中心】-【组织管理】-【新增】 |
| 3 | 生成**带口令**的 RSA 密钥（ohpm 只支持加密私钥） | `ssh-keygen -m PEM -t RSA -b 4096 -f ~/.ssh_ohpm/ohpm-key` |
| 4 | 上传**公钥**（`ohpm-key.pub` 全文） | 【个人中心】-【认证管理】-【新增】 |
| 5 | 复制**发布码** | 【个人中心】首页「复制发布码」 |

> 个人开发者**不需要**实名/企业认证：中心仓的「认证管理」就是 SSH 公钥；「企业实名认证」只在创建**企业组织**时需要。
> 带组织包要求「发包人属于该组织」——`fellow99` 是个人组织，创建者本人即成员。

### 2.2 本机 CLI 配置

配置落在 `C:\Users\<你>\.ohpm\.ohpmrc`（Linux/macOS：`~/.ohpm/.ohpmrc`）：

```bash
export DEVECO_SDK_HOME="<command-line-tools>/sdk"   # hvigor 找 SDK 用
export PATH="<command-line-tools>/bin:$PATH"        # ohpm / hvigorw / 自带 node

ohpm config set publish_registry https://ohpm.openharmony.cn/ohpm
ohpm config set publish_id   <发布码>
ohpm config set key_path     ~/.ssh_ohpm/ohpm-key
ohpm ping                    # 期望 PONG / {"success":true}
ohpm config list             # 复核上面三项
```

Windows 下把 `ohpm` 换成 `cmd //c "<CLT>\bin\ohpm.bat"`。

> ⚠️ **本机环境提示**：曾用于发布的 `D:\oh-workspace\command-line-tools` 在本机已不存在（`ohpm` 不在 `PATH`）。
> 发布前需重新装 DevEco Studio 的 **command-line-tools**（含 `hvigorw` + `ohpm` + 内置 node），
> 或改 `OHPM_BIN` / `HVIGORW_BIN` 环境变量指向新路径（`tools/release.mjs` 认这两个变量）。

### 2.3 工具链版本基线

| 工具 | 用途 | 备注 |
|---|---|---|
| `ohpm`（历史实测 6.1.2.285） | `prepublish` / `publish` / `info` | publish 只吃 `.har` / `.tgz` 文件，**不能对源码目录发** |
| `hvigorw`（DevEco command-line-tools） | `assembleHar` | 必须 `-p buildMode=release` |
| Node.js（DevEco 自带或系统） | `tools/*.mjs` | `tools/` 全为零第三方依赖脚本 |
| `ssh-keygen` / GnuPG（可选） | 生成发布密钥 | ohpm 用 RSA PEM |

---

## 3. 发布前检查清单（服务端规则的本地复刻）

中心仓的自动校验分**本地能判**和**只能在服务端判**两部分——`ohpm prepublish` **只覆盖前者**：
即使缺 `CHANGELOG.md` / `LICENSE`、README 里没有 `ohpm install`，`prepublish` 也可能打印 `succeed`。
因此 `verify-har.mjs` 把服务端规则复刻成本地硬门禁（§6.1）。

| # | 规则（服务端） | 判据 | 两包现状 |
|---|---|---|---|
| 1 | 四件套齐全 | 包内 `package/oh-package.json5`、`README.md`、`CHANGELOG.md`、`LICENSE` 都存在且非空 | ✅ 均已具备（`library/`、`runtime-har/` 根目录） |
| 2 | README 含安装命令 | 文本出现 `ohpm install <包名>` 或 `ohpm i <包名>` | ✅ 两个 README 首屏都有 |
| 3 | CHANGELOG 含当前版本号 | 文本含 `version` 字符串 | ✅（`0.2.0` / `24.2.0`） |
| 4 | `name` / `version` 合规 | 小写、`@group/name`、SemVer、无 `v` 前缀 | ✅ |
| 5 | `description` | 6–512 字符、必填 | ✅ |
| 6 | `author` | **必须是对象**：`name` + (`email` 或 `url`) 至少一个 | ✅ 两包都是 `{name,email,url}` 对象 |
| 7 | `repository` | 开源包必填，URL ≤1024 | ✅ 各指向自己仓库 |
| 8 | `license` + LICENSE 文件 | 开源包必填 | ✅ `MIT` + 文件在包内 |
| 9 | `sourceType` | 显式 `open` | ✅ |
| 10 | `main` 或 `types` 指向包内真实文件 | 打包后 hvigor 会写 `types`；缺文件直接 ERROR | ✅ `Index.d.ets` / `Index.ets` 在包内 |
| 11 | `.so` 位置 | 必须位于 `package/libs/` | ✅ `package/libs/arm64-v8a/*.so` |
| 12 | 体积 | 压缩 ≤500 MB、解压 ≤1500 MB、条目 ≤102,400、README ≤10 MB | ✅ 远低于上限 |
| 13 | 无 `.exe`、无签名材料 | 扫描包内条目 | ✅ |
| 14 | release 构建 | `metadata.debug === false` | ⚠️ **必须 `-p buildMode=release`**；debug 包会带 `debug:true` 与 source map |
| 15 | `nativeComponents` 覆盖 `libs/*.so` | 未声明只 WARN（不阻断） | 🔶 属 WARN 级，可接受 |

**打不进 HAR 的东西**（历史实测，规划发布内容时按此边界设计）：

| 放进 module 根的内容 | 是否进 HAR |
|---|---|
| `README.md` / `CHANGELOG.md` / `LICENSE` | ✅ 进（hvigor 只认这三个约定文件） |
| 其它 `.md`（如 `THIRD-PARTY-NOTICES.md`） | ❌ 不进 |
| 任意子目录（如 `docs/`） | ❌ 不进 |
| `src/main/resources/rawfile/**` | ✅ 进（**这就是「随包附带锁文件/许可清单」的唯一通道**） |

---

## 4. 出包 A：运行时包（`ohos-node-runtime`）

运行时包只装引擎：`libs/arm64-v8a/libnode.so` + `libc++_shared.so` + `Index.ets`（版本常量）+ 四件套。

```bash
cd ohos-node-runtime
export DEVECO_SDK_HOME="<command-line-tools>/sdk"
export PATH="<command-line-tools>/bin:$PATH"

# ① 校验已投放产物（七项 ELF 门禁 + SHA-256/大小 = runtime.lock.json）
node tools/provision.mjs --check

# ② 幂等投放（libnode.so 不入库；从 runtime/node-v<ver>/ 或 CI 产物投放）
node tools/provision.mjs --out runtime-har/libs/arm64-v8a

# ③ release 出包（模块标识符 runtimehar）
cmd //c "<CLT>\bin\hvigorw.bat" --mode module -p module=runtimehar@default \
  -p buildMode=release assembleHar --no-daemon

# ④ 本地硬门禁（34 项；运行时包特有大项：libnode.so 随包且 SHA/大小 = 锁值）
node tools/verify-har.mjs runtime-har/build/default/outputs/default/runtimehar.har
```

预期：`runtimehar.har` ≈ **41 MB**（本机实测 `42,977,822 B / 9 个文件`，2026-10-25 构建；解压 ≈121 MB）。
`verify-har` 通过时应打印 `✓ 34 项全过：@fellow99/node-runtime@24.2.0`，其中包专属断言为：

```text
OK   <abi>/libnode.so 随包        (package/libs/arm64-v8a/libnode.so)
OK   包版本 === lock.active.node  (24.2.0 vs v24.2.0)
OK   libnode.so 大小 = 锁值       (126,809,264 B)
OK   libnode.so SHA-256 = 锁值    (92e0ad87d9c7269b…)
WARN libs/*.so 均被 nativeComponents 声明（中心仓仅 WARN）
```

> ⚠️ 本机现存的这份 HAR 里是**自建** `libnode.so`（`92e0ad87…`），与已上架的 `24.2.0`（`3019bf5f…`）**不是同一份二进制**。
> 它只能作为**新版本号**（如真机验证通过后的 `24.22.0`/`24.22.1`）发布，不能用来「替换 24.2.0」。
七项 ELF 门禁：`ET_DYN` · `PT_INTERP=0` · `AArch64` · `DT_NEEDED 含 libc++_shared.so` · 导出 `_ZN4node5StartEiPPc` · SHA-256 匹配 · 大小+版本串匹配。

---

## 5. 出包 B：桥接包（`ohos-node-bridge`）

桥接包只装 ArkTS 字节码 + 两个引导 `.so` + 类型声明 + 四件套 + rawfile 里的事实源（`runtime.lock.json`、`THIRD-PARTY-NOTICES.md`、`compatibility.json`）。
它**绝对不能**包含 `libnode.so`（引擎由运行时包提供）。

```bash
cd ohos-node-bridge
export DEVECO_SDK_HOME="<command-line-tools>/sdk"
export PATH="<command-line-tools>/bin:$PATH"

# ① 交叉编译两个引导件（NDK clang 直调；产物落 src/main/jniLibs/arm64-v8a，不入库）
node tools/build-bridge.mjs

# ② release 出包（模块标识符 library）
cmd //c "<CLT>\bin\hvigorw.bat" --mode module -p module=library@default \
  -p buildMode=release assembleHar --no-daemon
```

预期：`library.har` ≈ **0.5 MB**（`0.1.0` 时实测 495,595 B / 18 个文件，含 rawfile 事实源；重新构建后体积会随 README/CHANGELOG 变动）。

> ⚠️ 本仓库**没有** `tools/verify-har.mjs` / `tools/release.mjs`（拆分后脚本留在了 `ohos-node-runtime`），
> 因此桥接包的门禁要么按 §6.3 手工逐条核，要么按下节方案把脚本搬过来。

---

## 6. 本地门禁：可用什么、缺什么（实测）

### 6.1 `verify-har.mjs`（唯一可靠的本地上架判据）

```bash
node tools/verify-har.mjs <har>                # 人读报告；exit 0 = 全过 / 2 = 有失败
node tools/verify-har.mjs <har> --json <path>  # 额外落机器可读报告
```

- 零第三方依赖，自己解 **gzip tar**；断言覆盖 §3 全部可本地判定项，**硬失败**（不降级为告警）。
- 两包断言集不同：运行时包额外断言「`libnode.so` 随包 + SHA/大小 = 锁值 + 包版本 = `lock.active.node`」；
  桥接包额外断言「**不得**含 `libnode.so`」。
- 位置：目前只在 `ohos-node-runtime/tools/verify-har.mjs`。

### 6.2 `release.mjs` 的现状：拆分后遗留 3 项失效断言

`ohos-node-runtime/tools/release.mjs` 是**拆分前**的「双包编排器」（`--check-version` / `--build` / `--all` / `--print-publish`），
它假设 `library/` 与 `runtime-har/` 在同一个仓库。拆分后实测：

```
$ node tools/release.mjs --check-version
① 版本一致性 … 12/12 OK（runtime 包 version = lock.active = CHANGELOG = Index.ets；已投放 .so SHA/大小 = 锁值）
② 主包版本   FAIL library/oh-package.json5 可解析
③ 随包事实源 FAIL rawfile 副本缺失：runtime.lock.json      （路径指向 runtime 仓的 library/…）
③ 随包事实源 FAIL rawfile 副本缺失：THIRD-PARTY-NOTICES.md
✗ 3 项不一致 —— 修完再构建/发布
```

⇒ **结论**：`--check-version` 的第 ① 段（运行时包）仍完全可用；第 ②③ 段是拆分前遗留，不能作为发布判据。
`--build` 里的 `hvigor library assembleHar` 与 `verify-har library.har` 同样会因找不到 `library/` 而失败。

**三种收口方案（择一，本文不擅自改代码）**：

| 方案 | 做法 | 代价 |
|---|---|---|
| A（推荐） | 给 `release.mjs` 加 `--pkg=runtime|bridge` 作用域；桥接包段改读 `ohos-node-bridge` 路径（或把 `release.mjs` + `verify-har.mjs` 复制一份到 `ohos-node-bridge/tools/`） | 需要一次脚本改造 + 自测 |
| B | 保持「一仓一脚本」：`ohos-node-runtime` 只保留运行时包逻辑，`ohos-node-bridge` 自带一份发布门禁 | 两份门禁需同步维护 |
| C（临时） | 只用 `verify-har.mjs` + §6.3 手工清单，绕过 `release.mjs` | 靠人，易漏 |

### 6.3 手工替代检查（在没有脚本的仓库里逐条核）

```bash
# 包版本
grep -n '"version"' library/oh-package.json5

# CHANGELOG 首条 = 包版本
grep -m1 -n '^## ' library/CHANGELOG.md

# README 含安装命令（服务端硬规则）
grep -n 'ohpm install @fellow99/node-bridge' library/README.md

# 包内四件套 + .so 位置 + 无签名材料（HAR 是 gzip tar）
gzip -dc library/build/default/outputs/default/library.har > /tmp/b.tar
tar -tf /tmp/b.tar | grep -E 'package/(oh-package.json5|README.md|CHANGELOG.md|LICENSE)$'
tar -tf /tmp/b.tar | grep -E '\.so$'          # 只允许 package/libs/arm64-v8a/*
tar -tf /tmp/b.tar | grep -E '\.(p12|p7b|cer)$|signing\.' && echo '有签名材料，必须排除'

# release 构建口径（metadata.debug 必须 false）
tar -xOf /tmp/b.tar package/oh-package.json5 | grep -o '"debug":[a-z]*'
```

---

## 7. 预校验与发布

### 7.1 `ohpm prepublish`（第二道门，不替代 §3）

```bash
# 以下两条分别在各自仓库根目录执行
ohpm prepublish runtime-har/build/default/outputs/default/runtimehar.har   # ohos-node-runtime
ohpm prepublish library/build/default/outputs/default/library.har          # ohos-node-bridge
```

预期两条 `prepublish … succeed`。可能出现的**不阻断**告警：

| 告警 | 成因 | 处置 |
|---|---|---|
| `the har file "library.har" contains source code …` | **仅由 `obfuscated === false` 触发**（与包内是否真有源码无关；定位到 `ohpm/lib/core/publish/util/checkAndAlarmAssetLeakage.js`） | 接受告警；在审核说明里写明「byteCodeHar，仅 `modules.abc` + `.d.ets`」 |
| `nativeComponents` 为空/未覆盖 `libs/*.so` | 运行时包/桥接包的声明不全 | 仅为 WARN，服务端不拦 |

### 7.2 `ohpm publish`（**顺序 + 交互式终端**）

```bash
# ① 运行时包先上架（桥接包 README 指向它的版本号）
ohpm publish runtime-har/build/default/outputs/default/runtimehar.har

# ② 桥接包
ohpm publish library/build/default/outputs/default/library.har
```

执行时会提示：

```text
what is your passphrase of the private key:      ← 输入生成密钥时设的口令（不回显）
```

成功后各生成一张**上架审核单**（【个人中心】-【消息】/【Package】跟踪），审核通过才对外可见。

### 7.3 三个必踩坑

| 坑 | 现象 | 根因 | 处置 |
|---|---|---|---|
| **不要加 `-t latest`** | `00623001 Tag Invalid Error: … cannot be "latest"` | 服务端把 `latest` 当保留字 | 去掉 `-t`，用默认 tag |
| **不要在没有 TTY 的环境发** | 约 190 s 后 `FATAL ERROR: … JavaScript heap out of memory`（~4 GB） | 不是包太大：读不到口令 → 反复 `Start getting passphrase from CLI` → 内存耗尽 | ① 在 CMD/PowerShell/Windows Terminal 手动发；② 或 `ohpm config set key_passphrase <口令>`（明文存在 `~/.ohpmrc`，用完 `ohpm config delete key_passphrase`） |
| **别指望 `prepublish` 拦住缺件** | 缺 `CHANGELOG`/`LICENSE`/README 无安装命令也 `succeed` | 本地校验未实现这些规则 | 以 `verify-har.mjs` 为准（§6.1） |

其它可用参数（`ohpm publish --help` 实测）：`--publish_registry`、`--publish_id`、`--key_path`、`--source_type`、
`--strict_ssl`、`--compatibility_log_level`、`--disallow_nested_package`、`--ensure_dependency_include`。

---

## 8. 审核、上架与核验

### 8.1 审核

- 中心仓**按包审核**，两个包各出一张审核单、各可能被问询；不要指望同时通过。
- 本库形态的**问询风险点**与建议材料（详细版见 `ohos-node-bridge/docs/提交到OpenHarmony三方库中心仓.md` §6）：
  「包内含 JS 引擎 / `dlopen` 动态加载 / 需要受限 ACL 开 JIT / 第三方许可 / 权限最小化」。
  运行时包尤其要能**单独站住**：说明「这是 Node.js vXX 的 arm64-v8a 共享库封装 + 来源 URL + SHA-256 + Node.js LICENSE 原文随包」。

### 8.2 上架核验（三条独立判据）

```bash
# ① 中心仓 detail 接口：已上架版本列表 + dist-tags
curl -s https://ohpm.openharmony.cn/ohpm/@fellow99/node-runtime
curl -s https://ohpm.openharmony.cn/ohpm/@fellow99/node-bridge

# ② 命令行查包
ohpm info @fellow99/node-runtime
ohpm info @fellow99/node-bridge

# ③ 干净目录真实安装（最强判据）
mkdir /tmp/ohpm-verify && cd /tmp/ohpm-verify
ohpm init
ohpm install @fellow99/node-bridge @fellow99/node-runtime@24.2.0
# 复核引擎二进制与锁值一致
sha256sum oh_modules/@fellow99/node-runtime/libs/arm64-v8a/libnode.so
```

期望 `sha256` = `runtime.lock.json` 的 `runtimes[active].sha256`（`active` = `v24.2.0` → `92e0ad87…037b`）。
⚠️ 注意两个「24.2.0」不是同一份二进制：**已上架** OHPM 包内是早期 electerm 预编译（历史记录 SHA-256 `3019bf5f…51a1`），
而**仓库 HEAD/lock** 已是自建产物（`92e0ad87…037b`）。所以对齐时要以「中心仓实际下发的 SHA」为准，而不是直接读 lock。
备用候选：lock 里的 `v24.22.1`（`c3aad438…`、`114,488,552 B`）尚未在 OHPM 发布，README 标注「待真机验证」。

### 8.3 上架之后

| 事项 | 说明 |
|---|---|
| 只改文档 | README/CHANGELOG 有**单独更新入口**（可只传 md 或 md+图片 zip），不必重发 HAR；仍需审核，且同一包同一版本同时只允许 1 个审核中申请 |
| 打 tag | `git tag v<版本>` 并推送（两仓各自打） |
| 记录 | CHANGELOG 补发布日期；（可选）把发布结果回写 lock/文档 |
| 消费方回归 | 装两包 → `assembleHap` → HAP 内 `libs/arm64-v8a/libnode.so` 存在且 SHA = 锁值 → 真机 `/facts` 200 |

---

## 9. 版本不可重用 ⇒ 失败重试策略

运行时包版本被 Node 版本锁死，而 ohpm 版本发布后**不能覆盖**。若 `24.2.0` 被驳回：

| 选择 | 后果 |
|---|---|
| `24.2.0-1`（预发布标签） | 合法、仍能与 Node 版本对应；排序低于 `24.2.0`，消费者需显式指定 |
| `24.2.1` | 会被误读成「Node 24.2.1」，**不推荐** |
| 内容改完后仍发 `24.2.0` | ❌ 不可能（版本已占用） |

⇒ 首发前必须把 §3 / §6 / §7.1 全绿跑完。桥接包是自身 semver，试错成本低（改 `0.2.1` 即可）。

**回滚成本**：全量下架后有 **24 小时冷却**；版本号不可重用。

---

## 10. 常见坑速查

| 现象 | 根因 | 处置 |
|---|---|---|
| `assembleHar` 找不到 `library/` | 在 `ohos-node-runtime` 跑了拆分前的 `release.mjs --build` | 按 §6.2 处理；出包命令分开跑（§4 / §5） |
| `-p module=runtime-har@default` 报 schema 错 | module 标识符不允许中划线 | 用 `runtimehar` |
| HAR 里缺 `CHANGELOG` / `LICENSE` | 文件没放在 **module 根**（放仓库根或 `docs/` 都不进包） | 放到 `library/` / `runtime-har/` 根 |
| 想随包带 `runtime.lock.json` / 许可清单 | 任意子目录不会被收集 | 放 `src/main/resources/rawfile/...` |
| 卡在口令提示直到 OOM | 无 TTY（CI/重定向） | 交互式终端，或临时 `key_passphrase` |
| `Tag Invalid … cannot be "latest"` | 手工 `-t latest` | 不传 `-t` |
| publish 成功但搜不到 | 审核中（曾出现数小时延迟） | 看【个人中心】审核单；`curl` detail 接口 |
| 桥接包体积异常变大 | 用了 debug 构建 / 误带 `libnode.so` | `-p buildMode=release`；`verify-har` 会硬失败 |
| `libc++_shared.so` 出现两份 | hvigor 对 native 模块**自动补入**，HAR 层无法排除 | 已实测：HAP 内按路径去重只保留一份，无需处理 |

---

## 11. 一页速查（TL;DR）

```bash
# ── 0) 前置（一次性）──────────────────────────────────────────
ohpm config set publish_registry https://ohpm.openharmony.cn/ohpm
ohpm config set publish_id <发布码>; ohpm config set key_path <私钥>; ohpm ping

# ── 1) 运行时包 ──────────────────────────────────────────────
cd ohos-node-runtime
node tools/provision.mjs --check
node tools/provision.mjs --out runtime-har/libs/arm64-v8a
hvigorw --mode module -p module=runtimehar@default -p buildMode=release assembleHar --no-daemon
node tools/verify-har.mjs runtime-har/build/default/outputs/default/runtimehar.har
ohpm prepublish runtime-har/build/default/outputs/default/runtimehar.har

# ── 2) 桥接包 ────────────────────────────────────────────────
cd ../ohos-node-bridge
node tools/build-bridge.mjs
hvigorw --mode module -p module=library@default -p buildMode=release assembleHar --no-daemon
# 门禁：脚本不在本仓 ⇒ 用 §6.3 手工清单
ohpm prepublish library/build/default/outputs/default/library.har

# ── 3) 发布（交互式终端；先运行时、后桥接）────────────────────
ohpm publish ../ohos-node-runtime/runtime-har/build/default/outputs/default/runtimehar.har
ohpm publish library/build/default/outputs/default/library.har

# ── 4) 核验 ─────────────────────────────────────────────────
curl -s https://ohpm.openharmony.cn/ohpm/@fellow99/node-runtime
curl -s https://ohpm.openharmony.cn/ohpm/@fellow99/node-bridge
```

**判据**：`verify-har` exit 0 → `prepublish … succeed` → publish 生成审核单 → detail 接口出现该版本 → 干净目录 `ohpm install` 成功且 `libnode.so` SHA = 锁值。

---

## 附录 A：核对依据（2026-11-01）

```text
# 中心仓已上架版本（detail 接口）
@fellow99/node-bridge   versions = ['0.1.0', '0.2.0']      dist-tags.latest = 0.2.0
@fellow99/node-runtime  versions = ['24.2.0', '22.23.4', '24.22.1', '26.13.0']  dist-tags.latest = 26.13.0

# 只发布 AARCH64（arm64-v8a）一个架构

# 仓库 HEAD 版本（文件）
ohos-node-bridge/library/oh-package.json5      version = 0.2.0
ohos-node-runtime/runtime-har/oh-package.json5 version = 26.13.0
ohos-node-runtime/runtime.lock.json            active.node = 26.13.0

# 门禁脚本实测
node tools/release.mjs --check-version   → ① 12/12 OK；②③ 3 项 FAIL（拆分遗留，见 §6.2）

# 工具链
gradle 9.4.1 / JDK 21（与 Maven 发布共用）；ohpm 需随 command-line-tools 安装
```
