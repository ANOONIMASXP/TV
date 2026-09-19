# jar 内嵌爬虫（py / js / 其他资源）说明

本文件汇总本话题全部改动：让同一个 `jar` 除 dex 外还能内嵌 `.py` / `.js` / `.json` 等资源，
加载 jar 时把这些资源解压到设备，站点通过 `jar://` 协议引用。

---

## 1. 协议约定

- 所有内嵌爬虫资源只放在 jar 的 `assets/` 目录下，**无二级目录**，例如：
  - `assets/spider.py`
  - `assets/index.js`
  - `assets/demo.json`
- 站点引用统一写作 `jar://assets/<文件名>`，路径必须与 jar 内条目完全一致（含 `assets/` 前缀）。
- jar 源由配置顶层 `spider` 或站点自身 `jar` 提供（两者至少一个）。
- **只有配置里的接口原始 scheme 为 `http(s)://`、配置了 `;md5;<值>`，且 jar 内提供了 `assets/version` 版本文件，才会解压 `assets/`**：
  - `"spider": "https://xxx/custom_spider.jar;md5;63ffa738426ba80376f03d03110e0b80"` → 解压。
  - `"spider": "https://xxx/custom_spider.jar"` → 只加载 dex，不解压。
  - `"spider": "file://.../custom_spider.jar;md5;<值>"` → 只加载 dex，不解压。
  - `"spider": "assets://custom_spider.jar;md5;<值>"` → 只加载 dex，不解压（`assets://` 虽经本地 http 中转，但接口原始 scheme 不是 http）。

### jar 本体缓存与更新判断

- **缓存位置与命名**：jar 本体缓存在 `cache/jar/<jar URL 的 md5>.jar`。
  这个哈希是 **jar 下载 URL 字符串**（配置里 `;md5;` 之前的部分，如 `https://xxx/custom_spider.jar`）经 `Crypto.md5(String)` 得到的结果，**不是**配置 `;md5;` 值，也不是 jar 文件内容的 md5。
  作用：每个 jar URL 固定一个缓存槽；同一 URL 升级时下载后直接覆盖该文件，不按版本堆积。
  例如 `"spider": "https://xxx/custom_spider.jar;md5;63ff..."` → 缓存文件名是 `md5("https://xxx/custom_spider.jar") + ".jar"`。
- **更新判断（内容校验，不靠文件名）**：`JarLoader.parseJar` 用 `Crypto.equals(Path.jar(jar), md5)`（`Crypto.equals(File, String)` 内部对缓存文件实际算 md5）与**解析后的配置 `;md5;`** 比对：
  - 相等 → 缓存命中，直接加载缓存 jar，不重新下载。
  - 不相等或文件不存在 → 从 URL 重新下载，落到同一个缓存文件并覆盖（`Path.create` 会先清空）。
- **版本信号来源**：
  1. 配置里 `;md5;` 字面值变化（换新 jar）；
  2. `;md5;` 写成 http 地址时，每次加载先 `OkHttp.string(md5)` 取回当前期望 md5，远端变化即触发重新下载。
- **注意**：若配置 `;md5;` 不变、但 URL 上的 jar 内容被替换，**不会**被判定为更新（命中缓存直接使用旧文件）。这是「md5 即版本契约」的设计。

### 解压与落盘位置

> 解压流程位于上面的「jar 本体缓存与更新判断」**之后**：先由 `parseJar` 确定最终 jar（缓存命中或重新下载覆盖），再对该 jar 解压。`file()` 也是先生成（含解压）再拼路径。

- 解压目录**按 jar 隔离**：`cache/jar/<jar URL 的 md5>/assets/xxx`（与 jar 本体同名，保留 `assets/` 前缀）。**不再**使用共享目录 `cache/jar/assets/`。
  例如 `"spider": "https://xxx/custom_spider.jar;md5;63ff..."` → `cache/jar/md5("https://xxx/custom_spider.jar")/assets/...`

| 配置 | jar 内条目 | 解压 / 读取位置 |
|---|---|---|
| `"api": "jar://assets/spider.py"` | `assets/spider.py` | `cache/jar/<md5(URL)>/assets/spider.py` |
| `"api": "jar://assets/index.js"` | `assets/index.js` | `cache/jar/<md5(URL)>/assets/index.js` |
| `"ext": "jar://assets/demo.json"` | `assets/demo.json` | `cache/jar/<md5(URL)>/assets/demo.json` |

- **版本文件驱动更新**：jar 内提供 `assets/version`（作者维护，内容为版本号字符串）。加载时读取该文件，与缓存侧 `cache/jar/<md5(URL)>/assets/version` 内容对比：
  - **不相等**（或缓存侧文件缺失）→ 先 `Path.clear` 掉该 jar 目录，再解压覆盖，最后写入版本文件；
  - **相等** → 跳过解压；
  - jar 内**没有** `assets/version`（或内容为空）→ **不解压、不建目录**。
- 解压时跳过 zip 内的 `assets/version` 条目，等拷贝循环结束后再写该文件，确保中途失败时旧记录不被误更新、下次可重试。
- `;md5;` 只用于 jar 本体的下载/校验，**不参与** assets 是否更新的判断；`<md5(URL)>` 取自 jar URL（`assets://` 先转本地 http，再去 `;md5;`），与 jar 本体缓存名一致。

> jar 本体 `cache/jar/<md5(URL)>.jar` 与解压目录 `cache/jar/<md5(URL)>/` 并存，互不冲突。

---

## 2. 配置示例

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

- `api` 为 `csp_XXX` 时行为不变：加载 dex 内 `com.github.catvod.spider.XXX`。
- `api` 为 `jar://assets/xxx.py` / `.js` 时：解析为本地 `file://` 路径后交给 Python / QuickJS 加载器。
- `ext` 为 `jar://assets/xxx.json` 时：读取解压后文件内容，作为 `extend` 传给 `spider.init`。

---

## 3. 改动明细

### 3.1 `app/src/main/java/com/fongmi/android/tv/api/loader/JarLoader.java`

- 常量：`ASSETS = "assets/"`、`VERSION = "version"`、`MAX_ENTRIES = 2000`、`MAX_BYTES = 64MB`。
- `load(String key, File file, boolean extract)`：仅当 `extract` 为 true 时在创建 `DexClassLoader` 前调用 `extract(file)`；`extract` 不再接收 md5。
- `parseJar`：`extract = 存在 ;md5; 且接口原始 scheme 为 http(s)`（在 `assets://` 转换前判定），先按 `;md5;` 决定用缓存 jar 还是重新下载覆盖，再在三个分支调用 `load(key, file, extract)`；最后才解压。
- `extract(File)`：由 jar 文件名的 stem 推导目录 `cache/jar/<md5(URL)>/`，用 `ZipFile` 遍历条目，只处理 `assets/` 前缀的文件（保留前缀）解压到该目录；含 zip-slip 前缀校验、条目数/字节上限。
  更新判断改用 **jar 内 `assets/version`**：与 `cache/jar/<md5(URL)>/assets/version` 内容对比，**不相等（或缓存缺失）才 `Path.clear` 目录后解压覆盖**，成功且最后写入版本文件；相等则跳过；jar 内无 `assets/version`（或空）则不解压、不建目录。
- `file(String link, String jar)`：把 `jar://assets/xxx` 映射为
  `cache/jar/<md5(URL)>/assets/xxx`（`url(jar)` 与 `parseJar` 的 URL 推导一致），存在则返回文件，否则返回 null。
- 新增 `ext(String link, String jar)`：用 `file()` 定位后读取文本内容返回；找不到则原样返回。

### 3.2 `app/src/main/java/com/fongmi/android/tv/api/loader/BaseLoader.java`

- `getSpider(...)`：
  - 若 `api` 以 `jar://` 开头，先 `resolveJar(api, jar)`；解析失败（空串）返回 `SpiderNull`。
  - 若 `ext` 以 `jar://` 开头，替换为 `jarLoader.ext(ext, jar)` 的内容。
  - 之后按原有 `isPy / isJs / isCsp` 路由，行为对现有 `csp_` / http `.py` / `.js` 不变。
- 新增 `resolveJar(String api, String jar)`：仅对 `.py` / `.js` 解析，返回 `file://<绝对路径>`。
- 新增公开 `ext(String ext, String jar)`：供 `Site.fetchExt()` 使用。

### 3.3 `app/src/main/java/com/fongmi/android/tv/bean/Site.java`

- `fetchExt()`：新增 `jar://` 分支，读取 jar 内资源内容替换 `ext`（type 4 场景）；
  原有 `http` 行为不变。

### 3.4 `quickjs/src/main/java/com/fongmi/quickjs/utils/Module.java`

- `fetch(String name)` 新增本地文件分支：
  `file://` → `Path.read(new File(name.substring("file://".length())))`。
- 主脚本与嵌套 `import`（`BytecodeModuleLoader.moduleNormalizeName` 会产出 `file://` 相对路径）都可读取。

### 3.5 `chaquo/src/main/python/app.py`

- `spider(cache, api)`：`api` 为 `file://` 时直接使用该本地文件路径（不再下载）；
  模块名在存在父目录时用 `<父目录名>_<文件名>`，避免不同 jar 同名脚本的 `sys.modules` 冲突；
  平铺的 http / 内联场景保持原模块名。
- `download(path, api)`：`api` 为 `file://` 时从本地文件读取字节写入目标路径；
  原有 http / 内联分支不变。
- Java 侧 `com.fongmi.chaquo.Spider.download` 无需改动：依赖经
  `UriUtil.resolve(file://主脚本, name)` 落到同目录 `file://`，再由 `download` 复制。

### 3.6 其他

- `.gitignore`：新增 `.kilo`（忽略 Kilo 本地计划/配置目录）。
- `.kilo/plans/1789569383349-jar-embedded-spider-plan.md`：本功能的实施计划（未纳入版本控制）。

> `catvod/.../utils/Path.java` 最终无净改动（曾新增的 `assets()` 辅助方法在后续调整中已移除）。

---

## 4. 运行流程

```
config.spider (jar URL) ──parseJar──► 先按 ;md5; 决定 jar 本体（缓存命中 / 重新下载覆盖 cache/jar/<md5(URL)>.jar）
                                              │  再随后解压（仅当原始 http(s) 且配置 ;md5; 且 jar 内有 assets/version）：
                                              │  assets/** → cache/jar/<md5(URL)>/assets/**，版本记录 assets/version
                                              ▼
site.api = "jar://assets/index.js" ──resolveJar──► file:///.../cache/jar/<md5(URL)>/assets/index.js ──► QuickJS
site.api = "jar://assets/spider.py" ─resolveJar──► file:///.../cache/jar/<md5(URL)>/assets/spider.py ──► Chaquopy
site.ext = "jar://assets/demo.json" ──ext────────► 读取 cache/jar/<md5(URL)>/assets/demo.json 内容作为 extend
site.api = "csp_XXX" ────────────────────────────► 反射实例化 dex 内 com.github.catvod.spider.XXX（不变）
```

---

## 5. 安全与限制

- 只在接口原始 scheme 为 `http(s)://`、配置了 `;md5;<值>`，且 jar 内提供 `assets/version` 时才解压内嵌资源；`file://`、`assets://` 接口、未配置 md5 或缺少版本文件的 jar 不会产生解压文件。
- 只解压 `assets/` 前缀条目，保留前缀落盘到按 jar 隔离的目录 `cache/jar/<md5(URL)>/assets/`；含 canonical 路径前缀校验，拒绝 `..` 等越界路径。
- **版本文件门槛**：jar 内没有 `assets/version`（或内容为空）时不执行任何解压动作（不建目录、不拷贝、不写记录）。
- 限制单 jar 解压条目数（2000）与总字节数（64MB），防止解压炸弹。
- 版本文件 `cache/jar/<md5(URL)>/assets/version` 在解压成功且最后写入；加载时与 jar 内 `assets/version` 对比，**不相等（或缓存缺失）才 `Path.clear` 目录后解压覆盖**，相等则跳过，避免每次启动都重复 I/O。
- **按 jar 隔离、版本变化才 clear**：每个 jar 独立目录，解压前 `Path.clear` 仅清本 jar 目录，不影响其它 jar；版本变化时清理旧文件，避免残留。
- `jar://` 必须带 `.py` / `.js` 后缀才能被识别并路由；`ext` 必须带 `assets/` 前缀。
- 资源型 jar（无 `classes.dex`）可用：`invokeInit` / `invokeProxy` / JS `createFun` 均已容错。
- 缓存清理：`JarLoader.clear()` 只清内存实例，不清解压文件；解压文件随后续版本变化覆盖或系统清缓存。

---

## 6. 验证

- 编译（本仓库实际使用，Java 21 工具链会由 Gradle 自动下载）：

  ```
  .\gradlew.bat :app:compileLeanbackDebugJavaWithJavac
  ```

  最近一次结果：`BUILD SUCCESSFUL`（app / catvod / quickjs / chaquo 全部通过，含 Python 任务）。

- 建议真机验证：
  1. 造含 `assets/version`（如内容 `1`）、`assets/spider.py`、`assets/index.js`、`assets/demo.json` 的测试 jar，并在 `spider` 中配置 `;md5;<真实 md5>`。
  2. 确认解压到 `cache/jar/<md5(URL)>/assets/...`，且 `assets/version` 内容与 jar 内一致；确认恶意条目 `../evil.py` 未被写出；再用不带 md5、`file://`、`assets://` 三种 jar 分别验证不会解压。
  3. 版本未变再次加载：版本文件内容相等 → 跳过解压；把 jar 内 `assets/version` 改成 `2` 并更新 `;md5;` 后加载：确认 `Path.clear` 后重解压、版本文件更新、旧版本已删除的条目不再存在。
  4. jar 内**没有** `assets/version`（或内容为空）：不创建 `cache/jar/<md5(URL)>/`、不拷贝任何文件。
  5. 分别用 `jar://assets/...` 验证 py / js / ext 三种引用。
  6. 回归 `csp_` 与 http 的 `.py` / `.js` 配置行为不变。

---

## 7. 新增文件：`Ikanbot.js`

位置：`D:\Spider\Ikanbot.js`（与参考 `D:\Spider\spider.js` 同目录，未纳入本仓库版本控制）。

- 参考 `D:\Spider\app\src\main\java\com\github\catvod\spider\Ikanbot.java` 与最新站点
  `https://www1.ikanbot.com` 重写为 QuickJS 爬虫。
- 遵循 QuickJS 契约：`export default createSpider`，数据方法返回 JSON 字符串。
- HTML 解析使用内建模块 `assets://js/lib/cat.js` 导出的 `cheerio`；HTTP 使用全局 `req`。
- 站点结构：榜单 `div.item-root`、分类 `a.item`、搜索 `a.cover-link`、详情
  `h1#video_title` / `og:image` / `div.detail > h3` / `#current_id` / `#e_token` / `#mtype`。
- 播放线路按现站逻辑：`/api/getResN?videoId=&mtype=&token=`，`token` 由 `get_tks` 从
  `current_id` 末四码与 `e_token` 推导；`resData` 解析为 `[{url,newName}]`，`url` 按 `#` 拆集、`$` 拆
  “名称$链接”，多线路用 `$$$` 分隔。
- 若放进 jar：置于 `assets/Ikanbot.js`，站点 `api` 写 `jar://assets/Ikanbot.js`。

配置示例：

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
