# Paper 事件、调度与生命周期参考（paper-api 26.3）
签名取自官方 Javadoc（paper-api 26.3.build.140-beta），引文取自 docs.papermc.io。

## 事件监听
监听器实现 `org.bukkit.event.Listener`；处理方法是标注 `@EventHandler`、返回 `void`、只接收一个事件参数的方法。
```java
public class ExampleListener implements Listener {
    @EventHandler public void onPlayerMove(PlayerMoveEvent event) { /* ... */ }
}
```
- 注册：`this.getServer().getPluginManager().registerEvents(new ExampleListener(), this);`，文档建议放在 `onEnable()`；事件必须带静态 `getHandlerList` 方法才能被监听。
- `@EventHandler` 默认 `priority = EventPriority.NORMAL`、`ignoreCancelled = false`；只有 `ignoreCancelled = true` 且事件已被取消时，该方法才被跳过，否则总是调用。
- 实现 `Cancellable` 的事件可 `event.setCancelled(true)`；别的插件可能已取消或改过它，处理前先检查。
- 注销走 `HandlerList`：`unregister(Plugin)`、`unregister(Listener)`、`unregister(RegisteredListener)`、`unregisterAll()`、静态 `unregisterAll(Plugin)` / `unregisterAll(Listener)`。
- **`PluginManager#unregisterEvents(...)` 不存在**；按插件清理用 `HandlerList.unregisterAll(Plugin)`，或依赖插件 disable 时的自动清理。
## 事件优先级
| 顺序 | 常量 |
|---|---|
| 1 | `EventPriority.LOWEST` |
| 2 | `EventPriority.LOW` |
| 3 | `EventPriority.NORMAL`（`@EventHandler` 默认） |
| 4 | `EventPriority.HIGH` |
| 5 | `EventPriority.HIGHEST` |
| 6 | `EventPriority.MONITOR` |

- 低优先级先调用、高优先级后调用，顺序即上表 1→6；**优先级越高越晚执行**，要有最后发言权就用 `HIGHEST`。
- `MONITOR` 只在最后调用、用来看结果，原文："No modifications to the event should be made under this priority"。**绝不要在 `MONITOR` 下修改状态。**
## 同步与异步事件
- **不存在 `AsyncEvent` 接口**：`org.bukkit.event` 包（26.3）只有 `Cancellable`、`Event`、`Event.Result`、`EventException`、`EventHandler`、`EventPriority`、`HandlerList`、`Listener`。
- 异步性是 `Event` 自身的属性：`Event()` 假定同步，`Event(boolean isAsync)` 显式声明，`public final boolean isAsynchronous()` 查询。
- `isAsynchronous()` 列出的注意事项：
  - 异步事件**永远不会**从同步事件触发的代码里被 fire，否则抛 `IllegalStateException`；反过来，异步处理器可以 fire 同步或异步事件。
  - 事件可能被同时、以任意顺序触发多次；事件开始执行后，新注册或注销的处理器都被忽略。
  - 处理器可以阻塞任意时长。
- **硬规则**：异步上下文不得触碰 Bukkit 游戏状态。调度页原文："Large portions of the Bukkit API are not safe to use from within asynchronous tasks"；聊天页：在异步上下文用 Bukkit API 不安全、可能抛异常。
## BukkitScheduler 签名表
`BukkitScheduler scheduler = this.getServer().getScheduler();`。1 tick = 50 ms，20 ticks = 1 s。下表方法都以 `Plugin plugin` 开头，`...` 是任务参数（`Runnable` 或 `Consumer<? super BukkitTask>`）。

| 方法 |
|---|
| `runTask(Plugin, ...)` |
| `runTaskAsynchronously(Plugin, ...)` |
| `runTaskLater(Plugin, ..., long delay)` |
| `runTaskLaterAsynchronously(Plugin, ..., long delay)` |
| `runTaskTimer(Plugin, ..., long delay, long period)` |
| `runTaskTimerAsynchronously(Plugin, ..., long delay, long period)` |

> **坑**：`Consumer<BukkitTask>` 重载返回 **`void`**，只有 `Runnable` 重载返回 `@NotNull BukkitTask`；文档示例用 `Consumer` 重载（`runTaskTimer(plugin, task, 0, 20)`），任务用传进来的 `BukkitTask` 自我取消。

| 返回类型 | 其余方法 |
|---|---|
| `Future<T>` | `callSyncMethod(Plugin, Callable<T>)` |
| `void` | `cancelTask(int)`、`cancelTasks(Plugin)` |
| `boolean` | `isCurrentlyRunning(int)`、`isQueued(int)` |
| `List<BukkitWorker>` / `List<BukkitTask>` | `getActiveWorkers()` / `getPendingTasks()` |
| `java.util.concurrent.Executor` | `getMainThreadExecutor(Plugin)` |

- `getMainThreadExecutor` 返回 **`java.util.concurrent.Executor`**；26.3 **没有** `MainThreadExecutor` 类；`@Deprecated`：6 个 `BukkitRunnable` 重载（`since="1.7.10"`，方法名同上、末参换成 `BukkitRunnable`）；遗留 `int` 系列 `schedule*Task` 中 `scheduleAsync*` 也已废弃；`BukkitTask`：`getTaskId()`、`getOwner()`、`isSync()`、`isCancelled()`、`cancel()`；26.3 **没有** `isRunning()`；`runTask` 下一个 tick 执行，`runTaskLater` 延迟指定 tick 数，`runTaskTimer` 反复运行直到被取消；`cancelTasks(Plugin)` 移除该插件的所有任务；**delay = 0 即下一个 tick**；原文警告：服务器正在启动或尚未 enable 时用 0 tick 延迟调度，任务会在服务器 enable 之前就执行。
## Folia 与区域化调度
```yaml
folia-supported: true
```
没有这个字段，插件在 Folia 上根本不会运行。警告原文："**Marking a plugin as supported on Folia is *not nearly enough*.**"

| 获取方式 | 用途 |
|---|---|
| `server.getGlobalRegionScheduler()` | 不属于任何区域的任务：游戏规则、时间、天气、控制台命令 |
| `server.getRegionScheduler()` | 某 location 所属区域的任务（传 `World`+chunk 坐标或 `Location`） |
| `server.getAsyncScheduler()` | 异步任务，时间参数用 **`TimeUnit` 而不是 tick** |
| `entity.getScheduler()` | 实体的任务，会随实体跨区域移动 |

- **关键可移植性事实**："If you use these schedulers when running Paper, they will be internally handled to provide the same functionality as if you were running Folia." 这套调度器在普通 Paper 上同样可用。
- 调度方法返回 `ScheduledTask`（**与 `BukkitTask` 不同**）：`getOwningPlugin()`、`isRepeatingTask()`、`cancel()`、`getExecutionState()`、默认 `isCancelled()`。
- 前两个调度器用 `execute`、`run`、`runDelayed`、`runAtFixedRate`；`AsyncScheduler` 用 `runNow`、`runDelayed`、`runAtFixedRate` + `cancelTasks(Plugin)`。
- **绝不要用 `RegionScheduler` 操作实体**。Javadoc 原文："It is entirely inappropriate to use the region scheduler to schedule tasks for entities."

Folia 上会坏掉的东西（官方文档明确记载的）：
- `BukkitScheduler` 是**非 Folia** 的："This guide is designed for non-Folia Bukkit servers. If you are using Folia, you should use its respective schedulers."
- 没有全局主线程与全局 tick 计数器："**In Folia, the Current Tick is maintained per region.**"，区域 "tick independently and in parallel"。
- 跨区域访问必须被调度：跨区域操作面对 "completely unknown state" 的区域，"**the call to teleport/place an entity must be invoked on the region owning the entity**"。
- Folia 禁用一批 vanilla 命令，FAQ 原列表含 Bossbar、Clone、Data、Datapack、Debug、Function、Item、Loot、**Reload**、Return、Ride、Rotate、Schedule、Scoreboard、Spectate、SpreadPlayers、Tag、Team、TeamMsg、Tick、Trigger、Perf、SaveAll、Restart，其中有 **`/reload`**。
## 生命周期
| 阶段 | 能做什么 |
|---|---|
| 构造器 | ⚠ 不要在这里写代码，也不要直接调用插件构造器 |
| `onLoad()` | "Most of the Bukkit API is not available at this point, so it is not safe to interact with it." |
| `onEnable()` | 注册监听器、开数据库连接、起线程；服务器还没开始 tick |
| `onDisable()` | 保存数据、关闭连接等清理 |

顺序：构造器 → `onLoad()` → `onEnable()` → … → `onDisable()`。`JavaPlugin#onLoad` Javadoc 保证：多个插件加载时，所有插件的 `onLoad()` 都在任何 `onEnable()` 之前被调用。

`LifecycleEvents`（`io.papermc.paper.plugin.lifecycle.event.types`）**只有三个常量**：

| 常量 | 可注册位置 |
|---|---|
| `COMMANDS` | `bootstrap(BootstrapContext)` 或 `JavaPlugin.onEnable()` |
| `TAGS`（`@Experimental`） | **只能** `bootstrap` |
| `DATAPACK_DISCOVERY`（`@Experimental`） | **只能** `bootstrap` |

- **没有 `LifecycleEvents.REGISTRIES`**：注册表改动走 `io.papermc.paper.registry.event.RegistryEvents`（`.compose()`、`.entryAdd()`），只在 bootstrap 阶段可用；这一阶段抛异常会导致服务器在加载前关闭。
- 用 `LifecycleEventManager` 注册的东西会在需要时被重新注册，不必自己处理 `/reload`。
- **注册某些 lifecycle handler 会禁用插件重载**："The reloading plugins functionality (via `/bukkit:reload` or `Server#reload()`) is disabled if plugins register handlers in certain situations."
- 资源重载会丢掉插件内容：`ServerResourcesReloadedEvent`（`io.papermc.paper.event.server`）用于在 `/minecraft:reload` 后重新注册可能丢失的自定义配方与进度；`Cause` 只有 `COMMAND` 与 `PLUGIN`。
- **`ReloadCommandEvent` 在 26.3 不存在**：`com.destroystokyo.paper.event.server` 里只有 `AsyncTabCompleteEvent`、`GS4QueryEvent`、`PaperServerListPingEvent`、`ServerExceptionEvent`、`ServerTickEndEvent`、`ServerTickStartEvent`、`WhitelistToggleEvent`。
## 来源

- https://docs.papermc.io/paper/dev/event-listeners/index.md https://docs.papermc.io/paper/dev/chat-events/index.md https://docs.papermc.io/paper/dev/scheduler/index.md
- https://docs.papermc.io/paper/dev/lifecycle/index.md https://docs.papermc.io/paper/dev/folia-support/index.md https://docs.papermc.io/folia/faq/index.md
- https://docs.papermc.io/folia/reference/overview/index.md https://jd.papermc.io/paper/26.3/org/bukkit/scheduler/BukkitScheduler.html https://jd.papermc.io/paper/26.3/io/papermc/paper/plugin/lifecycle/event/types/LifecycleEvents.html
