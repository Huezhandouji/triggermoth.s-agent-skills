# Paper 文本与 UI：Adventure、MiniMessage、物品与 GUI

**来源标记**：DOCS = docs.papermc.io 散文页；JAVADOC = jd.papermc.io（`paper-api 26.3`）。

## Adventure 与 Component

Component 是树状结构，样式与颜色从父节点继承；对象**不可变**，复杂文本用 builder，别每改一次就新建实例（DOCS）。`TextColor` = RGB，`NamedTextColor` = 预设色板，`TextDecoration` = `ITALIC` / `BOLD` / `STRIKETHROUGH` / `UNDERLINED` / `OBFUSCATED`。链式 API：`Component.text(String[, TextColor[, TextDecoration...]])`、`.color(...)`、`.decoration(TextDecoration, boolean)`、`.append(Component)`；builder 形式 `Component.text().content("A").color(...).append(...).build()`。

DOCS 完整示例：

```java
final TextComponent textComponent = Component.text("You're a ")
  .color(TextColor.color(0x443344))
  .append(Component.text("Bunny", NamedTextColor.LIGHT_PURPLE))
  .append(Component.text("! Press "))
  .append(Component.keybind("key.jump")
    .color(NamedTextColor.LIGHT_PURPLE)
    .decoration(TextDecoration.BOLD, true))
  .append(Component.text(" to jump!"));
```

## Adventure 5.x 的变化（容易踩坑）

| 5.x 之前 | 现在 |
|---|---|
| `BuildableComponent` | 已移除；用 `Component#toBuilder` 拿 `ComponentBuilder` |
| `ClickEvent` 是类 | 变成 **typed interface**，类型参数即 payload 类型；构造方式不变 |
| `ClickEvent.Action` 是 enum | **不再是 enum**，同样是 typed interface |
| `ClickEvent#value` | 移除 → `payload()` |
| `ClickEvent#create(Action, String)` | 移除 → 用接受 payload 的 `create`，或 `ClickEvent.openUrl(String)` 这类直接构造 |
| `PlainComponentSerializer`、`TranslationRegistry` | 移除 → `PlainTextComponentSerializer`、`TranslationStore` |

Adventure 5.x 最低 Java 21。`HoverEvent.Action` 是否仍是 enum：迁移页只讲了 `ClickEvent.Action`，**（未证实）**。

## MiniMessage

```java
final MiniMessage mm = MiniMessage.miniMessage();   // 默认 tags，非 strict
player.sendMessage(mm.deserialize("Hello <rainbow>world</rainbow>"));
```

自定义实例走 `MiniMessage.builder()...build()`，tag 由 `TagResolver` 提供（如 `StandardTags.color()`、`StandardTags.decorations()`）；依赖 `net.kyori:adventure-text-minimessage`，部分平台原生提供、无需自行添加。

**安全规则：绝不把玩家输入当 markup 解析。**

| 机制 | 文档化行为 |
|---|---|
| `MiniMessage.Preset.DEFAULT` | 全部特性，等同 `MiniMessage.miniMessage()` |
| `MiniMessage.Preset.NON_INTERACTABLE` | 禁用可交互特性（click、hover、insertion），用 post-processor 剥掉结果中的可交互元素；自定义 resolver 产出的组件同样处理 |
| `MiniMessage.Preset.FORMATTED_TEXT` | 只允许文本组件与格式（color、shadow、font、decoration），post-processor 移除非文本组件与可交互元素 |
| `Placeholder.unparsed(name, 用户输入)` | **推荐**：原样插入、不解析——"you can sanitize user input without escaping the tags directly"。`unparsed("name", "<red>TEST :)")` 塞进 `<gray>Hello <name>` → 灰色字面量 `Hello <red>TEST :)` |
| `Placeholder.parsed(name, 输入)` | 解析前就替换进字符串，**其中的 tag 会影响占位符之后的解析结果** → 玩家输入绝不能用它 |
| `strict(true)` | 默认永不因用户输入抛异常（非法 tag 当普通文本）；strict 对未闭合 tag 抛异常，但**仍**放过格式错误的 tag；禁止 `<reset>`，tag 须逆序闭合 |

`MiniMessage.miniMessage(MiniMessage.Preset.NON_INTERACTABLE)` 直接拿受限实例，再改配置用 `MiniMessage.builder(preset)`。**实例只建一次**并在中心位置复用。绝不要混用 MiniMessage 与 legacy `§`/`&`：FAQ 明确 "There are no working, recommended, or supported ways of using both MiniMessage and legacy color codes and there never will be."

## 发送文本与弃用对照表

`CommandSender` 继承 `Audience`：`sendActionBar`、`showTitle`、`showBossBar` 均可用。

| 不要用 | 改用 |
|---|---|
| `ChatColor` 与一切 `§` legacy 格式 | `NamedTextColor`、`TextColor.color(...)`、`TextDecoration`；Paper 中 legacy 相关方法与类型普遍带弃用标记 |
| `CommandSender#sendMessage(String)` | **它是 `@ApiStatus.Obsolete`，不是 Java `@Deprecated`** → `sendMessage(Component)` / `sendPlainMessage(String)` / `sendRichMessage(String)` |
| `sendMessage(UUID sender, String)`、`sendMessage(BaseComponent...)` | `@Deprecated`（前者 "sender UUID is ignored"）→ `sendMessage(Component)` |
| `ItemMeta#getDisplayNameComponent()` / `getLoreComponents()` | `@Deprecated` → `displayName()` / `lore()` |
| `LegacyComponentSerializer` 用于日常显示 | 该页标题即"（Deprecated）"、Paper 页也 caution 不推荐；**但仍是官方认可的一次性迁移桥梁**：`MiniMessage.miniMessage().serialize(LegacyComponentSerializer.legacySection().deserialize(legacyString))` |

## 物品

现代写法（JAVADOC）：`ItemStack.of(Material)` / `of(Material, int)` / `empty()`；数量 `getAmount()` / `setAmount(int)`；名字 `ItemMeta#displayName(Component)`；lore `ItemMeta#lore(List<? extends Component>)`（`ItemStack#lore(...)` 也可）；改 meta 用 `ItemStack#editMeta(Consumer<? super ItemMeta>)`。

已弃用：

| 弃用项 | 替代 |
|---|---|
| `ItemMeta#setCustomModelData(Integer)`、`getCustomModelData()`、`hasCustomModelData()` | `setCustomModelDataComponent(CustomModelDataComponent)` 等 |
| `getLore()` / `setLore(List<String>)` | `lore()` |
| `getDisplayName()` / `setDisplayName(String)` | `displayName(Component)` |
| `getLoreComponents()` / `getDisplayNameComponent()` | `lore()` / `displayName()` |
| `ItemFlag.HIDE_ADDITIONAL_TOOLTIP` | "does not exist anymore" → `TooltipDisplay` + `ItemStack.setData(...)` |
| `ItemStack#getI18NDisplayName()`、`getTranslationKey()` | `translationKey()`；`ItemStack` 自身 implements `Translatable` |

`ItemStack#translatable` **不存在**。lore 有父样式会强制斜体，必要时用 `Component.decorationIfAbsent()`。`Component.translatable(...)` 的服务端翻译对 `ItemStack` 的名字 / lore 不生效。

## GUI 正确性

官方散文文档**没有**通用 GUI 教程：`/paper/dev/api/inventories/index.md` 只有 `<PageCards />` 占位（`/paper/dev/inventory/` 404），可用材料只有 Custom InventoryHolders 与 Menu Type API 两页。**别用标题识别 GUI**（别的插件、玩家都可能造出同名 inventory）：用自定义 `InventoryHolder` 配 `getHolder(false) instanceof MyInventory`。

以下规则全来自 JAVADOC：

| 规则 | 内容 |
|---|---|
| 拖拽 | 拖拽**永远不触发** `InventoryClickEvent`，只在结束时触发 `InventoryDragEvent` → 只守 click 的 GUI 并不安全 |
| 视图操作 | handler 里别调 `closeInventory()` / `openInventory(Inventory)` / `InventoryView.close()`；用 `BukkitScheduler.runTask` 延到下一 tick，"this is not an exhaustive list" |
| 覆盖 | handler 改的槽位可能被事件结果覆盖 → **cancel 事件后自行应用全部改动**，或延到下一 tick |
| 刷新 | 改 contents 或 MaxStackSize 需要 `Player.updateInventory()` |
| 槽位 | `getRawSlot()` 对视图唯一（`InventoryView#getItem(int)` 要的就是它）；`getSlot()` 可能重复——"a view links two different inventories"；`InventoryView#convertSlot(int)` 做 raw → local；`InventoryView.OUTSIDE`（`-999`）时 `getInventory(int)` 返回 null |
| 点击位置 | `getClickedInventory()` 可能为 null（"null if clicked outside"）；docs 示例正用它避免误判玩家背包里的同类物品 |
| 标题 | `getTitle()` 弃用 → `title()`；`setTitle(String)` `@Deprecated(since="1.21.1")`："poorly defined and broken behaviors. It should not be used." |

Menu Type API 是视图体系的入口：`MenuType.GENERIC_9X6.builder().title(Component)...build(player).open()`，builder 可复用与克隆。`Bukkit.createInventory(holder, size, Component)` 在已抓取的散文页里**没有**出现（**未证实**）；`InventoryCloseEvent` 在已抓取的任何页面里都没出现（未证实）。

## 来源

**DOCS**
- https://docs.papermc.io/adventure/text/index.md
- https://docs.papermc.io/paper/dev/component-api/introduction/index.md
- https://docs.papermc.io/adventure/migration/adventure-4.x/index.md
- https://docs.papermc.io/adventure/minimessage/api/index.md
- https://docs.papermc.io/adventure/minimessage/dynamic-replacements/index.md
- https://docs.papermc.io/adventure/faq/index.md
- https://docs.papermc.io/paper/dev/custom-inventory-holder/index.md
- https://docs.papermc.io/paper/dev/menu-type-api/index.md

**JAVADOC（paper-api 26.3）**
- https://jd.papermc.io/paper/26.3/org/bukkit/command/CommandSender.html
- https://jd.papermc.io/paper/26.3/org/bukkit/inventory/ItemStack.html
- https://jd.papermc.io/paper/26.3/org/bukkit/inventory/meta/ItemMeta.html
- https://jd.papermc.io/paper/26.3/org/bukkit/inventory/ItemFlag.html
- https://jd.papermc.io/paper/26.3/org/bukkit/event/inventory/InventoryClickEvent.html
- https://jd.papermc.io/paper/26.3/org/bukkit/inventory/InventoryView.html
