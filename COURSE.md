# 书源规则编写教程

> 适用于 StarBox Fiction 引擎 v4.0 | 2026-05-07

## 一、什么是书源规则

书源规则是一段 JSON，告诉引擎如何从小说网站提取数据。支持三种提取方式：

| 前缀 | 引擎 | 适用场景 |
|------|------|---------|
| 无前缀 | Jsoup XPath | 静态 HTML 页面 |
| `@css:` | Jsoup CSS Selector | 静态 HTML，CSS 比 XPath 更直观 |
| `@js:` | WebView / Java 内置 | DOM 操作、加密 URL、动态页面 |
| `@json:` | Java JSONPath | JSON API 接口 |

同一规则的不同字段可以混用不同前缀。

---

## 二、JSON 完整结构

```json
{
  "name": "书源名称",
  "domain": "网站域名",
  "charset": "utf-8",
  "userAgent": "",
  "useWebView": false,
  "sslVerify": true,
  "search": { ... },
  "detail": { ... },
  "content": { ... }
}
```

| 字段 | 类型 | 必填 | 默认 | 说明 |
|------|------|------|------|------|
| `name` | string | 是 | — | 自定义书源名称 |
| `domain` | string | 是 | — | 网站域名 |
| `charset` | string | 否 | `utf-8` | 网页编码，可选 `gbk` |
| `userAgent` | string | 否 | 系统默认 | 自定义 UA |
| `useWebView` | boolean | 否 | `false` | 是否用 WebView 加载页面 |
| `sslVerify` | boolean | 否 | `true` | 是否验证 SSL 证书，过期证书站需关掉 |
| `search` | object | 是 | — | 搜索规则 |
| `detail` | object | 是 | — | 详情页规则 |
| `content` | object | 是 | — | 正文规则 |

### 何时开启 `useWebView`

- 网站有反爬保护（Cloudflare、浏览器验证）
- 页面内容由 JS 动态渲染（Vue/React）
- 章节列表通过 AJAX 加载

**注意：** WebView 比 OkHttp 慢很多，普通静态网站不要开。

### 何时关闭 `sslVerify`

- 网站 SSL 证书过期（如某些小站）
- 自签名证书

关闭后引擎信任所有证书。

---

## 三、前缀详解

### XPath（无前缀，默认）

所有未加前缀的字段值被当作 XPath 处理，使用 Jsoup 解析。

### `@css:` — CSS 选择器

Java 层执行，零额外开销。属性提取用 `/@attr` 后缀：

```json
"list": "@css:tbody#bookList tr",
"name": "@css:td.name a",
"detailUrl": "@css:td.name a/@href",
"cover": "@css:img/@data-src"
```

### `@js:` — JavaScript 提取

**WebView 模式：** 代码含 DOM API（`document`、`querySelector` 等）时，在 WebView 中执行，返回 JSON 字符串。

```json
"name": "@js:(function(){return document.querySelector('h1').textContent})()"
```

**内置模式：** 代码只含内置函数（加密/编码）时，Java 层直接执行，无需 WebView：

```json
"url": "@js:'/k-'+encodeURIComponent(btoa(AES.encrypt('{{key}}','password')))+'.html'"
```

| 内置函数 | 用途 |
|---------|------|
| `btoa(str)` | Base64 编码 |
| `encodeURIComponent(str)` | URL 编码 |
| `AES.encrypt(key, password)` | AES/CBC 加密，password 自动补位做 key/IV |

### `@json:` — JSONPath 提取

用于 JSON API 接口，Gson 解析后用点号路径提取字段。

**基础路径：**
```json
"list": "@json:data.list",
"name": "@json:name",
"author": "@json:data.author"
```

**数组索引：**
```json
"name": "@json:data.list[0].name"
```

**字符串模板（拼接 URL）：**
```json
"detailUrl": "@json:'https://api.example.com/detail?id='+id",
"chapterUrl": "@json:'/api/content?bid='+bid+'&cid='+id"
```

---

## 四、Search（搜索规则）

```json
"search": {
  "url": "https://example.com/search?keyword={{key}}",
  "method": "GET",
  "body": "",
  "list": "//table[@class='grid']//tr[td]",
  "name": "td[1]/a",
  "author": "td[2]",
  "cover": "",
  "detailUrl": "td[1]/a/@href",
  "latestChapter": "td[3]/a",
  "jsExtract": ""
}
```

| 字段 | 说明 |
|------|------|
| `url` | 搜索接口 URL，`{{key}}` 占位。也支持 `@js:` 动态构建 |
| `method` | `GET` 或 `POST` |
| `body` | POST 请求体，`{{key}}` 占位 |
| `list` | 搜索结果容器 XPath/CSS/JSONPath |
| `name` | 书名（相对 list 项） |
| `author` | 作者（相对 list 项） |
| `cover` | 封面（相对 list 项） |
| `detailUrl` | 详情链接（相对 list 项） |
| `latestChapter` | 最新章节（相对 list 项，可选） |
| `jsExtract` | JS 提取脚本（可选），返回 JSON 数组 |
| `urlEncrypt` | `aes://password` 格式的加密配置（可选） |

**关键规则：**
- `list` 用绝对路径，`name`/`author` 等用相对路径
- 相对路径用 `.//` 深度匹配，防止子元素嵌套不匹配
- POST 时 `{{key}}` 在 body 中保留原始中文

---

## 五、Detail（详情页规则）

```json
"detail": {
  "cover": "//div[@id='fmimg']/img/@src",
  "name": "//h1",
  "author": "//meta[@property='og:novel:author']/@content",
  "summary": "//div[@id='intro']",
  "chapterList": "//div[@id='list']//dd/a",
  "chapterName": ".",
  "chapterUrl": "./@href",
  "chapterListNextPage": "",
  "jsExtract": ""
}
```

| 字段 | 路径类型 | 说明 |
|------|---------|------|
| `cover` | 绝对 | 封面图 URL |
| `name` | 绝对 | 书名 |
| `author` | 绝对 | 作者 |
| `summary` | 绝对 | 简介 |
| `chapterList` | 绝对 | 章节链接容器 |
| `chapterName` | 相对 | 章节名，常用 `.` |
| `chapterUrl` | 相对 | 章节链接，常用 `./@href` |
| `chapterListNextPage` | 绝对 | 目录翻页链接。也支持 `@json:`/`@js:` 动态构建 |
| `jsExtract` | JS 提取脚本（可选），支持数组格式 |

### `chapterListNextPage` 详解

引擎先在当前页匹配 `chapterList`，匹配到 0 条时加载 `chapterListNextPage` 指向的下一页，递归收集。防重复加载（同一 URL 不会加载两次）。

**详情页 → 全部目录页：**
```json
"chapterListNextPage": "@css:.book_tit a.fr/@href"
```

**API 详情 → API 目录：**
```json
"chapterListNextPage": "@json:'https://api.example.com/chapters?id='+data.id"
```

**JS 分页（layui）：**
```json
"chapterListNextPage": "@js:(function(){var n=document.querySelector('#linkNext');return n&&n.getAttribute('href')||''})()"
```

### jsExtract 数组格式

多行 JS 脚本推荐数组写法，每行一个字符串，引擎自动用换行拼接：

```json
"jsExtract": [
  "(function(){",
  "  var btn=document.querySelector('.catalog-all');",
  "  if(btn)btn.click();",
  "  var r=[];",
  "  document.querySelectorAll('.chapter-list a').forEach(function(a){",
  "    r.push({name:a.textContent.trim(),url:a.getAttribute('href')});",
  "  });",
  "  return JSON.stringify(r);",
  "})()"
]
```

---

## 六、Content（正文规则）

```json
"content": {
  "text": "//div[@id='content']",
  "nextPage": "//a[contains(@class,'chapter-next')]/@href",
  "urlReplaceFrom": "",
  "urlReplaceTo": "",
  "filters": ["//script", "//div[@class='ad']"]
}
```

| 字段 | 说明 |
|------|------|
| `text` | 正文容器 XPath/CSS/JSONPath |
| `nextPage` | 下一页链接（可选），引擎自动拼接多页 |
| `urlReplaceFrom` | 章节 URL 替换-查找（可选） |
| `urlReplaceTo` | 章节 URL 替换-替换为（可选） |
| `filters` | 要删除的元素。支持 XPath 和 `@css:` 前缀 |

### `@json:` 内容自动处理

`@json:` 提取的正文如果含 HTML 标签，引擎自动剥离：
- `<br>` → 换行
- `<p>` 段落之间自动加空行

---

## 七、完整示例

### 示例 1：静态 HTML + XPath

```json
{
  "name": "笔书网",
  "domain": "www.biqukun.org",
  "charset": "utf-8",
  "search": {
    "url": "https://www.biqukun.org/modules/article/search.php?searchkey={{key}}",
    "method": "GET",
    "list": "//table[@class='grid']//tr[td]",
    "name": "td[1]/a",
    "author": "td[2]",
    "detailUrl": "td[1]/a/@href",
    "latestChapter": "td[3]/a"
  },
  "detail": {
    "cover": "//div[@id='fmimg']/img/@src",
    "name": "//div[@id='info']/h1",
    "author": "//div[@id='info']/p[1]",
    "summary": "//div[@id='intro']",
    "chapterList": "//div[@id='list']//dd/a",
    "chapterName": ".",
    "chapterUrl": "./@href"
  },
  "content": {
    "text": "//div[@id='content']",
    "filters": ["//script", "//center"]
  }
}
```

### 示例 2：CSS 选择器 + 过期证书

```json
{
  "name": "勇士小说",
  "domain": "www.337939.com",
  "charset": "utf-8",
  "sslVerify": false,
  "search": {
    "url": "https://www.337939.com/search/",
    "method": "POST",
    "body": "searchkey={{key}}&searchtype=all",
    "list": "//div[contains(@class,'category-commend')]/div",
    "name": ".//a/h3",
    "author": ".//span",
    "detailUrl": ".//a[contains(@href,'/book/')]/@href"
  },
  "detail": {
    "cover": "//div[@class='info-main']//img/@data-original",
    "name": "//h1",
    "author": "//div[@class='info-main']//a[contains(@href,'/author/')]",
    "summary": "//div[@class='info-main-intro']/p[1]",
    "chapterList": "//div[@class='info-chapters flex flex-wrap']/a",
    "chapterName": ".",
    "chapterUrl": "./@href"
  },
  "content": {
    "text": "//article[@id='article']",
    "nextPage": "//a[@id='next_url']/@href",
    "filters": ["//script", "//ins"]
  }
}
```

### 示例 3：WebView + @css: + JS 分节目录

```json
{
  "name": "错层小说",
  "domain": "www.cuoceng.com",
  "charset": "utf-8",
  "useWebView": true,
  "search": {
    "url": "https://m.cuoceng.com/book/so.html?k={{key}}",
    "method": "GET",
    "list": "@css:tbody#bookList tr",
    "name": "@css:td.name a",
    "author": "@css:td.author a",
    "detailUrl": "@css:td.name a/@href",
    "latestChapter": "@css:td.chapter a"
  },
  "detail": {
    "cover": "@css:.book_cover img/@data-src",
    "name": "@css:.book_info h1",
    "author": "@css:.book_info a.author",
    "summary": "@css:.intro_txt p:first-child",
    "chapterList": "@css:.dirWrap ul li a",
    "chapterName": ".",
    "chapterUrl": "./@href",
    "chapterListNextPage": "@js:(function(){var n=document.querySelector('#linkNext');if(n&&n.getAttribute('href'))return n.getAttribute('href');var a=document.querySelector('.book_tit a.fr');return a?a.getAttribute('href'):''})()"
  },
  "content": {
    "text": "@css:#showReading p",
    "nextPage": "@css:.nextPageBox a.next/@href",
    "filters": ["//script", "//ins"]
  }
}
```

### 示例 4：纯 JSON API（无需 WebView 的搜索 + 需要 WebView 的正文）

```json
{
  "name": "幻梦轻小说",
  "domain": "www.huanmengacg.com",
  "charset": "utf-8",
  "useWebView": true,
  "search": {
    "url": "https://www.huanmengacg.com/index.php/bookapi/search?password=huanmengapi&key={{key}}&page=1&size=20",
    "method": "GET",
    "list": "@json:data.list",
    "name": "@json:name",
    "author": "@json:author",
    "cover": "@json:pic",
    "detailUrl": "@json:'https://www.huanmengacg.com/index.php/bookapi/detail?password=huanmengapi&id='+id",
    "latestChapter": "@json:text_num"
  },
  "detail": {
    "cover": "@json:data.pic",
    "name": "@json:data.name",
    "author": "@json:data.author",
    "summary": "@json:data.intro",
    "chapterList": "@json:data.list",
    "chapterName": "@json:name",
    "chapterUrl": "@json:'https://www.huanmengacg.com/index.php/bookapi/content?password=huanmengapi&bid='+bid+'&cid='+id",
    "chapterListNextPage": "@json:'https://www.huanmengacg.com/index.php/bookapi/chapters?password=huanmengapi&id='+data.id+'&size=5000'"
  },
  "content": {
    "text": "@json:data.content",
    "nextPage": "",
    "filters": []
  }
}
```

---

## 八、App 内使用

### 可视化编辑器

书源管理 → 添加/编辑书源 → 基础信息/搜索/详情/内容四个 Tab 填写字段 → 测试搜索验证 → 保存

### JSON 编辑

Toolbar 点击「JSON 模式」直接编辑 → 切回「字段模式」自动回填

### GitHub 订阅

订阅管理 → 添加订阅 → 粘贴 GitHub raw URL → 检查更新 → 预览 → 勾选 → 导入选中

---

## 九、调试技巧

| 现象 | 可能原因 | 解决 |
|------|---------|------|
| 搜索无结果 | `list` XPath 不对 | 浏览器 `$x(...)` / `$$(...)` 验证 |
| 字段全空 | 相对路径错 | 用 `.//` 深度匹配 |
| 无章节 | `chapterList` 错 | 检查章节父容器 |
| 只有几章 | AJAX 加载 | 开 `useWebView` + `jsExtract` |
| 正文为空 | `text` XPath 错 | 试 `//div[@id='content']` |
| 中文乱码 | charset 不对 | 改成 `gbk` |
| 403/连接被拒 | 反爬保护 | 开 `useWebView` |
| SSL 错误 | 证书过期 | 关 `sslVerify` |
| 搜索 API 加密 | 需要 JS 加密 | `@js:` + `AES.encrypt` |
| JSON API 字段取不到 | 路径不对 | 检查 `@json:` 点号路径 |
