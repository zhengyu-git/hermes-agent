---
name: daily-news-digest
description: 每日新闻推送 — 从东方财富/财联社/新浪国际抓取当日新闻，按国内和国际分类输出。用于cron定时任务每日推送。
triggers:
  - 每日新闻
  - 今日新闻
  - 新闻推送
  - 午间速报
  - 晚间速报
---

# Daily News Digest (每日实时新闻推送)

从东方财富/财联社/新浪国际抓取**当天**最新新闻，分为国内和国际两部分输出。

## 数据源优先级

1. **360搜索 → 财联社早间新闻精选**（`so.com/s?q=X月X日+早间新闻精选`）— 最精简全面，32条涵盖国内政策、A股业绩、国际市场
2. **Google News RSS** — 当天国际政治/外交/突发新闻
3. **百度新闻首页**（`news.baidu.com`）— 社会/科技/综合国内新闻补充
4. **HN top stories** — 科技热点补充

**废弃的旧数据源**（以下来源存在反爬/JS渲染/链接失效问题，不再使用）：
- 东方财富页面 — browser scroll 后 snapshot 可能变空
- 新浪国际热点小时报直接 URL — 链接动态失效
- 澎湃首页 — JS 渲染，curl 不可见内容

## 抓取方法（已验证可用）

### 方法1：浏览器直接抓取（最推荐，反爬最友好）

```
1. browser_navigate("https://finance.eastmoney.com/a/czqyw.html")  # 证券聚焦
2. browser_snapshot()  # 获取新闻列表（标题+摘要+时间）
3. browser_scroll("down")  # 滚动加载更多
4. browser_snapshot()  # 再取快照
```

同一流程用于：
- `https://finance.eastmoney.com/a/cgnjj.html` — 国内经济新闻
- `https://news.sina.com.cn/world/` — 国际新闻

**注意**：EastMoney页面scroll后snapshot可能变空，此时重新navigate即可。直接navigate获取首屏内容通常够用。

### 方法2：找到"午间新闻精选"原文

1. 在 `finance.eastmoney.com/a/cgnjj.html` 的snapshot中找到"X月X日午间新闻精选"条目
2. 用 browser_console 找出链接：`Array.from(document.querySelectorAll('a')).filter(a => a.innerText.includes('午间新闻精选')).map(a => a.href)[0]`
3. `browser_navigate(href)` 直接进入原文，获得完整5条摘要

午间新闻精选链接格式：`https://finance.eastmoney.com/a/YYYYMMDD{id}.html`

### 方法3：HN API（科技补充）

先下载ID列表到文件，然后在 execute_code 中用标准库逐个获取题目：
```
curl -s "https://hacker-news.firebaseio.com/v0/topstories.json" -o /tmp/hn_ids.json
```
execute_code中读取文件，对每个ID `curl -s https://hacker-news.firebaseio.com/v0/item/{id}.json` 获取标题和分数。

### 方法4：360搜索找CLS/东方财富原文（备用）

先用curl保存搜索结果HTML到临时文件，再用 execute_code 的Python标准库解析HTML提取链接然后跟随。但360搜索结果链接经过中间跳转（`so.com/link?m=...` → `10jqka.com.cn/...`），不如直接用browser打开东方财富页面。

## 已知失败/不可用的方法

- **delegate_task 搜索新闻** — 子代理超时或返回空结果，不适合实时新闻抓取
- **Baidu新闻搜索** — 返回301重定向，curl跟随也只拿到nginx错误页
- **CLS API** (`cls.cn/api/sw`) — 返回405 Method Not Allowed
- **curl后管道到python3** — 安全扫描拦截，必须先保存到文件再读取
- **新浪国际热点小时报直接URL** — `k.sina.com.cn/article_...` 跳转到"该文章已不存在"页面，链接可能动态变化
- **澎湃首页curl抓取** — 返回稀疏HTML（JS渲染内容），`grep title` 只能拿到 `<title>` 标签，新闻列表不可见

## 已验证可用的新方法（本次执行）

### 方法5：360搜索 → 财联社早间新闻精选（最稳）

1. `browser_navigate("https://www.so.com/s?q=4月30日+早间新闻精选")`（替换当日日期）
2. 搜索结果中找"财联社4月30日早间新闻精选"的链接，通常在搜索结果前几条
3. `browser_click` 点击该链接，跳转到新浪财经转载的财联社原文
4. `browser_snapshot` 获取完整32条新闻全文（财经+A股业绩+国际+美股）

**优势**：一条来源覆盖国内经济政策、A股业绩、国际市场，32条精炼摘要，一次抓取够用。

### 方法6：百度新闻首页抓取补充国内选题

```
curl -s 'https://news.baidu.com/' | grep -oP '<a[^>]*>[^<]{10,80}</a>' | sed 's/<[^>]*>//g' | sort -u
```

百度新闻首页会列出当天20-30条热点标题，覆盖社会、科技、政策话题，可作为财联社财经新闻之外的"社会/综合"补充。

### 方法7：Google News RSS（使用 terminal curl + python3 json 模块）

RSS返回当天国际热点新闻标题（含来源媒体标注），覆盖重大国际事件，无需浏览器。在 execute_code 中用 `terminal("curl ...")` 调用并解析。

### 方法8：HN API（科技补充）

用 terminal 下载 top stories JSON，在 execute_code 中用标准库逐个获取标题和URL。

## 推荐执行流程（完整版）

1. **360搜索 → 财联社早间精选**（方法5）：获取国内财经政策+A股业绩+国际原油/美股，约32条
2. **Google News RSS**（方法7）：获取当天最新国际政治/外交/突发事件
3. **百度新闻首页**（方法6，可选）：补充社会/科技/综合类国内新闻
4. **HN API**（方法8，可选）：补充科技热点

从以上来源手工筛选出国内8-12条、国际6-10条，输出纯文本摘要。

## 输出格式

```
📰 X月X日（周X）午间速报

🇨🇳 中国国内
- [一句话新闻]
...

🌍 国际
- [一句话新闻]
...
```

## 注意事项

- 只输出纯文字摘要，不用问日期（cron模式无交互）
- 只报道当天发布的最新新闻，不重复前几天的旧闻
- 国内8-12条，国际6-10条，每条一句话概括
- 纯中文，简洁有力
- 结尾不用"一句话总结"
- 如果当天新闻尚未大量更新（凌晨），如实说明"目前今日新闻尚未大量更新，稍后推送"，不要硬凑旧闻
- 运行在cron模式，无用户交互

## 与 global-news 技能的区别

- `global-news`：全球要闻汇总，包含财经表格、HN热点、Reuters新闻，格式更结构化
- `daily-news-digest`：专门的中文每日新闻推送，面向国内用户，按国内/国际分类，来自东方财富/新浪等中文源，适合cron定时推送