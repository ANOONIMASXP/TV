# jar 内嵌爬虫（py / js / 其他资源）— 最终方案与待办

> 本话题大部分改动已实现并通过编译。本文件记录**最终约定**与**唯一待实现改动**，供实现代理执行而不回归既有行为。

## 最终约定（已实现，勿回归）

1. **jar 内容**：爬虫资源只放在 jar 的 `assets/` 目录下、无二级目录，例如
   `assets/Ikanbot.js`、`assets/spider.py`、`assets/demo.json`。
2. **引用协议**：站点 `api` / `ext` 写作 `jar://assets/<文件名>`，路径与 zip 条目完全一致（含 `assets/` 前缀）。
3. **落盘目录**：`cache/jar/<md5>/assets/...`，其中 `<md5>` 取配置 `;md5;` 的**字面值**（`JarLoader.dirKey`）；
   未配置 md5 或 md5 为 http 地址时退回 `Crypto.md5(jar)`。
4. **解析为 `file://`**：py/js 的 `jar://` 先解析为本地 `file://` 绝对路径，再交给 Python / QuickJS 加载器。
5. **ext 读取**：`ext: "jar://assets/demo.json"` 读取解压文件内容作为 `extend`。
6. **解压条件**：仅当**接口原始 scheme 为 `http(s)://` 且配置了 `;md5;<值>`** 才解压。
   - `https://xxx.jar;md5;<值>` → 解压。
   - `https://xxx.jar`（无 md5）→ 不解压。
   - `file://...jar;md5;<值>` → 不解压。
   - `assets://...jar;md5;<值>` → 不解压（`assets://` 是 APK 内资源，经本地 http 中转，但接口不是 http，见待办）。
7. **安全**：只解压 `assets/` 前缀条目、保留条目名；canonical 前缀校验防 zip-slip；条目数上限 2000、总字节上限 64MB；解压前 `Path.clear(root)`。

## 关键文件与已实现点

- `app/src/main/java/com/fongmi/android/tv/api/loader/JarLoader.java`
  - `dirKey(jar)`：解压/查找目录名。
  - `load(key, file, extract, dir)`：`extract` 为真时调用 `extract(file, dir)`。
  - `extract(file, dir)`：`ZipFile` 遍历 `assets/**` → `cache/jar/<dir>/assets/**`。
  - `file(link, jar)` / `ext(link, jar)`：`jar://assets/xxx` → `cache/jar/<dirKey>/assets/xxx`。
  - `parseJar(key, jar)`：解析 `;md5;`、走 md5 命中 / http 下载 / file 本地三分支。
- `app/src/main/java/com/fongmi/android/tv/api/loader/BaseLoader.java`
  - `getSpider(...)`：`jar://` api 先 `resolveJar`；`jar://` ext 先解析内容。
  - `resolveJar(api, jar)`：仅 `.py`/`.js`，返回 `file://` 绝对路径，失败返回空串→`SpiderNull`。
  - 公开 `ext(ext, jar)`。
- `app/src/main/java/com/fongmi/android/tv/bean/Site.java`：`fetchExt()` 支持 `jar://`。
- `quickjs/src/main/java/com/fongmi/quickjs/utils/Module.java`：`fetch` 增加 `file://` 分支。
- `chaquo/src/main/python/app.py`：`spider()` / `download()` 支持 `file://`，模块名防冲突。
- `catvod/src/main/java/com/github/catvod/utils/Path.java`：无净改动。

## 待实现（唯一改动）

**目标**：`assets://` 与 `file://` 的 jar 一律不解压，只有用户显式配置的 http(s) 才解压。

**文件**：`app/src/main/java/com/fongmi/android/tv/api/loader/JarLoader.java` — `parseJar(String key, String jar)`

**实现**：在 `assets://` 被转换成 http 之前先记录原始 scheme，用它判断：

```java
public void parseJar(String key, String jar) {
    if (loaders.containsKey(key)) return;
    boolean http = jar.startsWith("http");           // 新增：基于原始接口判断
    if (jar.startsWith("assets")) jar = UrlUtil.convert(jar);
    Object lock = locks.computeIfAbsent(key, k -> new Object());
    synchronized (lock) {
        if (loaders.containsKey(key)) return;
        String dir = dirKey(jar);
        String[] texts = jar.split(";md5;");
        String md5 = texts.length > 1 ? texts[1].trim() : "";
        boolean extract = !md5.isEmpty() && http;     // 修改：原为 texts[0].startsWith("http")
        if (md5.startsWith("http")) md5 = OkHttp.string(md5).trim();
        jar = texts[0];
        if (!md5.isEmpty() && Crypto.equals(Path.jar(jar), md5)) {
            load(key, Path.jar(jar), extract, dir);
        } else if (jar.startsWith("http")) {
            load(key, Download.create(jar, Path.jar(jar)).get(), extract, dir);
        } else if (jar.startsWith("file")) {
            load(key, Path.local(jar), extract, dir);
        }
    }
}
```

要点：
- `http` 必须在 `assets://` 转换前计算；`https://...` 也以 `http` 前缀命中。
- md5 字段为 http 地址（动态 md5）不影响：`extract` 只看原始接口 scheme 与 md5 是否存在。
- `assets://` 仍按原逻辑经本地 http 下载 jar 到 cache 并加载 dex，只是不解压。

## 验证

1. 编译：`.\gradlew.bat :app:compileLeanbackDebugJavaWithJavac`，期望 `BUILD SUCCESSFUL`。
2. 造含 `assets/Ikanbot.js`、`assets/spider.py`、`assets/demo.json` 的测试 jar：
   - `https://.../test.jar;md5;<真实md5>` → 解压到 `cache/jar/<md5>/assets/...`，`jar://assets/...` 可解析。
   - `https://.../test.jar`（无 md5）→ 不解压、无解压目录。
   - `file://.../test.jar;md5;<值>` → 加载 dex，不解压。
   - `assets://test.jar;md5;<值>` → 加载 dex，不解压。
3. 回归 `csp_` 与 http `.py`/`.js` 配置行为不变。

## 需切换到实现代理

上述 `JarLoader.parseJar` 为源码改动，需切换到可编辑代码的代理执行；本代理仅规划。
