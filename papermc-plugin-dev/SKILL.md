---
name: papermc-plugin-dev
description: 为 Minecraft Java 版 Paper 服务端编写、修改、调试 Java 插件的工作方法：确定版本基线、Gradle 工程与依赖坐标、plugin.yml 与 paper-plugin.yml、Bootstrap 与 Loader、Brigadier 命令、Adventure 文本与界面、事件、调度与 Folia 兼容、配置与持久化，以及 build → runServer 的实测闭环。内含官方文档地图与"查不到就别说"的传闻清单。
whenToUse: 当任务涉及 Paper / Spigot / Bukkit 服务端插件时加载——读写 Java 源码、build.gradle.kts、pom.xml、plugin.yml、paper-plugin.yml、命令、监听器、配置、调度任务，或排查插件加载失败、启动报错、行为异常。
---

# PaperMC 插件开发

Paper 迭代很快，同一个 API 在不同版本间会改名、改签名、改语义。这个技能的重点不是背 API，而是**用可验证的方式确定 API，然后真的把插件跑起来**。

## 0. 流程

1. **读工程**——先看清仓库已经怎么搭的，再决定怎么加。
2. **定基线**——Minecraft/Paper 版本、Java 版本、构建系统、描述符风格。一次定清楚，全程遵循。
3. **核实 API**——只写你确认过存在的东西（见第 2 节）。
4. **编译**——`.\gradlew.bat build`。不编译通过不算写完。
5. **实测**——能起服就起一次，看日志里插件有没有 enable 成功。

## 1. 先确认基线（不要跳）

```powershell
Get-ChildItem -Recurse -Depth 1 -Include build.gradle.kts,build.gradle,pom.xml,settings.gradle.kts
Get-Content gradle.properties -ErrorAction SilentlyContinue
Get-ChildItem src/main/resources -Filter *.yml | Select-Object Name
Get-Content src/main/resources/plugin.yml,src/main/resources/paper-plugin.yml -ErrorAction SilentlyContinue
```

判据：

| 观察到 | 含义 | 你遵守的写法 |
|---|---|---|
| 有 `paper-plugin.yml` | Paper 插件 | `bootstrapper:` / `loader:` 走 YAML FQCN；命令走 Brigadier；**不要写 `commands:` 段** |
| 只有 `plugin.yml`，无 `paper-plugin-loader` | 传统 Bukkit 风格 | `commands:` 声明命令，主类里 `getCommand(...).setExecutor(...)` |
| `plugin.yml` 含 `paper-plugin-loader` | 混合 | 保持现状 |
| `compileOnly("io.papermc.paper:paper-api:...")` | 标准 | 不要改成 `implementation` |
| `io.papermc.paperweight.userdev` | 用了服务端内部类 | 改动前先读它自己的文档 |

**版本号一律从工程里读。** 两种版本串都存在，别互相替换：`26.1`（即 `1.21.11`）及更早是 `{VERSION}-R0.1-SNAPSHOT`；之后是 `{VERSION}.build.+`。要查当前版本：`Invoke-RestMethod https://fill.papermc.io/v3/projects/paper`。

Java 版本跟着基线走：当前 Paper 要求 **Java 25**，`1.20`–`1.21.11` 是 Java 21。版本不匹配是启动失败最常见的原因之一。

细节：`references/01-versions-and-build.md`、`references/02-descriptors.md`。

## 2. 怎么核实一个 API

**不确定就不要写。** 四种手段，从最可靠排起：

```powershell
# 1) 读你编译时用的那个 jar —— 结论与编译结果一致
$jar = Get-ChildItem "$env:USERPROFILE\.gradle\caches\modules-2\files-2.1\io.papermc.paper" -Recurse -Filter "paper-api-*.jar" |
       Where-Object { $_.Name -notmatch 'sources|javadoc' } | Select-Object -First 1
javap -classpath $jar.FullName io.papermc.paper.plugin.lifecycle.event.types.LifecycleEvents

# 2) 读仓库里已有的同类实现（最贴近本工程的约定）

# 3) 官方文档 —— HTML 抓不到正文，URL 后加 /index.md 取原始 Markdown
Invoke-RestMethod https://docs.papermc.io/paper/dev/plugin-yml/index.md
Invoke-RestMethod https://docs.papermc.io/llms.txt     # 全站索引

# 4) Javadoc：https://jd.papermc.io/paper/（选对版本）
```

文档 Markdown 源里形如 `\{CONSTANT}` 的占位符是构建期才替换的，**抓到的原文里不是真实版本号**。

## 3. 常用 API 最小形态

### 命令（Brigadier，现代做法）

```java
this.getLifecycleManager().registerEventHandler(LifecycleEvents.COMMANDS, event -> {
    event.registrar().register(
        Commands.literal("mycmd")
            .requires(src -> src.getSender().hasPermission("myplugin.command"))
            .then(Commands.literal("reload").executes(ctx -> {
                reloadConfig();
                ctx.getSource().getSender().sendMessage(Component.text("已重载"));
                return Command.SINGLE_SUCCESS;
            }))
            .build(),
        "MyPlugin 主命令", List.of("mc"));
});
```

- 在 `onEnable()` 或 `PluginBootstrap.bootstrap(...)` 里注册；重载后会自动重新注册。
- `LifecycleEvents` 只有 `COMMANDS`、`TAGS`、`DATAPACK_DISCOVERY` 三个常量。
- 传统 `CommandExecutor` / `TabCompleter` / `onCommand` 已 `@Obsolete(since="26.3")`，新代码不要用。

细节：`references/03-commands.md`。

### 事件

```java
public final class MyListener implements Listener {
    @EventHandler(priority = EventPriority.NORMAL, ignoreCancelled = true)
    public void onBlockBreak(BlockBreakEvent event) { /* 只做必要判断 */ }
}
// 注册：getServer().getPluginManager().registerEvents(new MyListener(this), this);
```

优先级 `LOWEST → LOW → NORMAL → HIGH → HIGHEST → MONITOR`，越高越晚调用；`MONITOR` 只观察不改状态。**没有 `AsyncEvent` 接口**，异步性看 `Event#isAsynchronous()`。

### 调度与线程

```java
getServer().getScheduler().runTaskLater(this, () -> { /* 主线程 */ }, 20L);
getServer().getScheduler().runTaskAsynchronously(this, () -> {
    var data = loadFromDisk();                                    // 只做 IO/计算
    getServer().getScheduler().runTask(this, () -> apply(data));  // 回主线程碰游戏状态
});
```

1 tick = 50ms。**异步里不要碰世界、方块、实体、玩家、背包、计分板。**
要兼容 Folia 就换成 `getGlobalRegionScheduler()` / `getRegionScheduler()` / `getAsyncScheduler()` / `entity.getScheduler()`——它们在普通 Paper 上同样可用。

细节：`references/04-events-and-scheduling.md`。

### 文本与界面

```java
sender.sendMessage(Component.text("Hello ", NamedTextColor.GRAY)
    .append(Component.text(player.getName(), NamedTextColor.AQUA)));
Component msg = MiniMessage.miniMessage().deserialize("<green>完成</green> <gray>5s");
```

**MiniMessage 是标记语言，玩家可控的字符串不能直接喂进去**，否则能注入样式甚至点击事件：用 `Placeholder.unparsed(...)`，或 `MiniMessage.Preset.NON_INTERACTABLE`。实例建一次、复用。
`ChatColor` 已弃用；`sendMessage(String)` 已 `@ApiStatus.Obsolete`。

细节：`references/05-text-and-ui.md`。

### 配置

```java
saveDefaultConfig();                              // 释放 resources/config.yml
int limit = getConfig().getInt("limits.max", 10); // 永远给默认值
getConfig().set("limits.max", 20);
saveConfig();
```

读取要容错（缺键、非法值不能 NPE）。玩家数据优先 PDC 或 `ConfigurationSerializable`。

细节：`references/06-config-and-data.md`。

## 4. 改现有工程的规矩

- 不动无关代码：不重排 import、不换风格、不升依赖版本。
- 不造第二套模式：已有 `ConfigManager` 就别再写一个。
- 包结构与命名跟既有代码一致；注释与用户可见文案沿用工程语言。
- 新增配置项要同步默认资源并给默认值，且老配置缺键时不 NPE。
- 改公共行为前先搜引用，确认影响面。

## 5. 反模式（完整表见 `references/08-pitfalls-and-sources.md`）

| 反模式 | 后果 |
|---|---|
| 凭记忆写 API / 报版本号 | 编译不过、运行期 `NoSuchMethodError` |
| 异步线程里操作游戏状态 | 随机崩溃、数据损坏 |
| `onEnable` 里做阻塞 IO | 启动卡死、watchdog 杀服 |
| `onDisable` 不取消任务 | 重载后任务翻倍执行 |
| 只处理 `InventoryClickEvent` | 拖拽走 `InventoryDragEvent`，逻辑漏掉 |
| 同时用 `plugin.yml commands:` 和 Brigadier 注册同名命令 | 冲突 |

## 6. 参考文件（按需读，不要一次全读）

| 文件 | 内容 |
|---|---|
| `references/01-versions-and-build.md` | 版本基线、Java 版本、依赖坐标、paper-api vs paperweight、测试服务器 |
| `references/02-descriptors.md` | `plugin.yml` / `paper-plugin.yml` 逐字段表、`dependencies` 结构、Bootstrap 与 Loader |
| `references/03-commands.md` | Brigadier 注册与命令树、参数、权限、描述符与命令的关系 |
| `references/04-events-and-scheduling.md` | 事件与优先级、同步/异步规则、`BukkitScheduler` 签名、Folia 调度、生命周期 |
| `references/05-text-and-ui.md` | Adventure `Component`、Adventure 5.x 变化、MiniMessage 安全、物品、GUI 正确性 |
| `references/06-config-and-data.md` | 配置文件、序列化、持久化选型、PDC、线程与 IO |
| `references/07-build-run-debug.md` | 构建、runServer、日志分诊、`javap` 核实、文档抓取姿势 |
| `references/08-pitfalls-and-sources.md` | 反模式全表、官方文档地图、"传闻清单"、文档自相矛盾处 |
