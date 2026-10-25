# 构建、实测与排查

## 构建

用工程自带的 wrapper，不要用系统里的 gradle：

```powershell
.\gradlew.bat build        # Windows
./gradlew build            # Linux / macOS
```

产物在 `build/libs/`。很多工程会用 `tasks.register<Copy>` + `finalizedBy` 在 `build` 后把 jar 拷进测试服的 `plugins/`，改完代码只需 build 一次。

**没有跑通 `build` 的改动不算完成。** 编译错误必须解决，不能留给用户。

## 本地测试服务器

官方 PaperMC 文档**没有**提供 Gradle 启动任务，只链接到第三方插件 run-paper。若工程已经用了它，任务形态是：

```kotlin
plugins { id("xyz.jpenilla.run-paper") version "<run-paper 版本>" }
tasks { runServer { minecraftVersion("<MC 版本>") } }
```

run-paper 会自动把本插件的 jar（有 shadowJar 就用 shadowJar）塞进测试服。**版本号以工程里已有的为准**；要装新版本时查该项目的发布页，不要背版本号。

长跑的服务器必须放进后台作业，否则会占住会话：

- 启动：后台作业跑 `.\gradlew.bat runServer`
- 读控制台：`job_output`
- 停止：`job_kill`

第一次启动要下载服务端与依赖，会久；不要在超时后反复重启，看日志判断它在下载还是在报错。

## 日志分诊

按这个顺序看 `logs/latest.log`：

| 现象 | 通常原因 | 处理 |
|---|---|---|
| 搜不到 `Enabling <PluginName>` | 插件没被加载 | 看前面的 `Could not load plugin`；确认 jar 在 `plugins/` 且描述符 `name` 匹配 |
| `UnsupportedClassVersionError` | Java 版本低于 Paper 要求 | 对齐 toolchain 与服务器 JVM |
| 缺 `api-version` 的 legacy 警告 | 描述符里没写或写错 | 补 `api-version` |
| `NoClassDefFoundError` | 依赖没进 jar，或用了目标版本没有的类 | 检查依赖范围；核对 API 是否存在 |
| `NoSuchMethodError` | 编译时和运行时的 API 版本不一致 | 核对 `paper-api` 版本与服务器版本 |
| 插件互相依赖成环 | Paper 插件循环加载会直接关服 | 解开依赖环；应急可用 `-Dpaper.useLegacyPluginLoading=true` |
| `UnsupportedOperationException` 来自 `getCommand` | 在 Paper 插件里于 `onEnable()` 调了它 | 改用 Brigadier / `BasicCommand` 注册 |
| enable 成功但命令没反应 | 注册代码没执行，或权限 `requires` 拦掉了 | 加日志确认注册路径；用有权限的账号试 |
| 异步任务里崩 | 在异步线程碰了游戏状态 | 回主线程再操作（见 `04-events-and-scheduling.md`） |

## 用字节码核实签名（比查网页更可靠）

它查的就是你编译时用的那个 jar，所以结论与编译结果一致：

```powershell
$jar = Get-ChildItem "$env:USERPROFILE\.gradle\caches\modules-2\files-2.1\io.papermc.paper" -Recurse -Filter "paper-api-*.jar" |
       Where-Object { $_.Name -notmatch 'sources|javadoc' } | Select-Object -First 1
javap -classpath $jar.FullName io.papermc.paper.plugin.lifecycle.event.types.LifecycleEvents
javap -classpath $jar.FullName org.bukkit.event.block.BlockBreakEvent
```

`javap` 只给签名，不给语义；语义查官方文档。

## 查官方文档的正确姿势

`docs.papermc.io` 是 Astro/Starlight 站点，**渲染后的 HTML 抓不到正文**。每个页面同时提供原始 Markdown：在页面 URL 后加 `/index.md`。

```powershell
Invoke-RestMethod https://docs.papermc.io/paper/dev/plugin-yml/index.md
Invoke-RestMethod https://docs.papermc.io/llms.txt      # 全站页面索引
```

注意 Markdown 源里的 `\{CONSTANT}` 占位符是构建期替换的，抓到的原文里不会解析成实际版本号——**不要把它们当成真实版本号**。版本号去 `https://fill.papermc.io/v3/projects/paper` 查。

`jd.papermc.io/paper/` 会重定向到当前版本目录，Javadoc 用于确认签名与弃用状态。

## 汇报

改完要说清：改了哪些文件、验证到了哪一步（编译过 / 起服过 / 看到哪行日志）、还有什么没验证。没实测的部分就直说没实测。
