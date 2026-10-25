# PaperMC 版本基线与构建配置
本文只写官方文档、官方 API 与官方仓库里能查到的内容。标 `（未证实）` 的表示该来源里没有记录。

## 版本基线（先读工程，不要背版本号）
**先读工程已有文件，再决定写什么版本号。** 读取位置与取值：

| 位置 | 取值 |
|---|---|
| `build.gradle.kts` | 已有的 `paper-api` 坐标或 `paperweight.paperDevBundle(...)` 里的版本串 |
| `build.gradle.kts` | `java { toolchain { ... JavaLanguageVersion.of(n) } }` 里的 n |
| `plugin.yml` / `paper-plugin.yml` | `api-version` 字段（决定服务器最低版本） |

### 两套版本字符串，分界点在 26.1
| 时期 | 形式 | 例子 |
|---|---|---|
| `1.21.11` 及更早 | `{VERSION}-R0.1-SNAPSHOT`，无法指定具体 build | `1.21.11-R0.1-SNAPSHOT` |
| `26.1` 起（现行） | `<version>.build.+`；固定 build 用 `{VERSION}.build.25-stable` | `26.3.build.+` |

`project-setup` 原文（节选）：

> "Before `26.1` (`1.21.11` and below), the version string format used was `{VERSION}-R0.1-SNAPSHOT`, with no way to reference a specific build."

### 查实时数据
| 目的 | 端点 |
|---|---|
| 版本分组与版本列表 | `https://fill.papermc.io/v3/projects/paper` |
| 某版本全部 build | `https://fill.papermc.io/v3/projects/paper/versions/26.3/builds` |
| 某版本最新 build | `https://fill.papermc.io/v3/projects/paper/versions/1.21.11/builds/latest` |
| `paper-api` 已发布版本 | `https://repo.papermc.io/repository/maven-public/io/papermc/paper/paper-api/maven-metadata.xml` |
| `dev-bundle` 已发布版本 | `https://repo.papermc.io/repository/maven-public/io/papermc/paper/dev-bundle/maven-metadata.xml` |

build 对象含 `id`、`channel`（`STABLE` / `BETA` / `ALPHA`）、`downloads`。注意 `versions` 的键是**分组名**：`.../versions/26.1/builds` 返回 **HTTP 404** `version_not_found`，真实版本标识是 `26.1.1` / `26.1.2`。

研究时点观测值（会变，仅供对照）：最新 build `26.3.build.140-beta`；`paper-api` 元数据 `<latest>` 为 `26.3-pre-2.build.0-alpha`。这些字面量由文档构建脚本的常量替换逻辑推导 + Maven 元数据交叉验证得到，**不是文档页面直接打印的**。
## Java 版本
> "Paper requires at least **Java 25** to run."

| Paper 版本 | 推荐 Java |
|---|---|
| 1.7.10 – 1.11 | 8 |
| 1.12 – 1.16.4 | 11 |
| 1.16.5 | 16 |
| 1.17 – 1.19 | 17 |
| 1.20 – 1.21.11 | 21 |
| **26.1+** | **25** |

文档给出的工具链片段（源码里硬编码 25，不是占位符）：
```kotlin title="build.gradle.kts"
java {
  toolchain.languageVersion.set(JavaLanguageVersion.of(25))
}
```
工具链高于运行服务器的 JVM 时，类无法被加载（JVM 层面表现为 `UnsupportedClassVersionError`，**未证实**：研究文件里没有这个报错名；服务器侧另有 Java 版本检查，可用系统属性 `Paper.IgnoreJavaVersion`（默认 `false`）绕过，但不要拿它代替正确的工具链）。

## 依赖坐标

仓库与依赖（文档原文，configuration 是 `compileOnly`，因为运行期由服务器提供该 API）：
```kotlin title="build.gradle.kts"
repositories {
  maven(url = "https://repo.papermc.io/repository/maven-public/") {
    name = "papermc"
  }
}

dependencies {
  compileOnly("io.papermc.paper:paper-api:26.3.build.+")
}
```
Maven 变体被文档明确标注 **"Maven (Discouraged)"**，坐标同为 `io.papermc.paper:paper-api`（文档的 Maven 片段里 `<scope>provided</scope>`），动态解析用范围 `[26.3.build,)`，固定 build 用 `26.3.build.140-beta`。

文档说明：**两者并不等价**——`paper-api` 元数据的 `<release>` 是 `26.3-pre-2.build.0-alpha`，范围可能选中预发布，文档没有就此给出警告。
## paper-api 还是 paperweight

只有要访问 NMS / 服务器内部类时才需要 `io.papermc.paperweight.userdev`：
```kotlin title="build.gradle.kts"
plugins {
  id("io.papermc.paperweight.userdev") version "2.0.0-beta.24"
}
dependencies {
  paperweight.paperDevBundle("26.3.build.+")
}
```
SNAPSHOT 版需在 `settings.gradle.kts` 的 `pluginManagement.repositories` 里额外加 Paper 仓库。

| 事实 | 依据 |
|---|---|
| 唯一受支持的内部代码访问方式 | "This is the only supported way of accessing server internals, as redistributing the server JAR is against the Minecraft EULA..." |
| 用 dev bundle 就**删掉** `paper-api` | "You should remove any dependency on the Paper API, as the dev bundle includes that." |
| dev bundle 的底层坐标 | `io.papermc.paper:dev-bundle:<version>`（由 maven-metadata 交叉验证） |
| SNAPSHOT 版本只在 Paper 仓库 | "**paperweight-userdev** SNAPSHOT (pre-release) versions are only available through Paper's Maven repository." |
| 兼容范围 | 支持 Minecraft 1.17.1 及更新；"Only the latest version of `paperweight-userdev` is officially supported." |
| 26.1 起 reobfuscation 失效 | "From Minecraft version **26.1** onwards, Paper no longer supports obfuscated plugins"; "reobfuscated plugins will not work from Paper 26.1 onwards!" |
| NMS 不受官方支持 | "**PaperMC will offer no direct support for programming against Minecraft internals.**" |

`2.0.0-beta.24` 由 tag 推导（**非文档字面量**）。工具链不匹配导致 `paperweightUserdevSetup` 失败时，文档给的覆盖方式是 `paperweight { javaLauncher = javaToolchains.launcherFor { languageVersion = JavaLanguageVersion.of(17) } }`（示例场景是 Paper 1.17.1）。

### `PluginLoader` 名字冲突
- `io.papermc.paper.plugin.loader.PluginLoader`：Paper 的，`paper-plugin.yml` 的 `loader:` 用它，现行。
- `org.bukkit.plugin.PluginLoader` / `org.bukkit.plugin.java.JavaPluginLoader`：Bukkit 的，`@Deprecated(forRemoval = true)`。

两个不同接口，别因为都叫 "loader" 就混用。
## 本地测试服务器
官方 PaperMC 文档**没有任何 Gradle run task**，只外链第三方插件：

> "If you want to streamline the process of testing a plugin, you can use the [Run-Task](https://github.com/jpenilla/run-task) Gradle plugin."

下面这些出自 **run-paper / run-task（jpenilla）项目自己的 README，不是 PaperMC 文档**：
```kotlin
plugins { id("xyz.jpenilla.run-paper") version "VERSION" }
tasks {
  runServer {
    minecraftVersion("1.21.8")
  }
}
```
| 事实 | 值 |
|---|---|
| 插件 id | `xyz.jpenilla.run-paper` |
| 任务名 | `runServer` |
| 除应用插件外唯一必需配置 | `minecraftVersion("<mc version>")` |
| 插件 jar 注入 | 自动："Your plugin's jar (or shadowJar if present) will be used automatically." |
| 研究时点已发布版本 | `3.1.0`（Gradle Plugin Portal） |

远程调试：`java -agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=*:5005 -jar paper-26.3.jar --nogui`
## 不要做的事

- **不要凭记忆写版本号。** 文档页里的版本是构建期常量替换出来的；`1.21.11` 及更早用 `{VERSION}-R0.1-SNAPSHOT`，`26.1` 起用 `<version>.build.+`，混用必然解析失败。
- **不要把文档片段当成完整 `build.gradle.kts`。** 官方只打印 `repositories {}`、`dependencies {}`、`java { toolchain }` 三段，没有 `plugins {}`，没有 `group` / `version`。
- **不要去找官方推荐的 shadow / jar 任务。** 抓到的页面里没有；文档只在 `libraries:` 处说它 "removes the need to shade and relocate"，以及 paperweight 会自动检测 shadow 插件。
- **不要同时依赖 `paper-api` 和 `paperweight.paperDevBundle`。**
- **不要用 reobfuscation。** `26.1` 起已失效（"reobfuscated plugins will not work from Paper 26.1 onwards!"）。
- **不要用 `-headless` 后缀的 JDK。** 文档原话："Those variants miss required dependencies for Paper."
## 来源

- [Paper 项目搭建与依赖](https://docs.papermc.io/paper/dev/project-setup/index.md)
- [paperweight-userdev](https://docs.papermc.io/paper/dev/userdev/index.md)
- [Minecraft internals](https://docs.papermc.io/paper/dev/internals/index.md)
- [Paper 服务端入门与 Java 要求](https://docs.papermc.io/paper/getting-started/index.md)
- [Java 25 安装](https://docs.papermc.io/misc/java-install/index.md)
- [系统属性参考](https://docs.papermc.io/paper/reference/system-properties/index.md)
- [paper-plugin.yml 与依赖声明](https://docs.papermc.io/paper/dev/getting-started/paper-plugins/index.md)
- [下载 API：projects/paper](https://fill.papermc.io/v3/projects/paper)
- [paper-api maven-metadata.xml](https://repo.papermc.io/repository/maven-public/io/papermc/paper/paper-api/maven-metadata.xml)
- [dev-bundle maven-metadata.xml](https://repo.papermc.io/repository/maven-public/io/papermc/paper/dev-bundle/maven-metadata.xml)
- [JavaPluginLoader javadoc](https://jd.papermc.io/paper/26.3/org/bukkit/plugin/java/JavaPluginLoader.html)
- [run-task README（第三方）](https://raw.githubusercontent.com/jpenilla/run-task/master/README.md)
- [xyz.jpenilla.run-paper（第三方）](https://plugins.gradle.org/plugin/xyz.jpenilla.run-paper)
