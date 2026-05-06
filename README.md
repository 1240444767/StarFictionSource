# 书源规则编写教程

> 适用于 StarBox Fiction 引擎 v3.3 | 2026-05-07

## 一、什么是书源规则

书源规则是一段 JSON，告诉引擎如何从小说网站提取数据。你只需写 XPath 定位元素，引擎会自动完成 HTTP 请求、编码处理和数据提取。

支持**可视化字段编辑**，无需手写 JSON — 但理解 JSON 结构有助于排查问题。

---

## 二、JSON 完整结构

```json
{
  "name": "书源名称",
  "domain": "网站域名",
  "charset": "utf-8",
  "userAgent": "",
  "useWebView": false,
  "search": { ... },
  "detail": { ... },
  "content": { ... }
}
```

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `name` | string | 是 | 自定义书源名称 |
| `domain` | string | 是 | 网站域名，如 `biqukun.org` |
| `charset` | string | 否 | 网页编码：`utf-8`（默认）、`gbk`、或自定义字符集 |
| `userAgent` | string | 否 | 自定义 UA，留空使用 OkHttp 默认 |
| `useWebView` | boolean | 否 | 默认 `false`。设为 `true` 时该源所有请求走 WebView（详见第十一章） |
| `search` | object | 是 | 搜索规则 |
| `detail` | object | 是 | 详情页规则 |
| `content` | object | 是 | 正文规则 |

---

## 三、核心概念：XPath

引擎使用 Jsoup 解析 HTML，所有规则字段均为 XPath 表达式。**不支持 CSS 选择器。**

### 浏览器获取 XPath

1. F12 打开开发者工具
2. 左上角"选择元素"箭头 → 点击目标元素
3. Elements 面板右键高亮 HTML → Copy → Copy XPath
4. 浏览器复制的是绝对路径（如 `/html/body/div[3]/div[1]`），建议改为相对路径以提高稳定性

### 常用 XPath 语法

| 表达式 | 含义 | 示例 |
|--------|------|------|
| `//div` | 所有 div | |
| `//div[@class='list']` | class 为 "list" 的 div | |
| `//div[@id='content']` | id 为 "content" 的 div | |
| `//a[@href]` | 含 href 的 a 标签 | |
| `//tr[td]` | 包含 td 子元素的 tr | |
| `//a[contains(@href,'book')]` | href 含 "book" 的 a | |
| `//a[1]` | 第 1 个 a（序号从 1 开始） | |
| `//meta[@property='og:title']/@content` | 提取 meta 标签的 content 属性值 | |
| `.//td[1]/a` | 当前节点下第 1 个 td 中的 a | **相对路径，见搜索规则** |
| `./@href` | 当前节点的 href 属性值 | 相对路径，见章节规则 |
| `./text()` | 当前节点的文本 | 相对路径，见章节名 |

### 引擎支持的 XPath 后缀

- `/@attrName` — 提取属性值，如 `img/@src`、`a/@href`
- `/text()` — 提取文本（引擎自动剥离此后缀，等价于直接取该元素文本）
- 对 `<input>` / `<textarea>` 元素自动取 `value` 属性

### 相对路径 vs 绝对路径

搜索规则的 `name`、`author` 等字段在 `list` 匹配到的每个节点上执行。如果子元素不是直接子节点（中间隔了 `<div>` 等），需要用 `.//` 深度匹配：

```
❌ 错：td[1]/a          — 只找直接子 td 下的 a
✅ 对：.//td[1]/a       — 找所有后代 td 下的 a
✅ 对：.//h3/a          — 找所有后代 h3 下的 a
```

### XPath vs CSS 对照

| CSS | XPath |
|-----|-------|
| `.class` | `//*[@class='class']` |
| `#id` | `//*[@id='id']` |
| `div.class` | `//div[@class='class']` |
| `div > a` | `//div/a` |
| `div a` | `//div//a` |
| `a[href]` | `//a[@href]` |
| `a[href*="book"]` | `//a[contains(@href,'book')]` |
| `tr:nth-child(2)` | `//tr[2]` |
| `p:has(a)` | `//p[a]` |

---

## 四、Search（搜索规则）

用户搜索关键词 → 引擎替换 `{{key}}` → 请求页面 → 解析结果列表。

```json
"search": {
  "url": "https://www.biqukun.org/modules/article/search.php?searchkey={{key}}",
  "method": "GET",
  "body": "",
  "list": "//table[@class='grid']//tr[td]",
  "name": "td[1]/a",
  "author": "td[2]",
  "cover": "td[1]/a/img/@src",
  "detailUrl": "td[1]/a/@href",
  "latestChapter": "td[3]/a"
}
```

### 字段详解

| 字段 | 说明 | 写法 |
|------|------|------|
| `url` | 搜索接口 URL，`{{key}}` 占位搜索词 | 在网站搜索框输入任意词 → 看地址栏 URL → 搜索词换成 `{{key}}` |
| `method` | `GET` 或 `POST` | 默认 GET。如果网站搜索是 POST（打开 F12 Network 面板看）填 POST |
| `body` | POST 请求体（GET 时留空） | 同样用 `{{key}}` 占位，如 `searchkey={{key}}&page=1` |
| `list` | 每个搜索结果项的容器 XPath（**绝对路径**） | 定位到重复元素的共同父节点 |
| `name` | 书名（**相对 list 项**） | `.//td[1]/a` 或 `.//h3/a` |
| `author` | 作者（**相对 list 项**） | `.//td[2]` 或 `.//span[@class='author']/a` |
| `cover` | 封面（**相对 list 项**） | `.//img/@src` |
| `detailUrl` | 详情页链接（**相对 list 项**） | `.//h3/a/@href` |
| `latestChapter` | 最新章节名（**相对 list 项**，可选） | `.//td[3]/a` |
| `jsExtract` | JS 提取脚本（可选，见第十一章） | 仅 `useWebView` 时可用 |

### 关键规则

- `list` 用**绝对 XPath**（以 `//` 开头），在全页面匹配
- `name`、`author` 等用**相对 XPath**，以 `list` 匹配到的每个节点为上下文
- **相对路径用 `.//` 深度匹配**（而非直接 `td[1]/a`），防止子元素嵌套层次不匹配
- POST 请求时 `{{key}}` 在 body 中保留原始中文，引擎自动处理编码

---

## 五、Detail（详情页规则）

用户点击搜索结果进入详情页，引擎提取书籍信息和所有章节链接。

```json
"detail": {
  "cover": "//div[@id='fmimg']/img/@src",
  "name": "//div[@id='info']/h1",
  "author": "//meta[@property='og:novel:author']/@content",
  "summary": "//div[@id='intro']",
  "chapterList": "//div[@id='list']//dd/a",
  "chapterName": ".",
  "chapterUrl": "./@href",
  "chapterListNextPage": ""
}
```

### 字段详解

| 字段 | 路径类型 | 说明 |
|------|---------|------|
| `cover` | **绝对** | 封面图 URL，通常取 `img/@src` |
| `name` | **绝对** | 书名 |
| `author` | **绝对** | 作者 |
| `summary` | **绝对** | 简介/描述 |
| `chapterList` | **绝对** | 章节链接的容器，匹配所有章节项 |
| `chapterName` | **相对 chapterList** | 每章名称，常用 `.` 或 `./text()` |
| `chapterUrl` | **相对 chapterList** | 每章链接，常用 `./@href` |
| `chapterListNextPage` | **绝对**（可选） | 章节列表翻页链接，留空表示无翻页 |
| `jsExtract` | JS 提取脚本（可选，见第十一章） | 用于 AJAX 加载的章节目录 |

### og: 标签取巧法

很多小说站 head 中有 Open Graph 标签，比可见文本更稳定：

```html
<meta property="og:novel:book_name" content="剑来" />
<meta property="og:novel:author" content="烽火戏诸侯" />
<meta property="og:image" content="https://.../cover.jpg" />
```

```json
"name": "//meta[@property='og:novel:book_name']/@content",
"author": "//meta[@property='og:novel:author']/@content",
"cover": "//meta[@property='og:image']/@content"
```

---

## 六、Content（正文规则）

```json
"content": {
  "text": "//div[@id='content']",
  "nextPage": "//a[contains(text(),'下一页')]/@href",
  "urlReplaceFrom": "articles",
  "urlReplaceTo": "articlescontent",
  "filters": ["//script", "//div[@class='ad']", "//center"]
}
```

### 字段详解

| 字段 | 说明 |
|------|------|
| `text` | 正文容器 XPath（**绝对**）。通常是一个 `<div>`，也用 `//article//p` 匹配段落 |
| `nextPage` | 下一页链接 XPath（**绝对**，可选）。有些站一章分多页，引擎自动拼接 |
| `urlReplaceFrom` | 章节 URL 替换-查找字符串（可选） |
| `urlReplaceTo` | 章节 URL 替换-替换为字符串（可选） |
| `filters` | 要删除的元素 XPath 数组。在提取正文**之前**过滤广告/脚本/导航 |
| `jsExtract` | JS 提取脚本（可选，见第十一章） |

### urlReplace 用途

有些网站章节列表中的链接和实际阅读链接不同，需要替换一部分路径：

```
列表链接: /articles/123/1.html
实际阅读: /articlescontent/123/1.html
→ urlReplaceFrom: "articles"
→ urlReplaceTo: "articlescontent"
```

### filters 推荐

```json
"filters": [
  "//script",         // JS 脚本
  "//ins",            // 广告插件插入的内容
  "//div[contains(@id,'ad')]",    // id 含 ad 的 div
  "//div[contains(@class,'ad')]", // class 含 ad 的 div
  "//center",         // 居中的版权声明
  "//div[@class='footer']"
]
```

---

## 七、完整示例

### 示例 1：biqukun.org（GET 搜索，静态 HTML）

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
    "cover": "",
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

### 示例 2：wap.po18.work（POST 搜索）

```json
{
  "name": "PO18 手机版",
  "domain": "wap.po18.work",
  "charset": "utf-8",
  "search": {
    "url": "https://wap.po18.work/s.php",
    "method": "POST",
    "body": "searchkey={{key}}&page=1",
    "list": "/html/body/div[2]/ul/li",
    "name": "h2/a",
    "author": "span[@class='author']",
    "cover": "a[@class='top_img']/img/@src",
    "detailUrl": "h2/a/@href"
  },
  "detail": {
    "cover": "//div[@class='fm']/img/@src",
    "name": "//h1",
    "author": "//meta[@property='og:novel:author']/@content",
    "summary": "//div[@class='intro']",
    "chapterList": "//div[@class='chapters'][p[contains(text(),'全部章节')]]/ul/li[a]",
    "chapterName": "./a",
    "chapterUrl": "./a/@href",
    "chapterListNextPage": "//div[@class='listpage']//a[contains(text(),'下一页')]/@href"
  },
  "content": {
    "text": "//div[@id='content']",
    "filters": ["//script", "//div[@class='chapterpage']", "//div[@class='recommend']"]
  }
}
```

### 示例 3：ixdzs8.com（WebView + AJAX 章节 + jsExtract）

```json
{
  "name": "爱下电子书",
  "domain": "ixdzs8.com",
  "charset": "utf-8",
  "useWebView": true,
  "search": {
    "url": "https://ixdzs8.com/bsearch?q={{key}}",
    "method": "GET",
    "list": "//ul[@class='u-list']/li",
    "name": ".//h3[@class='bname']/a",
    "author": ".//span[@class='bauthor']/a",
    "cover": ".//div[@class='l-img']/a/img/@src",
    "detailUrl": ".//h3[@class='bname']/a/@href",
    "latestChapter": ".//p[@class='l-last']/a/span[@class='l-chapter']"
  },
  "detail": {
    "cover": "//div[@class='n-img']/img/@src",
    "name": "//h1",
    "author": "//a[@class='bauthor']",
    "summary": "//p[@id='intro']",
    "chapterList": "//ul[contains(@class,'u-chapter')]/li/a",
    "chapterName": ".",
    "chapterUrl": "./@href",
    "jsExtract": [
      "(function(){",
      "  var b=document.querySelector('.catalog-all');",
      "  if(b)b.click();",
      "  var c=document.querySelector('#a-list .u-chapter');",
      "  if(!c||c.querySelectorAll('li a').length===0){",
      "    c=document.querySelector('.cfirst');",
      "  }",
      "  var r=[];",
      "  if(c){",
      "    c.querySelectorAll('li a').forEach(function(a){",
      "      r.push({name:a.textContent.trim(),url:a.getAttribute('href')});",
      "    });",
      "  }",
      "  return JSON.stringify(r);",
      "})()"
    ]
  },
  "content": {
    "text": "//article[@class='page-content']//p",
    "nextPage": "//a[contains(@class,'chapter-next')]/@href",
    "filters": ["//script", "//ins", "//div[contains(@id,'bg-ssp')]"]
  }
}
````

---

## 八、App 内使用流程

### 方式一：可视化编辑器（推荐）

1. 书源管理 → 添加书源 → 选择导入方式
2. 也可直接点击已有书源进入编辑器
3. 在「基础信息 / 搜索 / 详情 / 内容」四个 Tab 中填写字段
4. 每个字段都有提示文字，告诉你怎么填
5. 点击「测试搜索」验证规则
6. 点击「保存」

### 方式二：JSON 编辑

1. 编辑器 Toolbar 点击「JSON 模式」切换到源码编辑
2. 粘贴或手写 JSON
3. 点击「字段模式」切回可视化，字段会自动回填
4. 来回切换不会丢失数据

### 方式三：GitHub 订阅

1. 书源管理 → 添加书源
2. 也可通过「订阅管理」从远程批量导入/更新
3. 支持 `github.com/.../blob/...` URL，自动转换为 raw URL
4. 订阅管理可「检查更新」，系统逐条对比远程规则与本地已导入的规则：
   - 新增的书源 → 标记为新
   - 已导入但 JSON 有修改 → 标记为「可更新」
   - 已导入且未变 → 灰色不可选
5. 勾选需要的规则 → 点击「导入选中」即可

---

## 九、订阅管理详解

### 添加订阅

1. 订阅管理 → 添加订阅
2. 粘贴 GitHub raw URL（支持 `github.com/.../blob/...` 自动转换）
3. 也支持镜像 URL，如 `https://ghfast.top/https://raw.githubusercontent.com/...`

### 检查更新

点击「检查更新」→ 系统逐条对比远程规则和本地已导入的规则：

| 状态 | 显示 | 行为 |
|------|------|------|
| 新书源 | 正常选中 | 导入 = 新增 |
| 已导入有修改 | 标为「可更新」 | 导入 = 覆盖更新 |
| 已导入未变 | 标为「已导入」灰色 | 不可选 |

**注意：** 检查更新后必须点击导入才算完成，只看不导下次还会提示有更新。

---

## 十、调试技巧

### 1. 浏览器 Console 验证 XPath

```javascript
$x("//table[@class='grid']//tr[td]")
```

返回匹配元素列表，点击可高亮。

### 2. 浏览器 Network 面板

- 搜索是 GET 还是 POST？
- 搜索参数名（`searchkey`、`keyword`、`kw`、`q` 等）
- 响应内容是什么格式？

### 3. App 内「测试搜索」按钮

编辑器底部 → 输入关键词 → 搜索 → 弹窗显示匹配到的书名和作者。

如果无结果，检查 list XPath；如果字段全空，检查相对路径序号或用 `.//` 深度匹配。

### 4. 常见问题

| 现象 | 可能原因 | 检查 |
|------|---------|------|
| 搜索无结果 | `list` XPath 不对 | 浏览器 `$x(...)` 验证 |
| 有结果但全是空 | 相对 XPath 路径错 | 用 `.//` 代替直接子元素路径 |
| 点进详情没章节 | `chapterList` 错 | 检查章节链接的父容器 |
| 正文为空 | `text` XPath 错 | 试 `//div[@id='content']` |
| 中文乱码 | charset 不对 | 改成 `gbk`，或浏览器看页面 `<meta charset>` |
| POST 搜索失败 | `body` 格式错 | 对比浏览器 Network 面板的 Form Data |
| 章节只有几章 | AJAX 加载/分页 | 开启 `useWebView` + 写 `jsExtract` |
| 页面打不开/403 | 反爬保护 | 开启 `useWebView` |

---

## 十一、WebView 模式与 jsExtract

### 什么时候需要 WebView？

大多数小说站是静态 HTML，用 OkHttp 直接请求即可。但以下情况必须开启 `useWebView: true`：

- **反爬保护**：CloudFlare 验证、浏览器检查（如 ixdzs8.com）
- **JS 动态渲染**：Vue/React 渲染内容、AJAX 加载数据
- 页面返回 403/503 但浏览器能正常打开

开启后引擎会用内嵌 WebView 加载页面，等 2.5 秒让 JS 执行完毕，再提取 HTML。

**注意：** WebView 比 OkHttp 慢很多，普通网站不要开。

### jsExtract：JS 提取脚本

当 XPath 无法提取到数据时（如 AJAX 动态加载的章节目录），可以用 `jsExtract` 写一段 JS 代码，在 WebView 中执行，返回 JSON 数据。

#### 字符串写法（简单脚本）

```json
"jsExtract": "(function(){return JSON.stringify([{name:'第一章',url:'/read/1/p1.html'}]);})()"
```

#### 数组写法（多行脚本，推荐）

每一行一个字符串，引擎自动用换行符拼接：

```json
"jsExtract": [
  "(function(){",
  "  var list=[];",
  "  document.querySelectorAll('.chapter-list a').forEach(function(a){",
  "    list.push({name:a.textContent,url:a.getAttribute('href')});",
  "  });",
  "  return JSON.stringify(list);",
  "})()"
]
```

#### 返回值格式

- **search.jsExtract** / **detail.jsExtract**：返回 JSON 数组 `[{name, author, url, ...}]`
- **content.jsExtract**：返回 JSON 对象 `{text: "正文内容"}` 或纯字符串

#### 实战：AJAX 加载的章节目录

像 ixdzs8 这种网站，详情页只显示 8 章，完整目录通过 AJAX 加载。jsExtract 点击"查看全部"按钮后提取：

```json
"jsExtract": [
  "(function(){",
  "  var btn=document.querySelector('.catalog-all');",
  "  if(btn)btn.click();",
  "  var list=document.querySelector('#a-list .u-chapter');",
  "  if(!list||list.querySelectorAll('li a').length===0){",
  "    list=document.querySelector('.cfirst');",
  "  }",
  "  var r=[];",
  "  list.querySelectorAll('li a').forEach(function(a){",
  "    r.push({name:a.textContent.trim(),url:a.getAttribute('href')});",
  "  });",
  "  return JSON.stringify(r);",
  "})()"
]
```

### WebView 模式下的下载

开启 `useWebView` 的书源，下载章节时也会走 WebView 加载，确保能正常获取正文。下载支持断点续传 — 中断后再次下载会从上次位置继续。

---

## 十二、从零适配一个网站

1. 打开目标网站 → 搜索「剑来」→ 观察 URL → 写 `search.url` 和 `search.method`
2. 搜索结果页 → 找到列表容器 → 写 `search.list`
3. 列表项中找书名、作者、封面、链接 → 写 `search.name/author/cover/detailUrl`（**用 `.//` 深度匹配**）
4. 点进详情页 → 找封面/书名/作者/简介 → 写 `detail.cover/name/author/summary`
5. 找章节列表 → 写 `detail.chapterList`
6. 点进第一章 → 找正文容器 → 写 `content.text`
7. 去掉广告 → 写 `content.filters`
8. 在 App 中用「测试搜索」验证整条链路
9. 如果网站有反爬或 AJAX → 设置 `"useWebView": true`，必要时写 `jsExtract`
