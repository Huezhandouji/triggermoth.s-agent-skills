# Paper 配置与数据持久化

**来源标记**：DOCS = docs.papermc.io 散文页；JAVADOC = jd.papermc.io（`paper-api 26.3`）。**（仅 JAVADOC）** = 只在 Javadoc 出现。

## config.yml 工作流

默认格式是 YAML（`.yml` / `.yaml`）；JSON、TOML 不是 Paper 原生支持。层级用 `.` 分隔：`root.another-key`。默认资源放在 `src/main/resources/config.yml`，随 jar 发布；写盘时按 jar 内层级原样落到数据文件夹。

```java
public void onEnable() {
    this.saveDefaultConfig();             // = saveResource("config.yml", false)
    String host = this.getConfig().getString("database.host");
    this.getConfig().set("database.host", "localhost");
    this.saveConfig();                    // 散文页写法是 getConfig().save(file)（抛 IOException）
}
```

| 方法 | 行为 |
|---|---|
| `saveDefaultConfig()` | "Saves the raw contents of the default config.yml … should fail silently if the config.yml already exists"（JAVADOC） |
| `saveResource(String, boolean replace)` | 从 jar 内嵌资源写出并保留子目录层级；`replace=true` 每次调用都覆盖（DOCS + JAVADOC） |
| `getConfig()` | "Gets a `FileConfiguration` for this plugin, read through \"config.yml\""（JAVADOC）；要先保存过才拿得到 |
| `reloadConfig()` | "Discards any data in `Plugin.getConfig()` and reloads from disk." **（仅 JAVADOC）** |
| `saveConfig()` | "Saves the `FileConfiguration` retrievable by `Plugin.getConfig()`." **（仅 JAVADOC）** |
| `FileConfiguration#save(File)` | 不存在则创建，存在则**覆盖**；DOCS 强调设置数据后必须 save 才持久化 |
| `options().copyDefaults(boolean)` | **（仅 JAVADOC）**，默认 **false**；为 true 时默认值被直接拷入，`contains()` 与 `isSet()` 永远同结果（分不清显式设置与默认值）；setter 可链式 |

读取器：`getString`、`getInt`、`getBoolean`、`getList`、`getStringList`、`getMapList`、`getItemStack`、`getLocation`、`getVector`、`getOfflinePlayer`、`getConfigurationSection`、`getKeys(boolean)`、`contains(String[, boolean])`、`isSet`；写侧 `set`、`createSection`；`getSerializable(String, Class[, T])` 读回自定义类型。`ItemStack`、`Location`、`Vector` 能直接存取，因为它们实现 `ConfigurationSerializable`。`getKeys(boolean)` 的布尔语义（深 / 浅遍历）在任何已抓取页面里都没说明（**未证实**）。

## 结构与序列化

一组字段被多处读写时，收成一个类型比到处写 `getConfig().getString("a.b.c")` 好：裸字符串 key 容易拼错。DOCS 的写法：

```java
public class TeleportOptions implements ConfigurationSerializable {
    public Map<String, Object> serialize() { /* 实例方法，返回 Map */ }
    public static TeleportOptions deserialize(Map<String, Object> args) { /* 返回实例 */ }
}
ConfigurationSerialization.registerClass(TeleportOptions.class); // 必须注册
```

DOCS caution 原文：不调用 `ConfigurationSerialization#registerClass(java.lang.Class)`，Paper 插件就既存不了也读不了自定义类。读回用 `getSerializable(String, Class[, T])`（**（仅 JAVADOC）**）。`ItemStack`、`ItemMeta`、`Location` 自身就是 `ConfigurationSerializable`（JAVADOC），所以 `getItemStack`、`getLocation` 可直接用。

## 数据持久化选型

| 方案 | 适合什么 |
|---|---|
| PDC（`PersistentDataContainer`） | 把数据挂在游戏对象上（`ItemStack`、`Entity`、`World`、`Chunk`、`TileState`、`OfflinePlayer`…）。文档称它比 NBT 反射 / 塞进 lore 更可靠高效，不依赖服务器内部实现，生命周期随对象 |
| `ConfigurationSerializable` + YAML | 自定义值类型写进 config.yml；必须先 `registerClass` |
| `getDataFolder()` 下的自有 `YamlConfiguration` | 不适合塞进 config.yml 的数据：`YamlConfiguration.loadConfiguration(file)` / `config.save(file)`；文件不存在或读取出错时返回空配置 |
| 数据库 | DOCS："Databases are the recommended way to store a large amount of data." SQLite 驱动 Paper 内置、不用 shade；MySQL / MariaDB / PostgreSQL 需自行 shade HikariCP；"**Always** use prepared statements" |

旧做法（文档明确不推荐）：NBT 反射 "generally unreliable in the long term"、把数据塞进 lore 或 display name（"Prone to collisions as well as slow to access"）。

## PDC

```java
NamespacedKey key = new NamespacedKey(plugin, "example_key"); // key 对象应复用
PersistentDataContainer pdc = world.getPersistentDataContainer();
pdc.set(key, PersistentDataType.STRING, "I love tacos!");
String v = pdc.getOrDefault(key, PersistentDataType.STRING, "<null>"); // get 可空
```

- **NamespacedKey**：`new NamespacedKey(plugin, "id")` 自动用小写插件名当 namespace，优先用它；也可 `(namespace, key)`。namespace 只允许小写字母数字与 `.` `_` `-`，key 还允许 `/`。`NamespacedKey#randomKey()` 已 `@Deprecated(since="1.20.5")`。
- **支持 PDC 的类**（文档原文列举）：`ItemStack`、`Chunk`、`World`、`Entity`、`TileState`、`Structure`、`GeneratedStructure`、`Raid`、`OfflinePlayer`、`ItemMeta`——它们实现 `PersistentDataHolder`，用 `getPersistentDataContainer()` 取。
- **操作**：`get`（可空）、`getOrDefault`、`set`、`has`。用 `getOrDefault` 是因为 "Some API parts, such as Adventure's `Component#text(java.lang.String)`, require non-null values"。`remove` 未出现在任何已抓取页面（**未证实**）。
- **内置类型**：`Byte`、`Byte Array`、`Double`、`Float`、`Integer`、`Integer Array`、`Long`、`Long Array`、`Short`、`String`、`Boolean`、Tag Containers（嵌套 PDC）、`Lists`。`Boolean` 只是便利类型。
- **列表与嵌套**：

```java
container.set(key, PersistentDataType.LIST.strings(), List.of("a", "list"));
List<String> s = container.get(key, PersistentDataType.LIST.strings());
PersistentDataContainer nested = container.getAdapterContext().newPersistentDataContainer();
```

- **自定义数据类型**：实现 `PersistentDataType<byte[], UUID>`，提供 `getPrimitiveType()`、`getComplexType()`、`toPrimitive(UUID, PersistentDataAdapterContext)`、`fromPrimitive(byte[], PersistentDataAdapterContext)`；做成单例后把实例传给 `get`/`set`/`has`：

```java
public class UUIDDataType implements PersistentDataType<byte[], UUID> {
    public static final UUIDDataType INSTANCE = new UUIDDataType();
    public byte[] toPrimitive(UUID complex, PersistentDataAdapterContext ctx) { /* ByteBuffer */ }
    public UUID fromPrimitive(byte[] primitive, PersistentDataAdapterContext ctx) { /* ... */ }
}
// container.set(key, UUIDDataType.INSTANCE, uuid);
```

- **只读视图**：`PersistentDataContainerView`。`ItemStack#getPersistentDataContainer()` 返回只读视图（**1.21.1+**；更早版本走 `ItemMeta`，1.16.5+ 有 `editMeta`）；写入用 `ItemStack#editPersistentDataContainer(Consumer)`（**1.21.4+**），"The persistent data container available in the consumer is not valid outside the consumer"。`OfflinePlayer` 完全没有写入途径——它直接从磁盘读，需要阻塞文件操作。`TileState` 要先 cast 再 `update()`：

```java
if (block.getState() instanceof Chest chest) {
    chest.getPersistentDataContainer().set(key, PersistentDataType.STRING, "x");
    chest.update();
}
```

- ⚠ **数据不会自动搬运**（DOCS caution 原文）："Data is **not** copied across holders for you, and needs to be **manually** copied if 'moving' between PersistentDataHolders. E.g. Placing an ItemStack as a Block (with a TileState) ***does not*** copy over PDC data."

## 线程与 IO

- DOCS danger 原文："Loading and saving files on the main thread will slow down your server. `load` and `save` operations should be executed asynchronously."
- 做法：文件与数据库 IO 一律放到异步线程，结果回到主线程再应用（世界、方块、实体、背包等状态只能在主线程改）。线程归属与「异步上下文不得触碰 Bukkit 状态」的规则见 `04-events-and-scheduling.md`，此处不重复。
- 异步上下文里碰 Bukkit API 本身就不安全："using the Bukkit API in an asynchronous context (i.e. the event handler) is unsafe and exceptions may be thrown"（DOCS）。
- `getDataFolder()` 在 `saveDefaultConfig()` 之前是否已存在：**任何已抓取页面都没说**（未证实），别依赖这个顺序。文档只保证它返回 "the folder that the plugin data files are located in"。

## 来源

**DOCS**
- https://docs.papermc.io/paper/dev/plugin-configurations/index.md
- https://docs.papermc.io/paper/dev/pdc/index.md
- https://docs.papermc.io/paper/dev/using-databases/index.md
- https://docs.papermc.io/paper/dev/chat-events/index.md

**JAVADOC（paper-api 26.3）**
- https://jd.papermc.io/paper/26.3/org/bukkit/plugin/java/JavaPlugin.html
- https://jd.papermc.io/paper/26.3/org/bukkit/configuration/file/FileConfiguration.html
- https://jd.papermc.io/paper/26.3/org/bukkit/configuration/ConfigurationOptions.html
- https://jd.papermc.io/paper/26.3/org/bukkit/NamespacedKey.html
- https://jd.papermc.io/paper/26.3/org/bukkit/inventory/ItemStack.html
- https://jd.papermc.io/paper/26.3/org/bukkit/inventory/meta/ItemMeta.html
