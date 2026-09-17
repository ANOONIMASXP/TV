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
- **只有配置里的接口原始 scheme 为 `http(s)://` 且配置了 `;md5;<值>` 才会解压 `assets/`**：
  - `"spider": "https://xxx/custom_spider.jar;md5;63ffa738426ba80376f03d03110e0b80"` → 解压。
  - `"spider": "https://xxx/custom_spider.jar"` → 只加载 dex，不解压。
  - `"spider": "file://.../custom_spider.jar;md5;<值>"` → 只加载 dex，不解压。
  - `"spider": "assets://custom_spider.jar;md5;<值>"` → 只加载 dex，不解压（`assets://` 虽经本地 http 中转，但接口原始 scheme 不是 http）。

### 落盘位置

jar 加载时解压到 `cache/jar/<md5>/`，其中 `<md5>` 就是配置 `;md5;` 里写的值（内容校验 md5），保留 `assets/` 前缀。
例如 `"spider": "https://xxx/custom_spider.jar;md5;63ffa738426ba80376f03d03110e0b80"` → `cache/jar/63ffa738426ba80376f03d03110e0b80/assets/...`

| 配置 | jar 内条目 | 解压 / 读取位置 |
|---|---|---|
| `"api": "jar://assets/spider.py"` | `assets/spider.py` | `cache/jar/<md5>/assets/spider.py` |
| `"api": "jar://assets/index.js"` | `assets/index.js` | `cache/jar/<md5>/assets/index.js` |
| `"ext": "jar://assets/demo.json"` | `assets/demo.json` | `cache/jar/<md5>/assets/demo.json` |

> jar 本体仍在 `cache/jar/<md5>.jar`，与解压目录 `cache/jar/<md5>/` 并存，不冲突。

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

- 新增常量：`ASSETS = "assets/"`、`MAX_ENTRIES = 2000`、`MAX_BYTES = 64MB`。
- `load(String key, File file, boolean extract)`：仅当 `extract` 为 true（配置了 `;md5;`）时在创建 `DexClassLoader` 前调用 `extract(file, key)`；未配置 md5 时跳过解压，只加载 dex。
- `parseJar`：`extract = 存在 ;md5; 且接口原始 scheme 为 http(s)`（在 `assets://` 转换前判定），三个加载分支（md5 命中 / http 下载 / file 本地）都按该标志传递。
- 新增 `dirKey(jar)`：解压/读取目录名取 `;md5;` 的字面值；未配置 md5 或 md5 是 http 地址时退回 `Crypto.md5(jar)`。
  `extract()` 与 `file()` 都用同一个 `dirKey`，保证落盘目录与 `jar://` 查找目录一致。
- 新增 `extract(File, String)`：用 `ZipFile` 遍历条目，只处理 `assets/` 前缀的文件，
  解压到 `cache/jar/<md5>/` 并保留完整条目名；含 zip-slip 前缀校验、条目数/字节上限，
  解压前 `Path.clear(root)` 清理陈旧文件。
- 新增 `file(String link, String jar)`：把 `jar://assets/xxx` 映射为
  `cache/jar/<md5>/assets/xxx`，存在则返回文件，否则返回 null。
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
config.spider (jar URL) ──parseJar──► DexClassLoader + extract(仅当原始 http(s) 且配置 ;md5;：assets/** → cache/jar/<md5>/assets/**)
                                              │
site.api = "jar://assets/index.js" ──resolveJar──► file:///.../cache/jar/<md5>/assets/index.js ──► QuickJS
site.api = "jar://assets/spider.py" ─resolveJar──► file:///.../cache/jar/<md5>/assets/spider.py ──► Chaquopy
site.ext = "jar://assets/demo.json" ──ext────────► 读取 cache/jar/<md5>/assets/demo.json 内容作为 extend
site.api = "csp_XXX" ────────────────────────────► 反射实例化 dex 内 com.github.catvod.spider.XXX（不变）
```

---

## 5. 安全与限制

- 仅在接口原始 scheme 为 `http(s)://` 且配置了 `;md5;<值>` 时才解压内嵌资源；`file://`、`assets://` 接口或未配置 md5 的 jar 不会产生解压目录。
- 只解压 `assets/` 前缀条目，保留条目名；含 canonical 路径前缀校验，拒绝 `..` 等越界路径。
- 限制单 jar 解压条目数（2000）与总字节数（64MB），防止解压炸弹。
- 每次加载同一 jar 前先清理其解压目录，jar 更新后不会残留旧文件。
- `jar://` 必须带 `.py` / `.js` 后缀才能被识别并路由；`ext` 必须带 `assets/` 前缀。
- 资源型 jar（无 `classes.dex`）可用：`invokeInit` / `invokeProxy` / JS `createFun` 均已容错。
- 缓存清理：`JarLoader.clear()` 只清内存实例，不清解压文件；解压目录随后续加载覆盖或系统清缓存。

---

## 6. 验证

- 编译（本仓库实际使用，Java 21 工具链会由 Gradle 自动下载）：

  ```
  .\gradlew.bat :app:compileLeanbackDebugJavaWithJavac
  ```

  最近一次结果：`BUILD SUCCESSFUL`（app / catvod / quickjs / chaquo 全部通过，含 Python 任务）。

- 建议真机验证：
  1. 造含 `assets/spider.py`、`assets/index.js`、`assets/demo.json` 的测试 jar，并在 `spider` 中配置 `;md5;<真实 md5>`。
  2. 确认解压到 `cache/jar/<md5>/assets/...`，并确认恶意条目 `../evil.py` 未被写出；再用不带 md5、`file://`、`assets://` 三种 jar 分别验证不会解压。
  3. 分别用 `jar://assets/...` 验证 py / js / ext 三种引用。
  4. 回归 `csp_` 与 http 的 `.py` / `.js` 配置行为不变。

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
