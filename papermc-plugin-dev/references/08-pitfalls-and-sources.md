# 反模式、文档地图与"传闻"清单

## 反模式

| 反模式 | 后果 | 正确做法 |
|---|---|---|
| 凭记忆写 API | 编译不过 / 运行期 `NoSuchMethodError` | `javap` 读 jar、读仓库同类代码、查 Javadoc |
| 从记忆里报版本号 | 换版本就炸 | 从工程基线读；查 `fill.papermc.io/v3` |
| 把 `paper-api` 写成 `implementation` | 运行时和服务器自带的类冲突 | `compileOnly`（dev bundle 则完全去掉它） |
| 在构造函数里写逻辑 | 构造早于加载、状态不完整 | 放到 `onLoad` / `onEnable`；**不要自己 `new` 插件主类** |
| 在 `onLoad` 里调用 Bukkit API | 此时大部分 API 不可用 | 移到 `onEnable` |
| 异步任务里操作世界/实体/玩家/背包 | 随机崩溃、数据损坏 | 调度器回主线程（`04-events-and-scheduling.md`） |
| `onEnable` 里做阻塞 IO | 启动卡死、可能被 watchdog 杀 | 异步加载，完成后再启用功能 |
| `onDisable` 不取消任务 | 重载后任务翻倍执行 | 持有 `BukkitTask`/`ScheduledTask` 并在 `onDisable` 里 `cancel()` |
| 用 `ChatColor` / 字符串拼消息 | 与新版 API 脱节，做不出交互文本 | Adventure `Component` / MiniMessage |
| 把玩家输入直接喂给 MiniMessage | 玩家可注入样式甚至点击事件 | `Placeholder.unparsed` 或 `NON_INTERACTABLE` |
| 在 `InventoryClickEvent` 里开/关界面 | 未定义行为 | 取消事件并延后一 tick 再开 |
| 只处理 `InventoryClickEvent` | 拖拽物品完全不触发它 | 另加 `InventoryDragEvent` |
| 在 `MONITOR` 优先级改状态 | 破坏其它插件的前提 | `MONITOR` 只用来观察 |
| 同时用 `plugin.yml commands:` 和 Brigadier 注册同名命令 | 重复注册 / 冲突 | 二选一；`paper-plugin.yml` 根本不要写 `commands:` |
| 顺手重构无关代码 | review 成本、引入回归 | 只改任务范围内的东西 |

## 官方文档地图

**URL 会变。** 下面这些是核实过的当前路径；抓 HTML 会只拿到标题，所以**一律加 `/index.md` 取原始 Markdown**。

| 主题 | 路径（`https://docs.papermc.io` 之后） |
|---|---|
| 工程搭建 / 依赖 | `/paper/dev/project-setup/` |
| Paper 插件与 `paper-plugin.yml` | `/paper/dev/getting-started/paper-plugins/` |
| `plugin.yml` 字段 | `/paper/dev/plugin-yml/` |
| 命令（Brigadier） | `/paper/dev/command-api/basics/{introduction,registration,executors,requirements,command-tree,arguments-and-literals,argument-suggestions,custom-arguments}/` |
| 命令（其它） | `/paper/dev/command-api/misc/{basic-command,comparison-bukkit-brigadier}/`、`/paper/dev/command-api/arguments/minecraft/` |
| 事件 | `/paper/dev/event-listeners/`、`/paper/dev/handler-lists/`、`/paper/dev/custom-events/` |
| 生命周期 | `/paper/dev/lifecycle/` |
| 调度 | `/paper/dev/scheduler/` |
| Folia 兼容 | `/paper/dev/folia-support/` |
| 文本（Component） | `/paper/dev/component-api/{introduction,audiences,i18n}/` |
| 聊天事件 | `/paper/dev/chat-events/` |
| 界面 | `/paper/dev/custom-inventory-holder/`、`/paper/dev/menu-type-api/` |
| 配置 | `/paper/dev/plugin-configurations/` |
| 持久化数据（PDC） | `/paper/dev/pdc/` |
| 数据库 | `/paper/dev/using-databases/` |
| 页面总索引 | `/llms.txt` |

已确认**不存在**的路径（不要再抓）：`/paper/dev/commands/`、`/paper/dev/command-api/`、`/paper/dev/event-api/`、`/paper/dev/events/`、`/paper/dev/api/plugins/`、`/paper/dev/plugin-yml-legacy/`、`/paper/dev/plugin-loader/`、`/paper/dev/paper-plugin-yml/`、`/paper/dev/inventory/`、`/paper/dev/configuration/`、`/paper/dev/itemstack/`。

其它权威来源：

- Paper API Javadoc：<https://jd.papermc.io/paper/>（选对版本）
- 版本与构建：<https://fill.papermc.io/v3/projects/paper>
- 源码与变更：<https://github.com/PaperMC/Paper>
- Adventure：<https://docs.advntr.dev/>

## "传闻"清单——不要照着写

以下名字流传很广，但**在当前官方文档与 Javadoc 里查不到**（有的是根本不存在）。写到代码里之前必须自己核实：

| 传闻 | 事实 |
|---|---|
| `@PluginBootstrap` / `@PluginLoader` / `@PluginFactory` 注解 | **不存在**。Bootstrap/Loader 只靠 YAML 里的 FQCN 字符串字段接线 |
| `AsyncEvent` 接口 | **不存在**。异步性来自 `Event(boolean isAsync)` / `isAsynchronous()` |
| `LifecycleEvents.REGISTRIES` | **不存在**。`LifecycleEvents` 只有 `COMMANDS`、`TAGS`、`DATAPACK_DISCOVERY`；注册表改动走 `RegistryEvents` |
| `ReloadCommandEvent` | 26.3 里**没有**。资源重载用 `ServerResourcesReloadedEvent` |
| `MainThreadExecutor` 类 | **不存在**。`getMainThreadExecutor(Plugin)` 返回 `java.util.concurrent.Executor` |
| `PluginProvider`、裸 `PluginClassLoader` | **不在 `paper-api`**；`org.bukkit.plugin.java.JavaPluginLoader` 已 `@Deprecated(forRemoval=true)` |
| `@Command` 注解 | 文档里查不到 |
| `BossBarManager` | 官方文档里查不到 |
| `InventoryCloseEvent` | 官方文档里查不到（Javadoc 里是否存在要自行确认） |
| `Bukkit.createInventory(holder, size, Component)` | 官方文档**没有**这个教程/签名，需自行核实 |
| `ItemStack#translatable` | **不存在** |
| "插件分享出去的 jar 不能带 shaded 库" | 官方文档**没有**这条明确警告，属二手经验 |
| "`getCommand` 已弃用" | 文档没说弃用；实际是 `@Obsolete(since="26.3")`，且 Paper 插件在 `onEnable()` 调它会抛 `UnsupportedOperationException` |
| "不要把插件状态存成 static" | 官方文档里**查不到**——是社区经验，不是官方规则 |
| "`addRecipe` 已弃用" | 官方文档里**查不到** |

判定标准很简单：**文档/Javadoc 里找不到的，就不要当事实写进代码或结论；要么核实，要么标成不确定。**

## 已知的文档自相矛盾（别被带偏）

官方文档之间本身有冲突，遇到时以「能编译 + 能实测」为准：

- 内部实现页仍有 `26.3-R0.1-SNAPSHOT` 这种旧版本串，而 `project-setup` 明确说该格式止于 `26.1`。
- `PluginMeta#getAPIVersion()` 的 Javadoc 说小版本被归并到主版本，与 `plugin.yml` 文档里"1.20.5 起支持小版本"矛盾。
- Gradle 的 `26.3.build.+` 与 Maven 的 `[26.3.build,)` 解析结果并不一致（Maven 的 `release` 是个 alpha 预发布）。
- `project-setup` 里有个指向 `#plugin-remapping` 的锚点，该章节并不存在。
- "loader" 一词同时指 `io.papermc.paper.plugin.loader.PluginLoader`（Paper 的类路径加载器）和已被废弃的 `org.bukkit.plugin.java.JavaPluginLoader`（Bukkit 主类加载器）——看上下文。

## 来源

- <https://docs.papermc.io/paper/dev/getting-started/paper-plugins/>
- <https://docs.papermc.io/paper/dev/plugin-yml/>
- <https://docs.papermc.io/paper/dev/project-setup/>
- <https://docs.papermc.io/paper/dev/event-listeners/>
- <https://docs.papermc.io/paper/dev/scheduler/>
- <https://docs.papermc.io/paper/dev/folia-support/>
- <https://jd.papermc.io/paper/>
- <https://fill.papermc.io/v3/projects/paper>
