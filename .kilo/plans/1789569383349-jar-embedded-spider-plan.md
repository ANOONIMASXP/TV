# 支持 jar 内嵌 py/js 爬虫并解压到设备

## 目标

允许同一个 `jar` 除 `classes.dex`（`Init`/`Proxy`/`csp_*` 类）外，还内嵌 `.py` / `.js` 爬虫文件。加载 jar 时把这些文件解压到设备 cache 目录，站点通过 `jar://` 协议引用它们。

## 已确认决策

1. **引用协议**：站点 `api` 使用 `jar://<jar内相对路径>`，例如 `jar://spider.py`、`jar://lib/x.js`。
2. **解压布局**：按 jar 分子目录并保留相对路径：
   - `cache/py/<md5(jar)>/spider.py`
   - `cache/js/<md5(jar)>/lib/x.js`
3. **解析形态**：`jar://` 解析为本地 `file://` 绝对路径后再交给 JS/Python 加载器，例如 `file:///data/.../cache/js/<md5>/lib/x.js`。

## 配置文件写法

`jar://` 后面的路径必须**与 jar 内 zip 条目路径完全一致**，且必须带 `.py` / `.js` 后缀。

jar 源可放顶层 `spider`，也可放站点自身 `jar`：

```json
{
  "spider": "http://host/xxx.jar;md5;<hash>",
  "sites": [
    {
      "key": "豆瓣",
      "name": "豆瓣",
      "type": 3,
      "api": "jar://spider.py",
      "changeable": 1,
      "searchable": 0,
      "ext": ""
    },
    {
      "key": "豆瓣JS",
      "name": "豆瓣JS",
      "type": 3,
      "api": "jar://lib/douban.js",
      "changeable": 1,
      "searchable": 0,
      "ext": ""
    },
    {
      "key": "豆瓣Dex",
      "name": "豆瓣Dex",
      "type": 3,
      "api": "csp_Douban",
      "changeable": 1,
      "searchable": 0,
      "ext": ""
    }
  ]
}
```

- `api: "jar://spider.py"` → jar 根条目 `spider.py` → 解压到 `cache/py/<md5>/spider.py`。
- `api: "jar://lib/douban.js"` → jar 条目 `lib/douban.js` → 解压到 `cache/js/<md5>/lib/douban.js`。
- `api: "csp_Douban"` → 保持原样，加载 dex 内 `com.github.catvod.spider.Douban`。
- 若条目为 `assets/spider.py`，`api` 必须写 `jar://assets/spider.py`（不做前缀剥离）。
- 顶层 `spider` 与站点 `jar` 至少存在一个，否则 `jar://` 无来源可解析。
- 需要 `getDependence` 的 py 依赖同样放进 jar（如根条目 `dep.py`），`getDependence` 返回 `dep`，运行时从同目录 `file://` 复制到平铺 `cache/py/dep.py` 再加载。

## 现状（关键事实）

- `JarLoader.load()` 用 `DexClassLoader` 加载 jar，只解析 `classes.dex`；其余 zip 条目可读但被忽略（`app/.../api/loader/JarLoader.java:54-62`）。
- 站点按 `api` 路由：`csp_*`→JarLoader；`.js`→JsLoader（会 `dex(jar)` 取 `com.github.catvod.js.Function`）；`.py`→PyLoader（**完全不触发 jar 加载**）（`BaseLoader.java:58-63`）。
- JS 内容经 `Module.fetch` 获取，仅支持 `http`/`assets`/`lib/`，无本地文件分支（`quickjs/.../utils/Module.java:22-29`）。
- JS 嵌套模块由 `BytecodeModuleLoader.moduleNormalizeName` 用 `UriUtil.resolve(base, ref)` 解析相对路径，因此 `file://` 主路径可正确解析同目录模块（`quickjs/.../crawler/Spider.java:166-176`）。
- Python 入口 `app.py` 的 `spider(cache, api)` 非 http 时把 api 字符串当源码写入；`download(path, api)` 也只会处理 http/内联（`chaquo/src/main/python/app.py:7-20`）。
- Python 依赖：Java `Spider.init` → `getDependence` → `download(name)`，`url = UriUtil.resolve(api, name)`，落盘到 `Path.py(name)`（**平铺** `cache/py/<name>.py`），再由 `base/spider.py loadModule` 从平铺目录加载（`chaquo/.../Spider.java:31-36,132-136`、`base/spider.py:72-78`）。
- 已有 zip 解压参考与 zip-slip 防护：`FileUtil.zipDecompress`（`app/.../utils/FileUtil.java:176-207`）。

## 改动文件

- `app/src/main/java/com/fongmi/android/tv/api/loader/JarLoader.java` — 解压入口。
- `app/src/main/java/com/fongmi/android/tv/api/loader/BaseLoader.java` — `jar://` 解析与 jar 加载触发。
- `quickjs/src/main/java/com/fongmi/quickjs/utils/Module.java` — 增加 `file://` 读取分支。
- `chaquo/src/main/python/app.py` — `spider()` / `download()` 支持 `file://`。
- 复用 `catvod/.../utils/Path.java`（`py()`/`js()`/`read(File)`）、`Crypto.md5`。

## 实施任务（按顺序）

1. **JarLoader 增加解压**
   - 在 `load(String key, File file)` 中、`file.setReadOnly()` 校验通过后，调用新增 `extract(file, key)`。
   - `extract` 用 `java.util.zip.ZipFile` 遍历 `entries()`：
     - 仅处理名字以 `.py` 或 `.js` 结尾的文件条目；跳过目录。
     - 目标根：`.py` → `new File(Path.py(), key)`；`.js` → `new File(Path.js(), key)`。
     - 解压前先 `Path.clear(root)` 再 `mkdirs()`（清除同一 jar 上次的陈旧文件，保证 jar 变更后无残留）。
     - zip-slip 防护：`out.getCanonicalPath()` 必须以 `root.getCanonicalPath() + File.separator` 开头，否则跳过。
     - 限制条目数与总字节数（参考 `FileUtil.zipDecompress` 的 `maxEntries`/`maxBytes`）。
     - 用 `Path.write(out, zip.getInputStream(entry))` 或 `Path.copy(InputStream, File)` 落盘。
   - 保持现有顺序：解压不依赖 dex 加载成功，资源型 jar（无 `classes.dex`）也应可用；`invokeInit`/`invokeProxy` 已 `try/catch`，无需改动。

2. **BaseLoader 增加 `jar://` 解析并在路由前触发 jar 加载**
   - 在 `getSpider(String key, String api, String ext, String jar)` 开头解析：
     ```java
     if (api.startsWith("jar://")) api = resolveJar(api, jar);
     ```
   - `resolveJar(api, jar)`：
     - `String rel = api.substring("jar://".length());`
     - `String key = Crypto.md5(jar);`
     - `jarLoader.dex(jar);` —— 触发 `parseJar`→`load`→`extract`（对 py 也生效）。
     - `File base = rel.endsWith(".py") ? Path.py(key) : Path.js(key);`
     - `return "file://" + new File(base, rel).getAbsolutePath();`
     - 若 `jar` 为空或加载后目标文件不存在：返回原 `api`（后续路由到 `SpiderNull`），并打印日志。
   - 注意解析必须发生在 `isPy/isJs/isCsp` 判断之前（`file://.../x.py` 仍包含 `.py`，路由不变）。
   - `setRecent` 无需改（对 `jar://x.py`/`jar://x.js` 的 `isPy`/`isJs` 判断仍成立）。
   - `Path.py(key)`/`Path.js(key)` 目录可能不存在，`resolveJar` 只返回路径，目录由步骤 1 解压时创建；为稳妥可在返回前 `mkdirs()`。

3. **Module.fetch 增加 `file://` 分支**（`quickjs/.../utils/Module.java`）
   - 在 `assets`/`lib` 分支后新增：
     ```java
     else if (name.startsWith("file")) cache.put(name, content = Path.read(new File(name.substring("file://".length()))));
     ```
   - 该分支同时服务主脚本与嵌套 `import`（`moduleNormalizeName` 会产出 `file://` 相对路径）。

4. **app.py 支持 `file://`**（`chaquo/src/main/python/app.py`）
   - `spider(cache, api)`：
     - 若 `api.startswith('file://')`：`path = api[7:]`，**不再** `download`（文件已在设备）。
     - 否则维持原逻辑（http 下载 / 内联源码写文件）。
     - 模块名用避免冲突的取值（见风险 4），再 `SourceFileLoader(name, path).load_module().Spider()`。
   - `download(path, api)`：
     - 若 `api.startswith('file://')`：读取 `api[7:]` 本地文件字节写入 `path`。
     - 否则维持 http / 内联分支。
   - 依赖解析：Java `Spider.download` 用 `UriUtil.resolve(file://主路径, name)` 得到同目录 `file://` 依赖路径，再经 app.py `download` 复制到平铺 `Path.py(name)`，由 `loadModule` 加载 —— 无需改 `chaquo/.../Spider.java`。

## 数据流

```
config.spider (jar URL) ──parseJar──> DexClassLoader + extract(.py/.js)
                                             │
site.api = "jar://lib/x.js" ──resolveJar──> file:///.../cache/js/<md5>/lib/x.js ──> JsLoader/QuickJS (Module.fetch file分支)
site.api = "jar://spider.py" ─resolveJar──> file:///.../cache/py/<md5>/spider.py ──> PyLoader/Chaquopy (app.py file分支)
```

## 风险与失败模式

1. **zip-slip / 解压炸弹**：必须做 canonical 前缀校验与条目数/字节上限；拒绝含 `..`、绝对路径的条目。
2. **陈旧文件**：jar 更新后旧条目残留 → 每次 `load` 前 `Path.clear(root)` 再解压。
3. **`jar://` 路径与 jar 条目不一致**：api 路径必须等于 jar 内条目名；文档说明不做前缀剥离（如条目为 `assets/spider.py`，则写 `jar://assets/spider.py`）。
4. **Python 模块名冲突**：两个 jar 都有 `spider.py` 时 `SourceFileLoader` 的 `sys.modules` 键冲突。建议模块名包含 jar 子目录（如 `md5(jar)[:8] + "_" + stem`）；不改会沿用现有平铺冲突行为。
5. **无 jar / jar 未定义**：`jar://` 无可解析来源 → 返回原 api → `SpiderNull`；记录日志。
6. **资源型 jar**：允许无 `classes.dex`；`invokeInit`/`invokeProxy`/JS `createFun` 均已 `try/catch`。
7. **缓存清理**：`JarLoader.clear()` 清内存实例，不清解压文件；解压目录随系统 cache 清理或下一次 `load` 覆盖。

## 验证

1. 造测试 jar：`spider.py`、`lib/x.js`（内含相对 `import`），可选打入 `classes.dex`。
2. 造恶意条目 `../evil.py`，确认解压后不越出 `cache/py/<md5>/`。
3. 配置：`spider: "http://host/test.jar;md5;<hash>"`；站点 `api` 分别为 `jar://spider.py`、`jar://lib/x.js`、`csp_Test`（回归验证 jar 内 dex 类）。
4. 设备端 `adb shell` 确认 `cache/py/<md5>/spider.py`、`cache/js/<md5>/lib/x.js` 存在且内容正确。
5. JS：确认主脚本执行、嵌套 `import` 的模块经 `file://` 正确加载。
6. Python：确认主脚本从 `file://` 加载；`getDependence` 返回的依赖能从同目录 `file://` 复制并在平铺目录加载。
7. 回归：纯 `csp_` jar、纯 `http` 的 `api...py`/`api...js` 行为不变。
8. 运行项目现有构建/lint（`gradlew assembleLeanbackDebug` 或仓库既定命令）。

## 需切换到实现代理

本计划涉及源码改动（Java/Python/Gradle 构建），需切换到可编辑代码的代理执行。
