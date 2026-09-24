# AGENTS.md — 本仓库 AI 工作指南与经验总结

> 本文件由原 `JAR-EMBEDDED-SPIDER.md` 整理、扩充而来，汇总 TV-fongmi 仓库已完成的主要改动、协议与踩坑经验，供以后 AI / 开发者参考。
> **改动源码后务必执行构建命令验证。**

---

## 0. 快速开始 / 构建

- 平台 Android（Gradle），工作目录 `D:\TV-fongmi`。
- 编译 leanback 变体（最常用）：

  ```
  .\gradlew.bat :app:compileLeanbackDebugJavaWithJavac --console=plain
  ```

- 同时编译 mobile 变体：

  ```
  .\gradlew.bat :app:compileMobileDebugJavaWithJavac --console=plain
  ```

- Java 21 工具链由 Gradle 自动下载（首次较慢）。
- 模块：`app`（主应用，含 `src/main`、`src/mobile`、`src/leanback` 三套源集）、`catvod`（工具与爬虫基类）、`quickjs`（QuickJS 引擎与 `Module.fetch`）、`chaquo`（Python/Chaquopy，`chaquo/src/main/python/app.py`）、`hook`、`thunder`。
- **注意**：`app` 有 `mobile` 与 `leanback` 两套 `HomeActivity` / `SettingFragment` 等，改 UI/启动逻辑时两套都要改。

---

## 1. AI 约定

- 除非明确要求，**不添加注释**。
- 只做需求范围内的改动；改完执行上面的编译命令。
- 仅在明确要求时提交 Git。
- 优先复用现有工具类：`Path`、`UrlUtil`、`Crypto`、`OkHttp`、`Prefers`、`PermissionUtil` 等。
- Groovy 构建脚本为 `app/build.gradle`（无 `.kts`）。

---

## 2. 存储与权限模型

- 内部存储：`Path.cache()` = `Init.context().getCacheDir()`；`Path.files()` = `getFilesDir()`（`catvod/src/main/java/com/github/catvod/utils/Path.java`）。
- 外部存储：`Path.root()` = `Environment.getExternalStorageDirectory()`（`/sdcard`）；`Path.tv()` = `/sdcard/TV`（备份/字体/mpv 等已在用）。
- 权限声明（`app/src/main/AndroidManifest.xml`）：`MANAGE_EXTERNAL_STORAGE`（Android 11+）、`WRITE_EXTERNAL_STORAGE(maxSdkVersion=29)`、`requestLegacyExternalStorage=true`。
- 授权引导（`app/src/main/java/com/fongmi/android/tv/utils/PermissionUtil.java`）：
  - `requestFile(activity, callback)`：用户触发文件操作时请求（Android 11+ 跳系统「所有文件访问」设置页，**无法静默授予**；10 及以下走运行时权限）。
  - `requestFileAuto(activity, callback)`：**启动时一次性引导**；用 `Prefers` 键 `permission_file_asked` 记录，避免反复弹设置页。
  - `canUseExternalStorage()`：Android R+ 看 `Environment.isExternalStorageManager()`，R 以下看 `WRITE_EXTERNAL_STORAGE` 是否授予。
- 无外部权限时功能需**回退内部存储**，不能阻塞加载。

---

## 3. jar 内嵌爬虫（核心特性）

### 3.1 协议约定

- 内嵌资源只放在 jar 的 `assets/` 目录下，**无二级目录**，例如：
  - `assets/spider.py`
  - `assets/index.js`
  - `assets/demo.json`
  - `assets/version`（版本文件，见 3.3）
- 站点引用统一写作 `jar://assets/<文件名>`，路径必须与 jar 内条目完全一致（含 `assets/` 前缀）。
- jar 源由配置顶层 `spider` 或站点自身 `jar` 提供（至少一个）。
- **只有满足全部条件才解压**：接口原始 scheme 为 `http(s)://` + 配置了 `;md5;<值>` + jar 内提供 `assets/version`：
  - `"spider": "https://xxx/custom_spider.jar;md5;63ff..."` → 解压。
  - `"spider": "https://xxx/custom_spider.jar"`（无 md5）→ 只加载 dex，不解压。
  - `"spider": "file://.../x.jar;md5;<值>"` → 只加载 dex，不解压。
  - `"spider": "assets://x.jar;md5;<值>"` → 只加载 dex，不解压（`assets://` 虽经本地 http 中转，但原始 scheme 不是 http）。

### 3.2 jar 本体缓存与更新判断

- **缓存命名**：jar 本体存为 `md5(jar URL 字符串) + ".jar"`（`Path.jar(url, external)`）。这个哈希是 **URL 字符串**的 `Crypto.md5(String)`，**不是**配置 `;md5;` 值，也不是 jar 内容 md5。每个 URL 固定一个缓存槽，升级覆盖、不按版本堆积。
- **位置**：外部优先 `/sdcard/TV/jar/<md5(URL)>.jar`；无外部权限回退 `cache/jar/<md5(URL)>.jar`。
- **更新判断（内容校验，不靠文件名）**：`Crypto.equals(cachedFile, 配置md5)`（内部对缓存文件实际算 md5）与解析后的配置 `;md5;` 比对：
  - 相等 → 缓存命中，直接用，不联网；
  - 不相等 / 文件不存在 → 从 URL 重新下载覆盖。
- **版本信号**：配置 `;md5;` 字面值变化，或 `;md5;` 写成 http 地址时每次加载先 `OkHttp.string(md5)` 取回期望值。
- **注意**：配置 `;md5;` 不变、远端 jar 被替换时**不会**判定为更新（本地缓存有效就永远不联网）。这是「md5 即版本契约」的设计。

### 3.3 解压与落盘

- **目录（按 jar 隔离）**：外部 `/sdcard/TV/jar/<md5(URL)>/assets/...`；无外部权限回退 `cache/jar/<md5(URL)>/assets/...`。保留 jar 内 `assets/` 前缀。
- **版本文件驱动更新**：jar 内提供 `assets/version`（作者维护，内容为版本号字符串）。
  - 读 jar 内 `assets/version` 与缓存侧 `.../assets/version` 对比：
    - 不相等（或缓存缺失）→ `Path.clear` 该 jar 目录 → 解压覆盖 → **最后**写入版本文件；
    - 相等 → 跳过；
    - jar 内无 `assets/version`（或内容为空）→ **不解压、不建目录**。
- 解压时**跳过** zip 内的 `assets/version` 条目，拷贝循环结束后再写该文件，保证中途失败可重试。
- **DexClassLoader 优化目录**：用内部 `Path.dex()`（`cache/dex`），不放外部（外部存储 noexec/FUSE 不可靠）。
- **流程顺序（重要）**：解压必须在「jar 本体缓存与更新判断」**之后**。`parseJar` 先按 `;md5;` 确定 jar 本体（缓存命中 / 重新下载覆盖），再在 `load` 内 `extract(file)`；`extract` 读取的 `assets/version` 来自这个最终 jar。`file()` 同样是「先 `parseJar`（含解压）再拼路径」。

### 3.4 配置示例

```json
{
  "spider": "http://host/xxx.jar;md5;<hash>",
  "sites": [
    {
      "key": "Ikanbot",
      "name": "爱看机器人",
      "type": 3,
      "api": "jar://assets/Ikanbot.js",
      "changeable": 1,
      "searchable": 1,
      "ext": ""
    },
    {
      "key": "豆瓣",
      "name": "豆瓣",
      "type": 3,
      "api": "csp_Douban",
      "changeable": 1,
      "searchable": 0,
      "ext": "jar://assets/demo.json"
    }
  ]
}
```

直播源 `url` 也支持 `jar://`（LiveConfig 的 `lives[].url`）：

```json
{
  "lives": [
    { "name": "我的直播", "url": "jar://assets/live.txt" }
  ]
}
```

### 3.5 路由与集成

- `BaseLoader.getSpider(...)`：`api` 以 `jar://` 开头先 `resolveJar`（仅 `.py`/`.js`，解析为 `file://<绝对路径>`，失败返回 `SpiderNull`）；`ext` 以 `jar://` 开头替换为其内容。
  - `csp_*` → JarLoader（反射 `com.github.catvod.spider.XXX`，`api.split("csp_")[1]`）
  - `.js` → JsLoader（QuickJS，加载 jar 内 `com.github.catvod.js.Function`）
  - `.py` → PyLoader（Chaquopy）
- `Site.fetchExt()`（`app/.../bean/Site.java`）：`ext` 为 `jar://` 时用 `BaseLoader.get().ext(ext, jar)` 读取 jar 资源内容替换。
- `LiveParser.getText(Live)`（`app/.../api/parser/LiveParser.java`）：`Live.url` 为 `jar://` 时用 `BaseLoader.get().ext(url, jar)` 读取直播源文本（m3u/txt/json）；读不到抛异常。
- `quickjs/.../utils/Module.java` `fetch`：`file://` 本地读取（主脚本与嵌套 import）。
- `chaquo/src/main/python/app.py`：`spider`/`download` 支持 `file://`；模块名 `<父目录名>_<文件名>` 避免不同 jar 同名脚本 `sys.modules` 冲突。
- `ImgUtil.getUrl`（`app/.../utils/ImgUtil.java`）：URL 后缀支持 `@Headers=<json>@`、`@Referer=`、`@User-Agent=`。
- `Ikanbot.js`：豆瓣图片 CDN 需要 `Referer: https://movie.douban.com/`（UA 单独无效）。

### 3.6 运行流程

```
config.spider (jar URL) ──parseJar──► 先按 ;md5; 决定 jar 本体（缓存命中 / 重新下载覆盖 <jarDir>/<md5(URL)>.jar）
                                              │  <jarDir> = /sdcard/TV/jar（有外部权限）或 cache/jar（回退）
                                              │  再随后解压（仅当原始 http(s) + 配置 ;md5; + jar 内有 assets/version）：
                                              │  assets/** → <jarDir>/<md5(URL)>/assets/**，版本记录 assets/version
                                              ▼
site.api = "jar://assets/index.js" ──resolveJar──► file:///.../<jarDir>/<md5(URL)>/assets/index.js ──► QuickJS
site.api = "jar://assets/spider.py" ─resolveJar──► file:///.../<jarDir>/<md5(URL)>/assets/spider.py ──► Chaquopy
site.ext = "jar://assets/demo.json" ──ext────────► 读取 <jarDir>/<md5(URL)>/assets/demo.json 内容作为 extend
LiveConfig lives[].url = "jar://assets/live.txt" ──LiveParser.getText──► 读取文本解析频道
site.api = "csp_XXX" ────────────────────────────► 反射实例化 dex 内 com.github.catvod.spider.XXX（不变）
```

### 3.7 安全与限制

- 仅在「原始 scheme 为 `http(s)://` + 配置 `;md5;<值>` + jar 内提供 `assets/version`」时解压；否则不产生解压文件。
- 只解压 `assets/` 前缀条目，保留前缀落到按 jar 隔离的目录；含 canonical 路径前缀校验，拒绝 `..` 等越界路径。
- **版本文件门槛**：无 `assets/version`（或空）不执行任何解压动作（不建目录、不拷贝、不写记录）。
- 限制单 jar 解压条目数（`MAX_ENTRIES = 2000`）与总字节数（`MAX_BYTES = 64MB`），防解压炸弹。
- 版本文件在解压成功且最后写入；加载时对比，不相等才 `Path.clear` 目录后重解压。
- 按 jar 隔离：`Path.clear` 仅清本 jar 目录，不影响其它 jar。
- `jar://` 必须带 `.py` / `.js` 后缀才能被识别并路由；`ext` 必须带 `assets/` 前缀。
- 资源型 jar（无 `classes.dex`）可用：`invokeInit` / `invokeProxy` / JS `createFun` 均已容错。
- `JarLoader.clear()` 只清内存实例，不清解压文件。

---

## 4. Ikanbot.js

位置：`D:\Spider\Ikanbot.js`（与参考 `D:\Spider\spider.js` 同目录，未纳入本仓库版本控制）。

- 参考 `D:\Spider\app\src\main\java\com\github\catvod\spider\Ikanbot.java` 与最新站点
  `https://www1.ikanbot.com` 重写为 QuickJS 爬虫。
- 遵循 QuickJS 契约：`export default createSpider`，数据方法返回 JSON 字符串。
- HTML 解析使用内建模块 `assets://js/lib/cat.js` 导出的 `cheerio`；HTTP 使用全局 `req`。
- 站点结构：榜单 `div.item-root`、分类 `a.item`、搜索 `a.cover-link`、详情
  `h1#video_title` / `og:image` / `div.detail > h3` / `#current_id` / `#e_token` / `#mtype`。
- 播放线路：`/api/getResN?videoId=&mtype=&token=`，`token` 由 `get_tks` 从 `current_id` 末四码与 `e_token` 推导；
  `resData` 解析为 `[{url,newName}]`，`url` 按 `#` 拆集、`$` 拆“名称$链接”，多线路用 `$$$` 分隔。
- 图片：`vod_pic` 用 `picUrl()` 处理，豆瓣图片加 `Referer: https://movie.douban.com/`。
- 放进 jar 时置于 `assets/Ikanbot.js`，站点 `api` 写 `jar://assets/Ikanbot.js`：

```json
{
  "key": "Ikanbot",
  "name": "爱看机器人",
  "type": 3,
  "api": "jar://assets/Ikanbot.js",
  "searchable": 1,
  "quickSearch": 1,
  "changeable": 0,
  "ext": ""
}
```

---

## 5. 经验与坑（lessons）

- **单共享标记的教训**：曾用「所有 jar 共用 `cache/jar/assets/` + 一个 `.md5` 标记」，多个带 `;md5;` 的 jar 会互相覆盖标记 → 反复重解压。现改为**按 jar 隔离目录 + `assets/version` 版本文件**。
- **`File.setReadOnly()` 在外部存储可能返回 false**（FUSE/sdcardfs），不能作为加载门槛；`JarLoader.load` 已放宽为只校验文件存在，`setReadOnly()` 仅尽力而为。
- **`Path.copy` / `Path.write` 静默吞 IOException**（返回 void/原 file），可能表面成功但文件缺失；排查问题时留意。
- **版本比较是字符串不等**：作者每次更新 assets 必须改 `assets/version` 内容，否则不会重解压；仅改代码不改版本号无效。
- **jar 本体命名用 URL 的 md5**（每 URL 一个缓存槽，升级覆盖不堆积）；配置 `;md5;` 只用于内容校验与下载判断。
- **`assets://`** 是 APK 自身 assets 的虚拟地址（经本地 http 中转），原始 scheme 非 http，因此不解压。
- **`assets/version` 既是被解压的资源也是版本标记**；解压时跳过它、最后写入，避免半解压被误判完成。
- 外部写入需权限：Android 11+ 只能引导用户到系统设置开启「所有文件访问」，无法静默获取；启动用 `requestFileAuto` 引导一次，拒绝则回退内部。
- Gradle 构建脚本是 `app/build.gradle`（Groovy），不是 `.kts`；`kilo.json`/`.kilo` 为本项目 AI 配置目录（已 gitignore）。

---

## 6. 关键文件清单

- `app/.../api/loader/JarLoader.java`：jar 本体缓存/下载判断、`extract`（版本文件驱动、按 jar 隔离、外部优先回退内部）、`file()`/`ext()` 的 `jar://` → 路径映射、`Path.dex()` 优化目录。
- `app/.../api/loader/BaseLoader.java`：`resolveJar`、公开 `ext`、`csp/js/py` 路由。
- `catvod/.../utils/Path.java`：`cache()/files()/tv()/jar(boolean)/jar(String,boolean)/dex()` 等路径工具；`Crypto`、`OkHttp`。
- `app/.../bean/Site.java`：`fetchExt()` 的 `jar://`。
- `app/.../api/parser/LiveParser.java`：`getText(Live)` 的 `jar://`。
- `quickjs/.../utils/Module.java`、`chaquo/src/main/python/app.py`：脚本 `file://` 加载。
- `app/.../utils/PermissionUtil.java`、`app/src/{mobile,leanback}/.../HomeActivity.java`：存储权限与启动引导。
- `app/.../utils/ImgUtil.java`：图片 URL 头后缀解析。
- `AGENTS.md`：本文件。

---

## 7. 验证清单

1. 构建（见第 0 节）应 `BUILD SUCCESSFUL`。
2. 造含 `assets/version`（如 `1`）+ `assets/spider.py` / `assets/index.js` / `assets/demo.json` 的 jar，配置 `;md5;<真实 md5>`：
   - 有外部权限：解压到 `/sdcard/TV/jar/<md5(URL)>/assets/...`，文件管理器可见；版本文件内容与 jar 内一致。
   - 无外部权限（拒绝）：回退 `cache/jar/<md5(URL)>/assets/...`，功能不中断。
3. 版本未变再次加载：跳过解压；把 `assets/version` 改成 `2` 并更新 `;md5;` 后加载：`Path.clear` 后重解压、旧版本删除的条目不再存在，其它 jar 不受影响。
4. jar 内无 `assets/version`（或空）：不建目录、不解压。
5. `jar://assets/*.py` / `*.js` / `ext` 与 LiveConfig live `url=jar://...` 回归。
6. 恶意条目 `../evil.py` 被拒；`MAX_ENTRIES` / `MAX_BYTES` 生效。
7. 非 http、未配置 `;md5;` 的 jar 不解压；`csp_` 与 http 的 `.py`/`.js` 行为不变。
8. `DexClassLoader` 优化目录在内部 `cache/dex`。

---

## 8. 变更历史（要点）

1. 引入 jar 内嵌 py / js / json：`assets/` 约定、`jar://assets/...` 引用、解压与加载器路由。
2. 解压框架演进：共享目录 + `.md5` 标记 → **按 jar 隔离目录 + `assets/version` 版本文件**（弃用 md5 标记）。
3. 曾新增「解压前 assets 预检」，后因版本文件门槛已覆盖而移除。
4. `LiveConfig` 的 live `url`（`lives[].url`）支持 `jar://`。
5. jar 缓存与解压改到外部 `/sdcard/TV/jar`（启动一次性权限引导 + 无权限回退内部）；`DexClassLoader` 优化目录内部化（`Path.dex()`）；放宽 `setReadOnly()` 门槛。
6. `Ikanbot.js` 豆瓣图片 Referer 修复；`ImgUtil` 头后缀支持。
