# 发布到 Maven Central

> **本文是什么**：三个 Android 制品（`node-bridge` / `node-runtime` / `node-pty`）**发布到
> [central.sonatype.com](https://central.sonatype.com/)（Maven Central）的方法与可照抄的流程（runbook）。
>
> | 项 | 值 |
> |---|---|
> | 文档日期 | 2026-11-01 |
> | 覆盖仓库 | [`aosp-node-bridge`](../aosp-node-bridge)、[`aosp-node-runtime`](../aosp-node-runtime)、[`aosp-node-pty`](../aosp-node-pty) |
> | 发布机制 | Gradle + [`com.vanniktech:gradle-maven-publish-plugin`](https://github.com/vanniktech/gradle-maven-publish-plugin) `0.37.0`（**Central Portal 直传**，无 OSSRH staging 流程）+ `signing` |
> | 依据 | 三仓 `publish.gradle.kts`、`gradle.properties`、`gradle.properties.local.example`、`tools/*.mjs`、`*.lock.json`；以及 repo1 上的真实制品（§附录 A） |
>
> 本文只讲**发布面**。引擎如何交叉编译见各仓库 `README.md` 与 `docs/000-工程规划.md`。

---

## 0. 速览

### 0.1 三个制品

| Maven 坐标 | 仓库 | Gradle module | AAR 内容 | 版本来源 | Central 现状 |
|---|---|---|---|---|---|
| `io.github.fellow99:node-bridge` | `aosp-node-bridge` | `:node-bridge` | Kotlin API + `libnode_bridge.so` + `libnode_host_client.so` + `assets/node-bridge/*.js` | `VERSION_NAME`（自身 semver） | ✅ 已发布 `0.1.0`、`0.2.0`（latest `0.2.0`） |
| `io.github.fellow99:node-runtime` | `aosp-node-runtime` | `:node-runtime` | `jni/arm64-v8a/libnode.so` + `libc++_shared.so` | `RUNTIME_VERSION`（**= Node 版本**） | ✅ 已发布 `18.20.4`、`22.23.4`、`24.2.0`、`24.22.1`、`26.13.0`（latest `26.13.0`） |
| `io.github.fellow99:node-pty` | `aosp-node-pty` | `:node-pty` | `jni/arm64-v8a/libpty_node.so`（**只有这一个 .so**） | `PTY_VERSION`（**= 配对 Node 版本**） | ✅ 已发布 `22.23.4`、`24.22.1`、`26.13.0`（latest `26.13.0`） |

> ⚠️ **只发布 AArch64（arm64-v8a）一个架构**，无 x86/x86_64/arm32 制品。

#### 已发布版本一览（一版本一链接，2026-11-01 实测在线）

| Maven 坐标 | 版本 | 操作系统 | 架构 | 发布链接 |
|---|---|---|---|---|
| `io.github.fellow99:node-bridge` | `0.1.0` | Android | arm64-v8a | <https://repo1.maven.org/maven2/io/github/fellow99/node-bridge/0.1.0/> |
| `io.github.fellow99:node-bridge` | `0.2.0` | Android | arm64-v8a | <https://repo1.maven.org/maven2/io/github/fellow99/node-bridge/0.2.0/> |
| `io.github.fellow99:node-runtime` | `18.20.4` | Android | arm64-v8a | <https://repo1.maven.org/maven2/io/github/fellow99/node-runtime/18.20.4/> |
| `io.github.fellow99:node-runtime` | `22.23.4` | Android | arm64-v8a | <https://repo1.maven.org/maven2/io/github/fellow99/node-runtime/22.23.4/> |
| `io.github.fellow99:node-runtime` | `24.2.0` | Android | arm64-v8a | <https://repo1.maven.org/maven2/io/github/fellow99/node-runtime/24.2.0/> |
| `io.github.fellow99:node-runtime` | `24.22.1` | Android | arm64-v8a | <https://repo1.maven.org/maven2/io/github/fellow99/node-runtime/24.22.1/> |
| `io.github.fellow99:node-runtime` | `26.13.0` | Android | arm64-v8a | <https://repo1.maven.org/maven2/io/github/fellow99/node-runtime/26.13.0/> |
| `io.github.fellow99:node-pty` | `22.23.4` | Android | arm64-v8a | <https://repo1.maven.org/maven2/io/github/fellow99/node-pty/22.23.4/> |
| `io.github.fellow99:node-pty` | `24.22.1` | Android | arm64-v8a | <https://repo1.maven.org/maven2/io/github/fellow99/node-pty/24.22.1/> |
| `io.github.fellow99:node-pty` | `26.13.0` | Android | arm64-v8a | <https://repo1.maven.org/maven2/io/github/fellow99/node-pty/26.13.0/> |

现状核对命令（2026-11-01 实测）：

```bash
curl -s https://repo1.maven.org/maven2/io/github/fellow99/node-bridge/maven-metadata.xml
curl -s https://repo1.maven.org/maven2/io/github/fellow99/node-runtime/maven-metadata.xml
curl -s -o /dev/null -w "%{http_code}\n" https://repo1.maven.org/maven2/io/github/fellow99/node-pty/
```

### 0.2 三个版本的「配对纪律」

| 关系 | 规则 | 校验在哪 |
|---|---|---|
| bridge ↔ runtime | **编译期不链接、运行期 `dlopen`**：换 Node 版本 **不需要**重编 bridge | `runtime.lock.json` 记录组合 |
| pty ↔ runtime | **必须逐字同版本**（`NODE_MODULE_VERSION` 不同即不可用）；`node-pty:<node>` 与 `node-runtime:<node>` 成对消费 | `pty.lock.json` 的 `pairedRuntime.coordinates` |
| runtime 版本 | 制品 version **等于** Node 版本（`24.22.1` = Node v24.22.1） | `runtime.lock.json` 的 `active.node` |

⇒ **发布顺序必须是：`node-runtime` → `node-pty` → `node-bridge`**。理由：
① pty 的 `DT_NEEDED libnode.so` 与 N-API 版本必须与某个**已在 Central 上**的 runtime 对齐；
② bridge 的 README 指向某个 runtime 版本；③ bridge 本身不依赖 runtime，最后发最安全。

### 0.3 三句话流程

1. **二进制就位**：`provision.mjs --check`（runtime/pty）或 `build-bridge.mjs`（bridge）必须真的产出 `.so`。
2. **本地全绿**：`publishToMavenLocal` + 结构/POM 断言（bridge 有 `tools/release-maven.mjs --check-version`）。
3. **上传 + 发布**：`gradle :<module>:publishToMavenCentral -PRELEASE=true` → 到 Portal 点 **Publish**（或用 `-PAUTOMATIC_RELEASE=true` 自动）。

---

## 1. 坐标、版本与工程约定

### 1.1 坐标是写死的

`publish.gradle.kts`（三仓各一份，内容几乎相同）里：

```kotlin
mp("coordinates", "io.github.fellow99", artifactId, versionName)   // groupId 硬编码
```

- `groupId` = **`io.github.fellow99`**（与 `GROUP` 属性一致；也与 GitHub 账号 `fellow99` 对得上，这是 `io.github.*` 命名空间校验的前提）。
- `artifactId` = **Gradle module 名**（`node-bridge` / `node-runtime` / `node-pty`），**不**取自 `POM_NAME`。
- `packaging` = `aar`（Android library）。

### 1.2 版本属性：**不能**用 `NODE_*` 前缀

| 仓库 | 属性 | 当前值 | 说明 |
|---|---|---|---|
| `aosp-node-bridge` | `VERSION_NAME` | `0.2.0` | 自身 semver |
| `aosp-node-runtime` | `RUNTIME_VERSION` | `24.22.1` | = Node 版本；**多版本发布**靠 `-PRUNTIME_VERSION=<node>` |
| `aosp-node-pty` | `PTY_VERSION` | `24.22.1` | = 配对 Node 版本；`-PPTY_VERSION=<node>` |

> ⚠️ **已知约束（工程注释 + 实操纪律）**：**Gradle 9.4 把所有 `NODE_` 前缀属性保留给内置 Node 工具链**，
> 写 `NODE_VERSION=…` 并用 `-PNODE_VERSION=…` 覆盖会被忽略（静默用默认值）。所以：
> 多版本发布一律用 `-PRUNTIME_VERSION=` / `-PPTY_VERSION=`；模块 `build.gradle.kts` 里 `version = providers.gradleProperty(...)`
> 取值后，还在 `afterEvaluate` 里对 publication 反射 `setVersion(...)` 做第二道覆盖。

### 1.3 每个 module 都会重新覆写 publication 版本

`node-runtime/build.gradle.kts`、`node-pty/build.gradle.kts`：

```kotlin
version = providers.gradleProperty("RUNTIME_VERSION").orNull ?: error("…需要 RUNTIME_VERSION…")
afterEvaluate {
    extensions.getByType(PublishingExtension::class.java).publications.forEach { pub ->
        (pub as Any).javaClass.methods.first { it.name == "setVersion" && it.parameterCount == 1 }
            .invoke(pub, version.toString())
    }
}
```

⇒ 发布时**只要 `-P` 与 lock 文件不一致就会打出错误的坐标版本**，所以 §4.2 的版本一致性检查是硬前置。

---

## 2. 一次性前置（只做一次）

### 2.1 账号与命名空间

| # | 事项 | 说明 |
|---|---|---|
| 1 | 注册/登录 [central.sonatype.com](https://central.sonatype.com/) | 用 **GitHub 账号**登录（`io.github.<user>` 命名空间靠 GitHub 归属校验） |
| 2 | 命名空间 `io.github.fellow99` | `io.github.<你的 GitHub 用户名>` 形式；`fellow99` 与账号一致 |
| 3 | 校验状态 | **已通过**——制品在线即为证据（`repo1.maven.org/maven2/io/github/fellow99/` 返回 200）。换账号/换 namespace 时重新走 Portal 的校验流程 |

> 命名空间**必须**与 GitHub 用户名逐字一致，否则 Portal 会拒绝上传（`400/409 Namespace not authorized`）。
> 如果以后想用 `com.example.*` 之类，需要在 Portal 里做 DNS TXT 校验。

### 2.2 Central Portal Token（代替旧 OSSRH 的账号口令）

Portal → **Profile** → 生成 **User Token** → 得到一对 `username` / `password`。
它们**不是**网页登录口令，只能用于上传；可随时吊销。

写入用户级 `~/.gradle/gradle.properties`（Windows：`C:\Users\<你>\.gradle\gradle.properties`）：

```properties
mavenCentralUsername=<token username>
mavenCentralPassword=<token password>
```

### 2.3 GPG 密钥（Central 强制签名）

```bash
# 生成（建议 4096 位、设口令；CI 场景可用无口令密钥 + 严格保管）
gpg --full-gen-key                     # RSA 4096 / 无过期 / 姓名 + 邮箱

# 查 KeyID（末 16 位十六进制即可填 signing.keyId）
gpg --list-secret-keys --keyid-format LONG

# 发布公钥（任选其一/多个；Central 校验时会去 keyserver 取）
gpg --keyserver keys.openpgp.org     --send-keys <KEYID>
gpg --keyserver keyserver.ubuntu.com --send-keys <KEYID>

# 备选：导出一份「内存密钥」直接塞进 gradle.properties（推荐，免文件路径问题）
gpg --armor --export-secret-keys <KEYID> > node-bridge-signing.asc
```

> 常见误解：GnuPG **2.1+ 不再生成 `secring.gpg`**，所以 `signing.secretKeyRingFile=~/.gnupg/secring.gpg`
> 往往不存在。本工程的 `publish.gradle.kts` 已兼容：**没给 `signing.keyData` 但给了 `secretKeyRingFile` 时，
> 会自动用 `gpg --export-secret-keys` 临时导出成 in-memory keyData**（导出失败即硬失败）。
> 最稳的写法是直接把 `signing.keyData` 填成全文本（`.properties` 里多行值用**反斜杠 + 换行**续行；也可写成单行把换行转义成 `\n`）。

### 2.4 用户级凭据文件（**绝不入库**）

对应三仓的 `gradle.properties.local.example`：

```properties
# ── Central Portal ──
mavenCentralUsername=<token username>
mavenCentralPassword=<token password>

# ── GPG 签名 ──
signing.keyId=<KEYID 末 16 位>
signing.password=<私钥口令>
# 二选一：
signing.keyData=<-----BEGIN PGP PRIVATE KEY BLOCK----- … -----END PGP PRIVATE KEY BLOCK----->
#signing.secretKeyRingFile=C:/Users/<你>/.gnupg/secring.gpg   # 由脚本自动导出为 in-memory
# ── 发布开关（默认 false；true 时 Central/签名任务才可执行）──
RELEASE=true
```

仓库内的 `gradle.properties` **只放公共元数据**，且 `RELEASE=false` 兜底：

| 属性 | 落到 POM 的位置 | 三仓现状 |
|---|---|---|
| `GROUP` | `groupId`（实际由 `coordinates()` 硬编码 `io.github.fellow99`） | 一致 |
| `POM_NAME` | `<name>` | `aosp-node-bridge` / `node-runtime` / `node-pty` |
| `POM_DESCRIPTION` | `<description>` | 各仓独立 |
| `POM_URL` | `<url>` | 各仓 GitHub URL |
| `POM_LICENCE_NAME` / `POM_LICENCE_URL` | `<licenses><license>` | MIT |
| `POM_SCM_CONNECTION` / `POM_SCM_DEVELOPMENT_CONNECTION` | `<scm><connection>` / `<developerConnection>` | 各仓 git 地址 |
| `POM_DEVELOPER_ID` / `POM_DEVELOPER_NAME` | `<developers><developer>` | `fellow99` |
| （`<scm><url>`） | Portal **强制要求** | 由 `publish.gradle.kts` 的 `generatePomFile*` 补丁写入（§3.3） |

### 2.5 工具链

| 工具 | 本机实测 | 用途 |
|---|---|---|
| JDK | **21**（Temurin） | AGP 9.2.1 / Gradle 9.4.1 的要求 |
| Gradle | **9.4.1**（系统安装；三仓**无** `gradlew`） | `publishToMavenCentral` 等任务 |
| Android SDK | `platforms/android-36`、`build-tools 36.x` | AAR 构建；`local.properties` 的 `sdk.dir`（gitignored） |
| Android NDK | **`27.2.12479018`**（bridge 编译引导件必需） | 见 §4.1 的「NDK 静默跳过」陷阱 |
| GnuPG | **2.4.9** | 签名 / 导出密钥 |
| Node.js | ≥18 | `tools/*.mjs`（`provision` / `build-bridge` / `release-maven`） |

---

## 3. 发布管线：`publish.gradle.kts` 到底做了什么

三个仓库共用同一套发布配置（`apply(from = rootProject.file("publish.gradle.kts"))`），要点逐条：

### 3.1 插件加载方式

`settings.gradle.kts` 的 `pluginManagement` 里声明：

```kotlin
plugins { id("com.vanniktech.maven.publish.base") version "0.37.0" }
```

模块 `build.gradle.kts` 里 `id("com.vanniktech.maven.publish.base")`。
（历史踩坑：插件类在 0.37.0 变成 abstract，脚本里反射 `newInstance` 不可行；最终改为让 Gradle 自己按 id 加载。）

### 3.2 关键配置（求值期完成，顺序不能乱）

| 调用 | 作用 |
|---|---|
| `coordinates("io.github.fellow99", artifactId, version)` | 固定 groupId/artifactId/version |
| `pomFromGradleProperties()` | 从 §2.4 的属性生成 POM 字段 |
| `publishToMavenCentral(false, <validation>)` | 直传 Central Portal；`requiresToken=false` 让**配置期**不因缺 Token 报错（执行期另由 `enabled` 门禁兜住） |
| `validation = NONE` | 默认：**上传后人工在 Portal 点 Publish**（`USER_MANAGED`） |
| `validation = PUBLISHED` | 加 `-PAUTOMATIC_RELEASE=true` 时：校验通过后**自动发布并等待上线** |
| `signAllPublications()` | 所有 publication（`.aar` / `.pom` / `.module` / `sources.jar`）都产出 `.asc` |
| `configure(AndroidSingleVariantLibrary(JavadocJar.None, SourcesJar.Sources, "release"))` | 只发 **release** variant + `-sources.jar`，**不发 javadoc jar**（Android 制品在 Central 上不需要；已发布制品可证） |

### 3.3 两个「补丁」——都是为了过 Portal 的强制校验

1. **`<scm><url>` 补丁**：`pomFromGradleProperties()` 只填 `connection`/`developerConnection`，而 Portal 要求 `<scm>` 里必须有 `<url>`。
   实现：`tasks.configureEach { if (name.startsWith("generatePomFile")) doLast { … 往 `build/publications/maven/pom-default.xml` 注入 <url> … } }`。
   已发布 POM 里能看到结果（§附录 A）。
2. **`signing.secretKeyRingFile` → in-memory keyData 桥接**：见 §2.3 的说明。

### 3.4 发布门禁（防误发）

```kotlin
val releaseEnabled   = providers.gradleProperty("RELEASE").orNull?.toBoolean() == true
val portalTokenReady = mavenCentralUsername 与 mavenCentralPassword 都非空

tasks.configureEach {
    if (name.startsWith("signMaven") || name == "publishToMavenCentral" || name.contains("ToMavenCentral"))
        enabled = enabled && releaseEnabled && portalTokenReady
}
```

⇒ **默认 `RELEASE=false` 时这些任务是 SKIPPED（不是失败）**：很容易误判成「发布成功但什么都没上传」。
若本地也要验签名（例如跑 `release-maven.mjs` 的 `.asc` 断言），必须同时满足 `RELEASE=true` + Token + GPG。

### 3.5 一次发布会产生什么文件

以已上线的 `node-runtime:24.22.1` 为例（repo1 目录实测）：

```
node-runtime-24.22.1.aar              27,607,401 B
node-runtime-24.22.1.pom
node-runtime-24.22.1.module           ← Gradle Module Metadata（Gradle 消费方优先用它）
node-runtime-24.22.1-sources.jar      ← 261 B（空的源码包，Central 要求存在）
+ 上述每个文件的 .asc / .md5 / .sha1
```

---

## 4. 发布前检查（顺序执行，全绿才发）

### 4.1 二进制必须真的就位（最容易静默失败的一步）

| 仓库 | 命令 | 判据 | 陷阱 |
|---|---|---|---|
| `aosp-node-runtime` | `node tools/provision.mjs --check` | 八项 ELF 门禁 + SHA/大小 = `runtime.lock.json` | 未投放时 `--check` 会失败 ⇒ 先 `--out node-runtime/src/main/jniLibs` |
| `aosp-node-pty` | `node tools/provision.mjs --check` | 八项 + **`DT_NEEDED` 含 `libnode.so`** + 未定义 `napi_*` | 它**只**投放 `libpty_node.so`，绝不带 `libnode.so`/`libc++_shared.so` |
| `aosp-node-bridge` | `node tools/build-bridge.mjs`（`preBuild` 已自动依赖它） | 产出 `libnode_bridge.so` + `libnode_host_client.so` | ⚠️ **有两处静默跳过**（详见下） |

#### ⚠️ 发布前必须识破的两个静默跳过（`build-bridge.mjs` 都 `exit 0`）

| 缺什么 | 脚本行为 | 后果 | 修法 |
|---|---|---|---|
| Android **NDK** | `未找到 Android NDK —— 跳过原生构建` | AAR 里**一个 `.so` 都没有** | `sdkmanager "ndk;27.2.12479018"` 或 `export ANDROID_NDK_HOME=<ndk>` |
| **NAPI 头文件**（`node-bridge/src/main/cpp/include/node/`，**未入库**，fresh clone 里是空目录） | `跳过 node_host_client：缺 Node NAPI 头` | AAR 里**只有** `libnode_bridge.so`，Node→Kotlin 的宿主桥丢失（**功能缺失但发布成功**） | 取同版本 Node headers 补齐后重编（下）|

```bash
# 修 NAPI 头（取「配对 Node 版本」的 headers；只需 4 个头文件）
curl -sL https://nodejs.org/dist/v24.22.1/node-v24.22.1-headers.tar.gz | tar xz -C /tmp
cp /tmp/node-v24.22.1/include/node/{node_api.h,node_api_types.h,js_native_api.h,js_native_api_types.h} \
   node-bridge/src/main/cpp/include/node/

# 重编（本机实测：NDK 27.2.12479018 下两件都能出，23256 B + 11128 B）
node tools/build-bridge.mjs --out node-bridge/src/main/jniLibs/arm64-v8a
```

**发布前的硬判据**：AAR 内 `jni/arm64-v8a/` 恰有 **2 个** `.so`。已上线的 `0.2.0` AAR 实测就是
`libnode_bridge.so 23,256 B` + `libnode_host_client.so 14,384 B`（外加 `assets/node-bridge/*.js` 三个 + `classes.jar`）。

投放命令（从 CI/Release 产物）：见各仓 `README.md`；`runtime`/`pty` 的 `provision.mjs` 都支持
`--out <dir> --from <so>`、`--inspect <so>`，退出码 `0` 全过 / `2` 门禁不合格 / `1` 用法或 IO 错。

### 4.2 版本一致性（三处对齐，否则会发出错坐标的制品）

| 仓库 | 断言 | 命令 |
|---|---|---|
| runtime | `-PRUNTIME_VERSION` == `runtime.lock.json` 的 `active.node`；git tag `v<ver>` 与 GitHub Release 一致 | `grep -n '"node"' runtime.lock.json \| head -2` |
| pty | `-PPTY_VERSION` == `pty.lock.json` 的 `active.node` **且** == `pairedRuntime.coordinates` 里的版本 | `grep -n 'pairedRuntime' -A2 pty.lock.json` |
| bridge | `VERSION_NAME` == git tag `v<ver>` | `grep -n '^VERSION_NAME' gradle.properties` |

> ⚠️ **pty 的配对例外必须显式核对**：`pty.lock.json` 的 `active.node` 目前是 **`26.13.0`**，而 Central 上
> `node-runtime` 最高只到 **`24.22.1`**。若现在直接 `-PPTY_VERSION=26.13.0` 发布，消费者拿到的
> `node-pty:26.13.0` 会依赖一个**不存在的** `node-runtime:26.13.0`（N-API 147 也找不到引擎）。
> 发布 pty 前务必先确认下面的 URL 返回 200：
>
> ```bash
> curl -s -o /dev/null -w "%{http_code}\n" \
>   https://repo1.maven.org/maven2/io/github/fellow99/node-runtime/<ver>/node-runtime-<ver>.aar
> ```

### 4.3 本地产物验证：`publishToMavenLocal` + 结构断言

```bash
cd aosp-node-runtime          # 或 aosp-node-pty / aosp-node-bridge
gradle :node-runtime:publishToMavenLocal --no-daemon
ls ~/.m2/repository/io/github/fellow99/node-runtime/<ver>/     # .aar/.pom/.module/.sources.jar + .asc（需 RELEASE=true）
```

bridge 有现成脚本（**它会自己跑 `publishToMavenLocal`** 并断言 AAR 结构）：

```bash
cd aosp-node-bridge
node tools/release-maven.mjs --check-version
```

它检查三件事：

| 段 | 断言 |
|---|---|
| ① | `gradle.properties` 的 `VERSION_NAME` 是合法 semver |
| ② | `gradle :node-bridge:publishToMavenLocal` 后 `~/.m2/.../node-bridge-<ver>.aar` 存在，且**同名 `.asc` 存在** |
| ③ | AAR 内**恰有 2 个** `jni/arm64-v8a/*.so`；`assets/node-bridge/{bootstrap,host,selfcheck}.js` 都在 |

> ⚠️ 第 ② 段的 `.asc` 断言只有在 `RELEASE=true` + Token + GPG 齐全时才成立（否则签名任务被 §3.4 门禁 disable）。
> 这是**脚本与门禁的设计选择**，不是 bug：本地只验「能不能发」，真正签名发生在发布时。
> （本条为读码结论；本次只实跑了 `tools/build-bridge.mjs` 验证两件 `.so` 可产出，未实跑 gradle 发布链路。）

### 4.4 POM 自查（Portal 的硬性字段）

```bash
cd aosp-node-runtime
gradle :node-runtime:generatePomFileForMavenPublication --no-daemon
sed -n '1,60p' node-runtime/build/publications/maven/pom-default.xml
```

必须看到：`<modelVersion>`、`<groupId>/<artifactId>/<version>/<packaging>aar`、`<name>`、`<description>`、`<url>`、
`<licenses>`、`<developers>`、`<scm>` 含 `connection`+`developerConnection`+**`<url>`**。
（已上线 POM 逐字见 §附录 A。）

### 4.5 发布前最后一道：任务是否存在

```bash
gradle :node-pty:tasks --all | grep -i -E "mavenCentral|signMaven"
```

应当能看到 `publishToMavenCentral`、`publishMavenPublicationToMavenCentralRepository`、`signMavenPublication` 等。
若看不到，说明插件没被 `settings.gradle.kts` 的 `pluginManagement` 加载成功。

---

## 5. 正式发布

### 5.1 命令（逐仓，注意 `-PRELEASE=true`）

```bash
# ① 引擎（必须先发；pty 与 bridge 都按它对齐）
cd aosp-node-runtime
node tools/provision.mjs --check
gradle :node-runtime:publishToMavenCentral -PRELEASE=true -PRUNTIME_VERSION=24.22.1 --no-daemon

# ② node-pty（版本必须与已上线的 runtime 逐字一致）
cd ../aosp-node-pty
node tools/provision.mjs --check
gradle :node-pty:publishToMavenCentral -PRELEASE=true -PPTY_VERSION=24.22.1 --no-daemon

# ③ 桥接件（不依赖 runtime，最后发）
cd ../aosp-node-bridge
node tools/release-maven.mjs --check-version        # 先跑结构断言
gradle :node-bridge:publishToMavenCentral -PRELEASE=true --no-daemon
```

> `-P<版本>` 建议**显式传**：等于把 `gradle.properties` 的默认值与 lock 文件对齐一次，避免默认值漂移。
> 也可以只改 `gradle.properties` 再发；两种做法都必须保证 §4.2 的三处一致。

### 5.2 两种发布模式

| 模式 | 怎么触发 | 现象 | 适用 |
|---|---|---|---|
| **人工确认发布**（默认） | 不加 `-PAUTOMATIC_RELEASE` | 上传后 deployment 处于「待发布」，**repo1 上还查不到**；需在 Portal → **Deployments** → 点 **Publish** | 首次发布 / 需要人工过一遍校验报告 |
| **自动发布** | `-PAUTOMATIC_RELEASE=true` | 校验通过后自动发布并等待上线（gradle 任务等到上线才结束） | 版本已被证明可用（如 patch 级重发） |

```bash
gradle :node-runtime:publishToMavenCentral -PRELEASE=true -PRUNTIME_VERSION=24.22.1 \
        -PAUTOMATIC_RELEASE=true --no-daemon
```

### 5.3 上传过程中会发生什么

1. 构建 `release` variant → 生成 `.aar`；
2. `generatePomFileForMavenPublication`（含 §3.3 的 `<scm><url>` 补丁）；
3. 生成 `.module` 与 `-sources.jar`；
4. `signMavenPublication` 给每个文件出 `.asc`；
5. 打包成 Portal **deployment** 直传（`bundleName`/deploymentId 可记入 lock）；
6. 校验（POM 字段、签名可验证、GAV 未被占用）→ 通过后进入「待发布」或自动发布。

> 体积参考：`node-runtime` AAR ≈ **27.6 MB**（`24.22.1`）、`node-bridge` AAR ≈ **67.8 KB**（`0.2.0`）。
> 上传慢通常是网络，不是体积问题。

---

## 6. 发布后核验与回填

### 6.1 三条独立判据

```bash
# ① 目录能列出（最直接）
curl -s https://repo1.maven.org/maven2/io/github/fellow99/node-runtime/24.22.1/ | grep -E "\.(aar|pom|module|jar)(\.asc)?</a>"

# ② metadata 里出现该版本
curl -s https://repo1.maven.org/maven2/io/github/fellow99/node-runtime/maven-metadata.xml

# ③ 依赖真的能解析（最强：干净工程）
cd /tmp && rm -rf mc-verify && mkdir mc-verify && cd mc-verify
cat > settings.gradle.kts <<'EOF'
rootProject.name = "mc-verify"
EOF
cat > build.gradle.kts <<'EOF'
plugins { java }
repositories { mavenCentral() }
val probe by configurations.creating
dependencies {
    probe("io.github.fellow99:node-runtime:24.22.1@aar")
    probe("io.github.fellow99:node-bridge:0.2.0@aar")
}
tasks.register("resolveProbe") { doLast { probe.files.forEach { println(it) } } }
EOF
gradle resolveProbe --no-daemon -q
```

### 6.2 回填 lock（可追溯证据）

已上线的 `node-runtime:24.22.1` 在 `runtime.lock.json` 里长这样——这就是回填模板：

```json
"mavenCentral": {
  "coordinates": "io.github.fellow99:node-runtime:24.22.1",
  "aar": "https://repo1.maven.org/maven2/io/github/fellow99/node-runtime/24.22.1/node-runtime-24.22.1.aar",
  "aarSize": 27607401,
  "deploymentId": "bfe63820-c5b5-4b5e-b913-335fbb432b4a",
  "tag": "node-runtime-v24.22.1",
  "published": true
}
```

`node-pty` 侧（`pty.lock.json`）目前是 `"mavenCentral": { "coordinates": "…:node-pty:24.22.1", "published": false }`
—— 发布成功后把 `published` 改 `true` 并补 `aar` / `aarSize` / `deploymentId` / `tag`。

### 6.3 消费方回归

```kotlin
// 消费方 build.gradle.kts
dependencies {
    val nodeVer = "24.22.1"
    implementation("io.github.fellow99:node-bridge:0.2.0")
    runtimeOnly("io.github.fellow99:node-runtime:$nodeVer")
    runtimeOnly("io.github.fellow99:node-pty:$nodeVer")     // 与 runtime 必须同版本
}
```

装成 APK 后核对：`libs/arm64-v8a/libnode.so`（来自 runtime AAR）、`libpty_node.so`（来自 pty AAR）、
`libnode_bridge.so` + `libnode_host_client.so`（来自 bridge AAR）都在 `nativeLibraryDir`；
`libc++_shared.so` 只有一份（见 §8）。

---

## 7. 版本不可变与重发策略

| 场景 | Central 的现实 | 该怎么做 |
|---|---|---|
| 同版本重发（如 `24.22.1` 已存在，想换二进制） | **不可能**：GAV 不可覆盖 | 换版本号（新 Node 版本，或 `24.22.2`），或等下一次 Node 升级 |
| 仅元数据/POM 有错 | 同版本不可改 | 只能发新版本（Central 不接受覆盖；这与 ohpm 纪律一致） |
| 想撤回 | Portal 允许删除 deployment（已 released 的版本删除代价高，且可能破坏已有构建） | 优先「发新版本 + README 指明」；删除是最后手段 |
| pty 同 Node 版本内升级 node-pty 补丁 | Maven 无 classifier 约定 | 规划文档（`aosp-node-pty/docs/000-工程规划.md` §10）把「classifier / build 序号策略」列为 M3 待定项；当前 GitHub Release 阶段用 run id + `--clobber` 追溯 |

> 推论：**发布前的 §4 检查不可省**。`node-runtime` 的版本号被 Node 版本钉死，
> 一次「发错」就意味着要么发一个误导性的 `24.22.2`，要么等下一个 Node 版本。

---

## 8. 常见坑速查

| 现象 | 根因 | 处置 |
|---|---|---|
| `gradle publishToMavenCentral` 秒过、Central 上什么都没有 | `RELEASE=false`（或缺 Token）⇒ 任务被 §3.4 `enabled=false` **跳过** | 加 `-PRELEASE=true`，并确认用户级 `gradle.properties` 有 Token |
| `-PNODE_VERSION=…` 无效 | **Gradle 9.4 保留 `NODE_*` 前缀** | 用 `-PRUNTIME_VERSION` / `-PPTY_VERSION` |
| `Invalid signature` / 校验失败 | 公钥没上传到 keyserver，或口令错 | `gpg --send-keys`；核对 `signing.keyId/password` |
| `secring.gpg` 不存在 | GnuPG 2.1+ 不再写该文件 | 用 `signing.keyData`（推荐）或让脚本自动导出（§2.3） |
| POM 被拒：缺 `<scm><url>` | 插件只填 connection | 本工程已用 `generatePomFile` 补丁修好；换插件版本时需**重新核对** |
| AAR 里没有 `.so` 却发布成功 | **NDK 缺失时 `build-bridge.mjs` 打印「跳过」并 exit 0** | 发布前断言 AAR 内 `.so` 数量（bridge 恰 2 个） |
| AAR 里只有 1 个 `.so`（缺 `libnode_host_client.so`） | **NAPI 头目录未入库**，fresh clone 里为空 → 脚本静默跳过 | 按 §4.1 取 Node headers 补 `node-bridge/src/main/cpp/include/node/` 后重编 |
| runtime AAR 里没有 `libnode.so` | 没跑 `provision.mjs --out` | `node tools/provision.mjs --check` 必须先绿 |
| pty AAR 里出现 `libnode.so` | provision 用错 / 手工拷贝 | pty **只**投放 `libpty_node.so`；`DT_NEEDED libnode.so` 靠消费方 runtime 提供 |
| 消费方 APK 里出现两份 `libc++_shared.so` | 多个 AAR 都带 C++ 运行时 | 按仓库约定：`libc++_shared.so` **只由 `node-runtime` 提供**；bridge/pty 不带 |
| `Namespace not authorized` / `400` | namespace 与 GitHub 账号不一致 | 用 `io.github.fellow99`（= GitHub `fellow99`），或先做 DNS 校验 |
| 上传后 repo1 一直 404 | deployment 停在「待发布」（默认 `validation=NONE`） | 到 Portal → Deployments → **Publish**，或下次用 `-PAUTOMATIC_RELEASE=true` |
| pty 发布后消费方 `ClassNotFound`/`UnsatisfiedLinkError` | pty 与 runtime 版本不配对（N-API 不一致） | 两者版本号必须逐字相同；发布前用 §4.2 的 curl 断言 runtime 已在线 |
| `publishToMavenLocal` 没有 `.asc` | 签名任务被门禁 disable | 本地验签名需 `RELEASE=true` + Token + GPG |

---

## 9. 一页速查（TL;DR）

```bash
# ── 0) 一次性凭据（用户级 ~/.gradle/gradle.properties，绝不入库）──
mavenCentralUsername=<Portal token user>
mavenCentralPassword=<Portal token pass>
signing.keyId=<KEYID>            # gpg --list-secret-keys --keyid-format LONG
signing.password=<passphrase>
signing.keyData=<armored private key>       # 或 signing.secretKeyRingFile（脚本会导出）
RELEASE=true

# ── 1) 二进制 + 版本一致性 ───────────────────────────────
cd aosp-node-runtime && node tools/provision.mjs --check
cd ../aosp-node-pty   && node tools/provision.mjs --check
cd ../aosp-node-bridge && export ANDROID_NDK_HOME=<ndk>   # 否则 .so 会被静默跳过
ls node-bridge/src/main/cpp/include/node/                  # 必须有 4 个 NAPI 头，否则少了 host_client.so
node tools/build-bridge.mjs --out node-bridge/src/main/jniLibs/arm64-v8a
ls node-bridge/src/main/jniLibs/arm64-v8a                  # 必须恰好 2 个 .so

# ── 2) 本地验证 ────────────────────────────────────────
gradle :node-runtime:publishToMavenLocal --no-daemon
gradle :node-runtime:generatePomFileForMavenPublication --no-daemon
sed -n '1,60p' node-runtime/build/publications/maven/pom-default.xml   # 必须有 <scm><url>
cd ../aosp-node-bridge && node tools/release-maven.mjs --check-version

# ── 3) 发布（顺序：runtime → pty → bridge）────────────────
cd ../aosp-node-runtime
gradle :node-runtime:publishToMavenCentral -PRELEASE=true -PRUNTIME_VERSION=24.22.1 --no-daemon
cd ../aosp-node-pty
gradle :node-pty:publishToMavenCentral     -PRELEASE=true -PPTY_VERSION=24.22.1 --no-daemon
cd ../aosp-node-bridge
gradle :node-bridge:publishToMavenCentral  -PRELEASE=true --no-daemon

# ── 4) Portal 点 Publish（或上面加 -PAUTOMATIC_RELEASE=true）──

# ── 5) 核验 + 回填 lock ─────────────────────────────────
curl -s https://repo1.maven.org/maven2/io/github/fellow99/node-runtime/maven-metadata.xml
curl -s -o /dev/null -w "%{http_code}\n" \
  https://repo1.maven.org/maven2/io/github/fellow99/node-pty/24.22.1/node-pty-24.22.1.aar
```

**判据**：`provision --check` exit 0 → `publishToMavenLocal` 产物齐全 → `publishToMavenCentral` 上传成功
→ Portal Publish → repo1 目录出现 `.aar/.pom/.module/sources.jar` + `.asc` → 干净工程能解析依赖
→ lock 的 `mavenCentral.published = true`。

---

## 10. 五条 CLI 线的复用（2026-11-02 登记，⬜ 未实测）

> 工作区新增五条 exec 型工具线（`node-cli` / `python-cli` / `git-cli` / `uv-cli` / `pnpm-cli`，
> 见 `docs/aosp+node+python+git运行方案.md` §9）。它们沿用与 node-pty **完全同构**的发布面
> （settings pluginManagement 装 vanniktech 0.37.0 + 仓库根 `publish.gradle.kts` 镜像移植 +
> `<tool>-cli.lock.json` 同纪律），并采用 **「GitHub Release 先行、Maven Central 随后」**
> 的两段式节奏（pty 同款）：

| 仓 | 坐标（CLI_VERSION = 上游版本） | GitHub Release（先行） | Maven Central（随后） | 备注 |
|---|---|---|---|---|
| aosp-node-cli | `io.github.fellow99:node-cli:<Node版本>` | M1 | M3 | 与 runtime/pty 同版本配对 |
| aosp-python-cli | `…:python-cli:<CPython 版本>` | M1 | M3 | assets 携带 stdlib/ensurepip/CA |
| aosp-git-cli | `…:git-cli:<git 版本>` | M1 | M3 | GPLv2 对应源码随 Release 归档 |
| aosp-uv-cli | `…:uv-cli:<uv 版本>` | M1 | M3 | cargo 许可清单随 Release 归档 |
| aosp-pnpm-cli | `…:pnpm-cli:<pnpm 版本>` | M3.5 | M4 | 运行期依赖 node-cli；openProbes 先行 |

- **先行段**（各仓 M1/M3.5）：tag `v<上游版本>` 推送触发真实交叉编译流水线 → exec 门禁 →
  Release 归档（二进制 + assets + BUILD/FETCH-METADATA + LICENSES + `.sha256`）→
  lock 回写 `status=released`；
- **随后段**（各仓 M3/M4）：完全按本文 §2（一次性凭据）→ §4（发布前检查）→ §5（上传与
  两种发布模式）→ §6（核验 + 回填 `mavenCentral.published=true`）执行；版本源属性为
  `CLI_VERSION`（与 node 系 `PTY_VERSION`/`RUNTIME_VERSION` 同族纪律；CLI 线不存在
  NODE_ 前缀问题）；
- 本文 §1–§9 的全部纪律（版本不可变、Portal Publish、POM 自查、常见坑）对五仓同样适用；
  五仓各自的 `docs/000-工程规划.md`「发布模式」小节为权威入口；⬜ 五线首次实走 Maven 后
  回填本节实测事实。

---

## 附录 A：现状实测（2026-11-01）

```text
# repo1 metadata
io.github.fellow99:node-bridge  versions = 0.1.0, 0.2.0     latest = 0.2.0   lastUpdated 20261025111354
io.github.fellow99:node-runtime versions = 18.20.4, 22.23.4, 24.2.0, 24.22.1, 26.13.0   latest = 26.13.0
io.github.fellow99:node-pty     versions = 22.23.4, 24.22.1, 26.13.0   latest = 26.13.0

# 只发布 AARCH64（arm64-v8a）一个架构

# node-runtime/24.22.1 目录（实测）
node-runtime-24.22.1.aar                27,607,401 B
node-runtime-24.22.1.aar.asc                       201 B
node-runtime-24.22.1.pom / .pom.asc
node-runtime-24.22.1.module                      3,110 B
node-runtime-24.22.1-sources.jar                   261 B（+ .asc/.md5/.sha1）

# node-bridge/0.2.0 目录（实测）
node-bridge-0.2.0.aar                       67,827 B（+ .asc/.md5/.sha1）
node-bridge-0.2.0.pom                       1,918 B
node-bridge-0.2.0.module                    3,443 B
node-bridge-0.2.0-sources.jar              10,704 B

# node-bridge-0.2.0.aar 内容（实测，说明「必须 2 个 .so」的含义）
jni/arm64-v8a/libnode_bridge.so            23,256 B
jni/arm64-v8a/libnode_host_client.so       14,384 B
assets/node-bridge/{bootstrap,host,selfcheck}.js   4,476 / 5,145 / 10,251 B
classes.jar                                45,264 B

# 本机 build-bridge.mjs 实测（NDK 27.2.12479018，补齐 Node v24.22.1 headers 后）
libnode_bridge.so                          23,256 B
libnode_host_client.so                     11,128 B   （与已发布版差异来自 headers/flags）

# 已上线 POM 关键字段（node-bridge:0.2.0 实测，注意 <scm><url> 补丁生效）
<groupId>io.github.fellow99</groupId><artifactId>node-bridge</artifactId><version>0.2.0</version>
<packaging>aar</packaging>
<name>aosp-node-bridge</name>
<url>https://github.com/fellow99/aosp-node-bridge</url>
<licenses><license><name>MIT License</name><url>https://opensource.org/licenses/MIT</url></license></licenses>
<developers><developer><id>fellow99</id><name>fellow99</name></developer></developers>
<scm><connection>scm:git:git://…</connection><url>https://github.com/fellow99/aosp-node-bridge</url></scm>

# lock 里的发布事实
runtime.lock.json → 26.13.0: deploymentId 016f84e3-8dbf-4224-9cd2-13dcde11cc0c, published ✓（回填待补）
runtime.lock.json → 22.23.4: deploymentId 0a38873e-1c54-42b9-9fc3-7aa41c2ddf43, published ✓（回填待补）
runtime.lock.json → 24.22.1: deploymentId bfe63820-c5b5-4b5e-b913-335fbb432b4a, tag node-runtime-v24.22.1, published true
runtime.lock.json → 24.2.0 : deploymentId f610b50b-a831-4666-b8e0-47eae0cc954b, published true
pty.lock.json     → 26.13.0: deploymentId 99a6043a-afc2-4a33-bf1e-7846880f7894, published ✓（回填待补）
pty.lock.json     → 24.22.1: deploymentId 4f0170e1-f67e-433c-8cf8-975181846f0a, published ✓（回填待补）
pty.lock.json     → 22.23.4: deploymentId ffbabb73-1713-4cd9-a800-eb5456b4fb6d, published ✓（回填待补）

# 本机工具链
gradle 9.4.1 / Launcher JVM 21 (Temurin) / gpg 2.4.9
Android SDK: platforms/android-36, android-36.1；NDK 27.2.12479018（已安装）
console / note: 未实跑 gradle publish 链路（需真实凭据），本文发布命令与任务名来自
publish.gradle.kts + 已上线制品的反向验证。
```
