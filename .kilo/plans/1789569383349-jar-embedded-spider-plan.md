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

## 上一轮改动（已实现）

**目标**：`assets://` 与 `file://` 的 jar 一律不解压，只有用户显式配置的 http(s) 才解压。

状态：已实现并编译通过（`JarLoader.parseJar` 在 assets 转换前用 `boolean http = jar.startsWith("http")`，`extract = !md5.isEmpty() && http`）。

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

## 本次任务：Ikanbot.js 图片不显示修复

### 现象

`Ikanbot.js` 返回的 `vod_pic`（豆瓣图床 `img*.doubanio.com`）在 App 列表/详情显示为文字占位，图片不加载。

### 根因

1. 图片加载链路：`ImgUtil.load` → Glide；`OkGlideModule` 用 `OkHttp.client()` 作为 `OkHttpUrlLoader`（`app/.../utils/OkGlideModule.java:29`）。
2. 该 OkHttp 客户端**没有任何默认 User-Agent/Referer 拦截器**（`catvod/.../net/OkHttp.java:189-196` 只加 request/auth/response 拦截器），默认 UA 为 `okhttp/x`。
3. 豆瓣图床对非浏览器请求返回 **418**：实测抓取 `https://img9.doubanio.com/view/photo/s_ratio_poster/public/p2933198755.jpg` 得到非 2xx（418）。Glide 遇非 2xx → `onLoadFailed` → 绘制文字占位（`ImgUtil.java:155-169`）。
4. Ikanbot 页面图片为懒加载：真实地址在 `img[data-src]`，`src` 是 `data:image/svg+xml` 占位；当前 JS 的 `data-src || src` 在 `data-src` 缺失时会把 data 占位当图片。

### 修复方案（改仓库内 `D:\TV-fongmi\docs\Ikanbot.js`）

App 的 `ImgUtil.getUrl` 支持 URL 头后缀：`@Headers=<json>@` / `@Referer=` / `@User-Agent=`（`ImgUtil.java:94-105`）。据此：

1. 新增 `picUrl(url)`：
   - 空值或 `data:` 开头 → 返回 `''`。
   - 其它值追加浏览器 UA：`@Headers={"User-Agent":"<UA>"}@`。
   - host 含 `doubanio.com` 时再加 `Referer: https://movie.douban.com/`。
2. 在 `vodOf(id, name, pic, remarks)`（`docs/Ikanbot.js:75-82`）内把 `vod_pic: stringValue(pic).trim()` 改为 `vod_pic: picUrl(pic)`——一处集中，自动覆盖 `parseBillboard` / `parseVods` / `parseSearch`。
3. 三个解析器的图片取值增加 `data-original`：`img.attr('data-src') || img.attr('data-original') || img.attr('src')`；`data:` 占位由 `picUrl` 过滤。
4. `detail` 的封面（`docs/Ikanbot.js:328`）改为 `vod_pic: picUrl($('meta[property="og:image"]').attr('content'))`。

参考实现：

```js
const DOUBAN_REFERER = 'https://movie.douban.com/';

function picUrl(url) {
    const value = stringValue(url).trim();
    if (!value || value.startsWith('data:')) return '';
    const headers = { 'User-Agent': UA };
    if (/doubanio\.com/i.test(value)) headers['Referer'] = DOUBAN_REFERER;
    return value + '@Headers=' + JSON.stringify(headers) + '@';
}
```

- `docs/Ikanbot.js` 已有 `const UA`（第 37 行）与 `stringValue` 辅助函数，直接复用。
- 无需新增 `imgSrc` 包装：`vodOf` 已集中处理，解析器只需补 `data-original` 回退。

### 关键约束

- `@Headers=` 的 JSON 值内不能含 `@`；当前 UA/Referer 均不含。
- URL 变长且含 `{`/`"`，但 `UrlUtil.convert` 对 http scheme 原样返回、不会破坏后缀（已确认 `UrlUtil.java:21-28,59-67`）。
- 带后缀的 `vod_pic` 会写入历史/收藏，显示时由 `ImgUtil.getUrl` 解析，无副作用。

### 验证

1. 先离线确认头是否有效（由实现代理执行）：对同一图片分别用「仅 UA」「仅 Referer」「UA+Referer」请求，确认返回 200；据此决定是否保留 Referer。
2. 无需 JS 编译；把 `docs/Ikanbot.js` 通过站点 `api`（`file://.../Ikanbot.js`）或 jar `assets/Ikanbot.js` 加载后**冷启动** App（`ImgUtil.failed` 是内存集合，需重启清空）。
3. 确认首页榜单、分类、搜索、详情海报均显示；`data:` 占位不再被当图片加载。
4. 回归：非豆瓣图源（如部分搜索结果的 `ynztctv.com`）仅加 UA 仍能显示。

### 备选（若加头仍 418）

- 换用其它浏览器 UA，或补 `Accept`/`Accept-Language` 头。
- 仍失败则改为经 jar `Proxy` 代理图片（成本高，仅在前者无效时考虑）。

## 本次任务：解压"存在则跳过"（完成标记）

### 目标

仅在 jar 解压目录缺失或 jar 内容变化时解压；否则跳过，避免每次启动/配置重载都 `Path.clear` + 全量解压。

### 方案（已确认）

- 标记文件：`cache/jar/<dirKey>/.extracted`，**内容 = 实际 jar 文件的 md5**（`Crypto.md5(file)`）。
- `dirKey` 不变（配置里 `;md5;` 的字面值；md5 为 http 时回退 `Crypto.md5(整串)`），因此 `jar://` 映射路径 `cache/jar/<dirKey>/assets/...` **不变**。
- 跳过判定与标记都放在 `extract(File, String)` 内，`load`/`parseJar`/`file` 不改。

### 改动点

文件：`app/src/main/java/com/fongmi/android/tv/api/loader/JarLoader.java`

1. 新增常量：`private static final String MARKER = ".extracted";`
2. 改 `extract(File file, String dir)`（当前 `:98-122`）：
   ```java
   private void extract(File file, String dir) {
       File root = new File(Path.jar(), dir);
       String md5 = Crypto.md5(file);
       File marker = new File(root, MARKER);
       if (!md5.isEmpty() && md5.equalsIgnoreCase(Path.read(marker).trim())) return; // 已解压且内容一致 → 跳过
       Path.clear(root);
       root.mkdirs();
       try (ZipFile zip = new ZipFile(file)) {
           // ...现有 assets/** 解压逻辑与 MAX_ENTRIES/MAX_BYTES 上限保持不变...
           if (!md5.isEmpty()) Path.write(marker, md5.getBytes(StandardCharsets.UTF_8)); // 成功后才写标记
       } catch (Throwable e) {
           e.printStackTrace();
           Path.clear(root); // 失败清除半成品，避免残留
       }
   }
   ```
3. 需要 `import java.nio.charset.StandardCharsets;`（`Path.read(File)`/`Path.write(File, byte[])`/`Path.clear` 均已存在：`Path.java:162,206,281`）。

### 行为

- 首次：目录不存在 → 解压 → 写标记。
- 后续启动/重载：标记存在且等于当前 jar 文件 md5 → 跳过（不 clear、不写盘）。
- jar 变化（md5 变）→ 标记不匹配 → clear + 重解压 + 写新标记。
- 半解压（无标记）或标记为空/md5 计算失败 → 不跳过、重解压（安全）。
- 仅 `extract==true`（接口原始 scheme 为 http 且配置了 `;md5;`）时触发；`file://`/`assets://`/无 md5 不受影响。
- `extract` 在 `parseJar` 的 `synchronized(lock)` 内调用，已串行化。

### 风险 / 边界

- 每次启动会对 jar 文件做一次全量 md5（I/O），通常远小于解压成本。
- 因 `MAX_ENTRIES`/`MAX_BYTES` 上限而 `break` 时仍写标记；上限对同一 jar 是确定性的，跳过安全。
- 若外部只删除 `assets/` 却保留 `.extracted`，会误跳过；正常路径 `Path.clear(root)` 会整体删除。如需更保守，可在跳过条件追加 `new File(root, ASSETS).exists()`（代价：不含 `assets/` 的 jar 每次都会重解压）。

### 验证

1. 编译：`.\gradlew.bat :app:compileLeanbackDebugJavaWithJavac`。
2. 配置 http + `;md5;` jar，冷启动一次 → 确认 `cache/jar/<md5>/.extracted` 存在且内容 == jar 文件 md5。
3. 再次冷启动 → 确认未重新解压（目录/文件 mtime 不变）。
4. 更换 jar 内容并更新配置 md5 → 确认重解压且标记更新。
5. 手动删除 `.extracted`（保留 `assets/`）→ 确认重解压。
6. 边界：手动删 `assets/` 下某文件但保留 `.extracted` → 不会自动修复（已知）。

## 需切换到实现代理

- 本次任务（`JarLoader.extract` 完成标记）为源码改动，需切换到可编辑代码的代理执行；本代理仅规划。
- 历史任务状态：`JarLoader` 解压/`jar://`/`file://`、`assets://` 不解压、`docs/Ikanbot.js` 图片头修复均已实现并通过编译/仿真验证，无需再动。
