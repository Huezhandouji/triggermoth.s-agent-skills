# Paper 命令参考（paper-api 26.3）
签名取自官方 Javadoc（paper-api 26.3.build.140-beta），引文取自 docs.papermc.io。

## 先选一条路
| 路径 | 注册入口 | 状态 |
|---|---|---|
| Brigadier 命令树 | `LifecycleEvents.COMMANDS` + `Commands.literal` / `Commands.argument` | 现代、推荐；支持子命令、原生参数类型、客户端补全 |
| `BasicCommand` | `JavaPlugin#registerCommand(...)` 或 `event.registrar().register(label, cmd)` | 未废弃；文档称其为 Bukkit 命令的直接替代品，适合 `String[] args` 简单命令 |
| Legacy Bukkit | `CommandExecutor` / `TabCompleter` / `onCommand` / `onTabComplete` | 全部 `@Obsolete(since="26.3")` |

- `CommandExecutor`、`TabCompleter` 是**类级** `@Obsolete(since="26.3")`；`JavaPlugin` 继承的 `onCommand`、`onTabComplete`、`getCommand` 各自单独标注同一注解；官方开发者文档**已不再记载**这条路径：`/paper/dev/commands/` 是 404，`llms.txt` 的 `paper/dev` 索引里没有相关条目。Javadoc 的 API Note 建议改用 Brigadier，需要直接替代品时用 Basic commands。官方材料里**没有** `@Command` 注解，不要臆造。
## Brigadier 注册
```java
this.getLifecycleManager().registerEventHandler(LifecycleEvents.COMMANDS, event -> {
    event.registrar().register(node, "description", List.of("alias"));
});
```
| 位置 | 拿到的 manager |
|---|---|
| `PluginBootstrap#bootstrap(BootstrapContext)` | `LifecycleEventManager<BootstrapContext>`（`context.getLifecycleManager()`）；需要 `paper-plugin.yml` |
| `JavaPlugin#onEnable()` | `LifecycleEventManager<Plugin>`（`this.getLifecycleManager()`，`public final LifecycleEventManager<Plugin> getLifecycleManager()`） |

两者差异对注册命令没有影响。lambda 参数类型是 `ReloadableRegistrarEvent<Commands>`，只有 `cause()` 与 `registrar()` 两个方法。**早期注册的收益**：文档原文说这样注册的命令可以在服务器启动的更早阶段就被 datapack function 使用；此外经 `LifecycleEventManager` 注册的东西会在需要时被重新注册，不必自己处理 `/reload`。
## 命令树与参数
```java
static LiteralArgumentBuilder<CommandSourceStack> literal(String literal)
static <T> RequiredArgumentBuilder<CommandSourceStack, T> argument(String name, ArgumentType<T> argumentType)
```
- `then(...)` 挂子节点，`executes(...)` 挂执行体，`requires(...)` 挂使用条件，`.build()` 得到 `LiteralCommandNode<CommandSourceStack>`。
- `executes` 签名是 `T executes(Command<S> command)`；`Command<S>` 是 `@FunctionalInterface`，含常量 `int SINGLE_SUCCESS = 1`。
- `restricted(Predicate<CommandSourceStack>)` 是 `.requires(...)` 谓词的包装，用于阻止在聊天点击事件等无人值守上下文中执行。
- `Commands` 是 `@NonExtendable` 接口，**没有** `Commands.Builder` 类型；`getDispatcher()` 是 delicate API，注册应优先用 `register` 方法。

原生 Brigadier 参数：

| 类型 | 解析结果与限制 |
|---|---|
| `BoolArgumentType.bool()` | Boolean |
| `IntegerArgumentType.integer()` | Integer |
| `LongArgumentType.longArg()` | Long |
| `FloatArgumentType.floatArg()` | Float |
| `DoubleArgumentType.doubleArg()` | Double |
| `StringArgumentType.word()` / `.string()` / `.greedyString()` | String：word 只允许字母、数字与 `+ - _ .`；string 是单字或引号内可含空格；greedyString 原样输入且必须是最后一个参数 |

取值方法统一为 `(CommandContext<?> context, String name)`：对应类型上的 `getBool` / `getInteger` / `getLong` / `getFloat` / `getDouble` / `getString`。

Paper 的 `io.papermc.paper.command.brigadier.argument.ArgumentTypes` 提供带原生校验与客户端补全的类型（`player()` → `Player`、`entities()` → `List<Entity>`、`component()` → Adventure `Component`、`world()` → `World`、`resource(RegistryKey<T>)` 等）。选择器类参数要解析：`context.getArgument("target", PlayerSelectorArgumentResolver.class).resolve(context.getSource()).getFirst()`。**不要用 `net.minecraft.commands.arguments.*`**：internals 页原文 "PaperMC will offer no direct support for programming against Minecraft internals."

完整示例（`Commands` Javadoc 官方示例，写在 `onEnable()` 中）：
```java
LifecycleEventManager<Plugin> manager = this.getLifecycleManager();
manager.registerEventHandler(LifecycleEvents.COMMANDS, event -> {
    final Commands commands = event.registrar();
    commands.register(
        Commands.literal("new-command")
            .executes(ctx -> {
                ctx.getSource().getSender().sendPlainMessage("some message");
                return Command.SINGLE_SUCCESS;
            })
            .build(),
        "some bukkit help description string",
        List.of("an-alias")
    );
});
```
## 权限
`.requires(Predicate<S>)` 限制命令树的分支，`S` 是 `CommandSourceStack`。文档强调权限通常检查**命令发送者**，那才是发起执行的一方，即使 executor 可能是别人。
```java
Commands.literal("testcmd")
    .requires(source -> source.getSender().hasPermission("permission.test"))
    .executes(context -> {
        context.getSource().getSender().sendRichMessage("<gold>You have permission to run this command!");
        return Command.SINGLE_SUCCESS;
    })
```
- `CommandSender` 是 `Permissible`：`boolean hasPermission(String)` / `hasPermission(Permission)`。原文："If a permission override is not set on this object, the default value of the permission will be returned."；**客户端不同步警告**：客户端不知道 `requires` 条件，仍把命令显示为可执行。用 `Player#updateCommands()` 重新下发命令表；它线程安全，但费带宽；`BasicCommand` 走 `permission()`（默认 `canUse` 读它）；覆写 `canUse(CommandSender)` 后 `permission()` 就失效。

```yaml
permissions:
  permission.node:
    default: op
    children:
      permission.node.child: true
  another.permission.node:
    default: notop
default-permission: true
```
- `default` 可为 `op`/`notop` 或 `true`/`false`；**未指定时取 `default-permission` 的值，而 `default-permission` 默认是 `op`**；子节点设 `true` 表示继承父权限；命令级权限键是 `permission` 与 `permission-message`，原文："Players will only see commands they have permission to use."
## 描述符与命令的关系
| 描述符 | 有 `commands:` 吗 | 结论 |
|---|---|---|
| `paper-plugin.yml` | 无 | 原文："Paper plugins do not use the `commands` field to register commands." 经 `LifecycleEvents.COMMANDS` 注册的 Brigadier 命令与 `BasicCommand` **完全不需要**描述符条目 |
| `plugin.yml` | 有 | 仍记载 `commands:` / `permissions:`；legacy `CommandExecutor` 路径**必须**在此登记 |

```yaml
commands:
  command:
    usage: "/command <arg>"
    aliases: [cmd, command]
    permission: permission.node
```
未在描述符登记时（`JavaPlugin#getCommand(String)` Javadoc）：命令必须在 `PluginDescriptionFile` 里登记才能在运行时存在，找不到时返回 `null`；在 Paper 插件里于 `onEnable()` 调用它还会抛 `UnsupportedOperationException`。
## 其它要点
- **别名不会覆盖已有命令**。`register` 每条重载都重复：别名不覆盖已有命令（命名空间形式除外）；"Aliases are not Brigadier redirects, they just copy the command to a different label"；主标签/命名空间标签**会**覆盖已有命令；`register(...)` 全部返回 `@Unmodifiable Set<String>`，含义是 "successfully registered root command labels (including aliases and namespaced variants)"；`registerWithFlags(...)` 是 `@Internal`："not intended for public use"、"not guaranteed to be stable"；`CommandRegistrationFlag` 同样是内部用途。**不要使用。**；`BasicCommand` 是 `@FunctionalInterface`：必实现 `void execute(CommandSourceStack commandSourceStack, String[] args)`，另有 `suggest`、`canUse`、`permission()` 三个默认方法。`registerCommand` 的四个重载全部标注 "Only valid to be called inside `onEnable()`."
## 来源

- https://docs.papermc.io/paper/dev/command-api/basics/introduction/index.md
- https://docs.papermc.io/paper/dev/command-api/basics/registration/index.md
- https://docs.papermc.io/paper/dev/command-api/basics/requirements/index.md
- https://docs.papermc.io/paper/dev/command-api/basics/arguments-and-literals/index.md
- https://docs.papermc.io/paper/dev/command-api/arguments/minecraft/index.md
- https://docs.papermc.io/paper/dev/command-api/misc/basic-command/index.md
- https://docs.papermc.io/paper/dev/plugin-yml/index.md
- https://docs.papermc.io/paper/dev/getting-started/paper-plugins/index.md
- https://docs.papermc.io/paper/dev/internals/index.md
- https://jd.papermc.io/paper/26.3/io/papermc/paper/command/brigadier/Commands.html
- https://jd.papermc.io/paper/26.3/io/papermc/paper/command/brigadier/BasicCommand.html
- https://jd.papermc.io/paper/26.3/org/bukkit/plugin/java/JavaPlugin.html
