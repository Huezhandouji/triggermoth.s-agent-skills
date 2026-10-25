# PaperMC 插件描述符：plugin.yml 与 paper-plugin.yml
## 两套描述符
| | `plugin.yml` | `paper-plugin.yml` |
|---|---|---|
| 用途 | Bukkit-style，入口 `main:` | Paper-style，另有 `bootstrapper:` / `loader:`；只有 Paper 的 Plugin Loader 能加载，可用 Bukkit 没有的 API |
| 依赖 | 扁平列表 `depend` / `softdepend` / `loadbefore` / `provides` | `dependencies.bootstrap` / `dependencies.server` 映射 |
| 命令 | `commands:` 字段 | **不使用** `commands:`，走 Brigadier Command API |
| 加载顺序 | 由依赖字段决定 | **不**由 `dependencies` 决定，靠 `load: BEFORE/AFTER/OMIT` |
| 类加载 | 常规 | 隔离；要 `join-classpath: true` 才能访问别人的 classpath |
| 文档状态 | 稳定 | 标为 *Experimental* |

规则：**项目里已经有哪个，就跟着哪个写。** 官方说它 "will not act as a drop-in replacement for `plugin.yml`"，但允许同一个 JAR 里两个都放。
## plugin.yml 字段表
位置：`src/main/resources/plugin.yml`。文档用 `*` 标必填（"If they have an asterisk (\*) next to them, that means they are required."）。

| 字段 | 必填 | 含义 |
|---|---|---|
| `name` | 是 | 插件名，显示在插件列表与日志中；设了 `prefix` 后日志里被它覆盖 |
| `version` | 是 | 插件版本，显示在插件信息与服务器日志中 |
| `main` | 是 | 主类 FQCN，必须 extends `JavaPlugin` |
| `author` / `authors` | 否 | 单个作者字符串，或作者列表 |
| `description` / `contributors` / `website` | 否 | 插件描述；非主要作者的贡献者列表；网站 / 仓库链接 |
| `api-version` | 否 | Paper API 版本，有效值 `1.13` – `26.3`。**不写会按 legacy 插件加载并打印警告**；比服务器新则服务器拒绝加载 |
| `load` | 否 | `STARTUP` 或 `POSTWORLD`，默认 `POSTWORLD` |
| `prefix` | 否 | 日志中代替插件名显示的字符串 |
| `libraries` | 否 | Maven 坐标列表，从中央仓库下载并加入 classpath；文档称其 "removes the need to shade and relocate" |
| `permissions` | 否 | 权限节点映射；子键 `description` / `default` / `children` |
| `default-permission` | 否 | 未指定 `default` 的权限节点的默认值：`op` / `notop` / `true` / `false`，默认 `op` |
| `commands` | 否 | 命令映射；子键 `description` / `usage` / `aliases` / `permission` / `permission-message` |
| `depend` | 否 | 加载期依赖的插件名列表；**插件不存在则本插件不加载** |
| `softdepend` | 否 | 只影响完整功能的依赖 |
| `loadbefore` | 否 | 本插件应早于其加载的插件列表 |
| `provides` | 否 | 声明本插件提供某插件的功能；`(soft)depend` 它的插件会把本插件视为其存在 |
| `folia-supported` | 否 | `true` 才允许在 Folia 上运行；文档强调这远不足以真正支持 Folia |
| `paper-plugin-loader` | 否 | Paper plugin loader 类的 FQCN，**Experimental** |
| `paper-skip-libraries` | 否 | `true` 则跳过 `libraries` 解析（便于把库加载交给 loader），默认 `false`，**Experimental** |

`bootstrapper` / `loader` **不是** `plugin.yml` 的字段，只在 `paper-plugin.yml` 里用。子键语义：`permissions.default` 缺省取 `default-permission`（最终默认 `op`），`children` 的 `true` 表示继承父权限；`commands.usage` 是 `/help <command>` 显示的内容，`aliases` 用于缩短命令，玩家只能看到自己有 `permission` 的命令。
## paper-plugin.yml 字段表
文档**没有**独立的 `paper-plugin.yml` 参考页，内容在 getting-started/paper-plugins 与 paper/reference/paper-plugins。

| 字段 | 必填 | 含义 |
|---|---|---|
| `name` / `version` / `main` | 是 | 与 `plugin.yml` 相同 |
| `description` / `api-version` / `folia-supported` | 否 | 与 `plugin.yml` 相同 |
| `bootstrapper` | 否 | 实现 `io.papermc.paper.plugin.bootstrap.PluginBootstrap` 的 FQCN |
| `loader` | 否 | 实现 `io.papermc.paper.plugin.loader.PluginLoader` 的 FQCN |
| `dependencies.bootstrap` / `.server` | 否 | bootstrapper 阶段 / 运行期需要的依赖 |

**`paper-plugin.yml` 没有 `commands:` 字段：**
> "Paper plugins do not use the `commands` field to register commands... Instead, you can register commands using the Brigadier Command API."

（`defaults` 字段在抓到的页面里都不存在 —— **未证实／很可能不存在**；`PluginMeta` 只有 `getPermissionDefault()`。）
## dependencies 结构
```yml
dependencies:
  bootstrap:
    RegistryPlugin:
      load: BEFORE
      required: true
      join-classpath: true
  server:
    RequiredPlugin:
      load: AFTER
      required: true
      join-classpath: false
```
| 键 | 取值 | 默认值 | 含义 |
|---|---|---|---|
| `load` | `BEFORE` / `AFTER` / `OMIT` | `OMIT` | 相对**本插件**的加载先后；`OMIT` 的顺序行为未定义 |
| `required` | Boolean | `true` | 该插件是否是本插件加载所必需 |
| `join-classpath` | Boolean | `true` | 本插件是否可访问对方的 classpath（用于直接操作其内部） |

- `bootstrap` 段只给 bootstrapper 用，`server` 段给运行期用；同一个插件可以两段都写；与 `plugin.yml` 的关键差异："Paper plugins do **not** use the `dependencies` field to determine load order."；循环加载**不会**被自动解决：服务器报 `[LoadOrderTree] Circular plugin loading detected` 并退出；逃生开关 `-Dpaper.useLegacyPluginLoading=true`（文档标 Legacy）；`join-classpath: true` 只表示"你能访问别人"；别人的 classloader 仍不向你之外的人开放。
## Bootstrap 与 Loader
签名取自 `paper-api 26.3.build.140-beta` 的官方 javadoc（两个接口的类声明上只有 `@Experimental @NullMarked @OverrideOnly`）：
```java
// io.papermc.paper.plugin.bootstrap.PluginBootstrap
void bootstrap(BootstrapContext context);
default JavaPlugin createPlugin(PluginProviderContext context);
// io.papermc.paper.plugin.loader.PluginLoader
void classloader(PluginClasspathBuilder classpathBuilder);
// io.papermc.paper.plugin.loader.PluginClasspathBuilder
@Contract("_ -> this") PluginClasspathBuilder addLibrary(ClassPathLibrary classPathLibrary);
PluginProviderContext getContext();
```
`PluginProviderContext` 访问器：`PluginMeta getConfiguration()`、`Path getDataDirectory()`、`ComponentLogger getLogger()`、`Path getPluginSource()`。`BootstrapContext extends PluginProviderContext, LifecycleEventOwner`，另有 `LifecycleEventManager<BootstrapContext> getLifecycleManager()`。

| 点 | 事实 |
|---|---|
| 运行阶段 | bootstrapper 在服务器加载前运行，**早于插件主类被创建** |
| 类加载器 | bootstrap 类与主类在同一个 classloader 上 |
| Bukkit 可用性 | "All calls to Bukkit may throw a NullPointerExceptions or return null unexpectedly." 只调用文档明确说可用于 bootstrapper 的 API |
| `createPlugin` | 服务器用它实例化主类，可改构造逻辑（如传入自定义参数） |
| loader 阶段 | 由**另一个 classloader** 调用，静态值不会保留到插件加载时 |

loader 只能加 `JarLibrary` 与 `MavenLibraryResolver`（`io.papermc.paper.plugin.loader.library.impl`）；后者构造时**不带任何仓库**，必须自己 `addRepository(...)`，并可用 `MAVEN_CENTRAL_DEFAULT_MIRROR`。
### 这些注解不存在
`@PluginBootstrap`、`@PluginLoader`、`@PluginFactory` **全都不存在**。装配方式**只有** `paper-plugin.yml` 里的 `bootstrapper:` / `loader:` 字符串 FQCN 字段——没有注解，也没有注册调用。`paper-api` 里也没有 `PluginProvider` 和裸的 `PluginClassLoader`：`io.papermc.paper.plugin.bootstrap` 只有 `BootstrapContext`、`PluginBootstrap`、`PluginProviderContext`。（相关 javadoc 页抓取失败，属**未证实**。）
## 最小示例
`src/main/resources/plugin.yml`：
```yaml
name: Example-Plugin
version: 1.0.0
main: com.example.paperplugin.ExamplePlugin
description: An example plugin
author: PaperMC
website: https://papermc.io
api-version: '26.3'
```
`src/main/resources/paper-plugin.yml`：
```yaml
name: Example-Plugin
version: '1.0'
main: com.example.paperplugin.ExamplePlugin
description: A Paper plugin showcase
api-version: '26.3'
bootstrapper: com.example.paperplugin.ExamplePluginBootstrap
loader: com.example.paperplugin.ExamplePluginLoader
```
## 来源

- [plugin.yml 全部字段](https://docs.papermc.io/paper/dev/plugin-yml/index.md) [Paper 插件入门：paper-plugin.yml / dependencies / 隔离](https://docs.papermc.io/paper/dev/getting-started/paper-plugins/index.md) [Paper 插件参考页：类加载与循环加载](https://docs.papermc.io/paper/reference/paper-plugins/index.md) [PluginBootstrap javadoc](https://jd.papermc.io/paper/26.3/io/papermc/paper/plugin/bootstrap/PluginBootstrap.html) [BootstrapContext javadoc](https://jd.papermc.io/paper/26.3/io/papermc/paper/plugin/bootstrap/BootstrapContext.html)
- [PluginProviderContext javadoc](https://jd.papermc.io/paper/26.3/io/papermc/paper/plugin/bootstrap/PluginProviderContext.html) [PluginLoader javadoc](https://jd.papermc.io/paper/26.3/io/papermc/paper/plugin/loader/PluginLoader.html) [PluginClasspathBuilder javadoc](https://jd.papermc.io/paper/26.3/io/papermc/paper/plugin/loader/PluginClasspathBuilder.html) [Folia 支持（folia-supported）](https://docs.papermc.io/paper/dev/folia-support/index.md)
