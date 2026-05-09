---
name: global-news
description: Aggregate news from multiple sources — financial markets, tech/AI, world events. Uses HN Algolia, Reuters via Google News RSS, dev.to, and Yahoo Finance.
---

# Global News Aggregator

Aggregates news from multiple sources: financial markets, tech/AI, and world events.

## Sources & Reliability

| Source | Type | Status | Notes |
|--------|------|--------|-------|
| Yahoo Finance | 行情数据 | ✅ 稳定 | browser_navigate 或直接 HTTP API |
| Hacker News (Algolia) | 科技热门 | ✅ 稳定 | hn.algolia.com/api/v1 |
| Reuters | 国际新闻 | ✅ 可用 | 通过 Google News RSS: `news.google.com/news/rss/search?q=site:reuters.com+world` |
| dev.to API | 开发者社区 | ✅ 稳定 | dev.to/api/articles |
| BBC RSS | 国际新闻 | ❌ 超时 | WSL 环境下 DNS/RSS 不通 |
| Reddit JSON API | 社区讨论 | ❌ 失败 | WSL 封禁 Reddit |
| MarketWatch RSS | 财经新闻 | ❌ 解析失败 | XML 结构不标准 |

## Quick Commands

### 1. Hacker News 热门科技新闻
```python
import urllib.request, json
from datetime import datetime, timedelta

url = 'https://hn.algolia.com/api/v1/search?tags=front_page&hitsPerPage=10'
with urllib.request.urlopen(url, timeout=10) as r:
    data = json.loads(r.read())

for item in data['hits'][:10]:
    print(f"[{item['points']}pts] {item['title']}")
    print(f"  → {item.get('url', '')[:70]}")
```

### 2. Reuters 国际新闻（通过 Google News RSS）
```python
import urllib.request
from html import unescape
import xml.etree.ElementTree as ET

url = 'https://news.google.com/news/rss/search?q=site:reuters.com+world&hl=en&gl=US&ceid=US:en'
req = urllib.request.Request(url, headers={'User-Agent': 'Mozilla/5.0'})
with urllib.request.urlopen(req, timeout=10) as r:
    content = r.read().decode('utf-8', errors='ignore')

root = ET.fromstring(content)
for item in root.findall('.//item')[:8]:
    title = unescape(item.find('title').text)
    pubdate = item.find('pubDate').text if item.find('pubDate') is not None else ''
    print(f'• {title} ({pubdate})')
```

### 3. dev.to 热门文章
```python
import urllib.request, json

url = 'https://dev.to/api/articles?tag=programming&per_page=10&top=7'
req = urllib.request.Request(url, headers={'User-Agent': 'Mozilla/5.0'})
with urllib.request.urlopen(req, timeout=10) as r:
    articles = json.loads(r.read())

for a in articles:
    print(f"[{a['public_reactions_count']}👍] {a['title']}")
    print(f"  #{a.get('tag_list', [])[:3]} | {a['reading_time_minutes']}min")
```

### 4. Yahoo Finance 市场数据
```python
import urllib.request, json

# Yahoo Finance API
symbols = {'标普500': '%5EGSPC', '纳斯达克': '%5EIXIC', '黄金': 'GC=F', '原油': 'CL=F'}
for name, sym in symbols.items():
    url = f'https://query1.finance.yahoo.com/v8/finance/chart/{sym}?interval=1d&range=2d'
    req = urllib.request.Request(url, headers={'User-Agent': 'Mozilla/5.0'})
    with urllib.request.urlopen(req, timeout=10) as r:
        data = json.loads(r.read())
    result = data['chart']['result'][0]
    current = result['meta']['regularMarketPrice']
    print(f'{name}: {current}')
```

## 汇报格式模板

```
📅 全球要闻汇总 · {日期}

💰 财经市场
| 品种 | 点位 | 涨跌 |
|------|------|------|
| 标普500 | 7,155 | +0.17% 📈 |

🤖 科技/AI 热门（Hacker News）
| 热度 | 标题 |
|------|------|
| 2146pts | GPT-5.5 发布 |

🌍 国际/地缘政治
- 特朗普称不会在伊朗战争中使用核武器
```

## HN Algolia 日期过滤（重要坑点）

`numericFilters` 中的 `>` 必须 URL 编码为 `%3E`，否则请求会返回空结果或报错：

```bash
# ❌ 错误 — shell 或 API 会误解 `>`
curl "https://hn.algolia.com/api/v1/search_by_date?query=AI&tags=story&numericFilters=created_at_i>123456"

# ✅ 正确 — 用 %3E 编码
THREE_DAYS_AGO=$(python3 -c "import time; print(int(time.time() - 3*86400))")
curl -s "https://hn.algolia.com/api/v1/search_by_date?query=AI&tags=story&hitsPerPage=30&numericFilters=created_at_i%3E${THREE_DAYS_AGO}"
```

**搜索策略：** 单一关键词覆盖面有限，应并行多个查询拼接结果：
```bash
# 多关键词并行搜索同一话题
for q in "Iran war" "Iran US attack military" "Iran ceasefire Hormuz"; do
  curl -s "https://hn.algolia.com/api/v1/search_by_date?query=$(echo $q | sed 's/ /+/g')&tags=story&hitsPerPage=20&numericFilters=created_at_i%3E${TS}" 
done
```

## Google News 浏览器抓取（RSS 的增强方案）

当 Google News RSS 结果不够丰富时，直接用 browser 打开 Google News 页面效果更好：

```python
# 步骤
1. browser_navigate("https://news.google.com/search?q=artificial+intelligence+AI&hl=en-US&gl=US&ceid=US:en")
2. browser_console(expression="document.body.innerText.substring(0, 5000)")  # 提取前5000字符
3. browser_scroll(direction="down")  # 继续加载
4. browser_console(expression="document.body.innerText.substring(5000, 10000)")  # 提取更多
```

Google News 页面结构：每条新闻 = `来源 + 标题 + 时间 + 作者`，用 `body.innerText` 即可提取全部文本。

## Notes
- WSL 企业网络封锁 Reddit、BBC RSS、feeds.reuters.com
- Google News RSS 是绕过封锁的好方法
- Yahoo Finance 直接 API 请求比 RSS 更可靠
- HN Algolia API 支持按时间过滤：`numericFilters=created_at_i%3E{timestamp}`（注意 URL 编码）
- HN 的 `search` 端点按相关度排序，`search_by_date` 端点按时间排序
- Reuters 通过 Google News RSS 访问时，中文环境需去掉 `hl=zh-CN&gl=CN` 参数
- 用户要求所有新闻输出必须使用纯中文，英文标题需翻译

## Cron Job 执行注意事项

### 1. Python 环境问题
- Cron 环境中 `pip` / `pip3` 可能不可用（路径问题），不要尝试安装 yfinance 等第三方包
- 直接使用 Python 标准库 `urllib.request` + Yahoo Finance Chart API，无需外部依赖
- 执行 Python 脚本用 `python3 /path/to/script.py`，确保脚本零外部依赖

### 2. 使用独立脚本而非管道
- 所有 HTTP 请求写在 Python 脚本里，用 `urllib.request` 处理，不依赖 shell 管道
- 正确流程：先用 `write_file` 工具创建脚本文件，再用 `terminal()` 执行
- `write_file` 在 `execute_code` sandbox 中不可用，需要作为独立工具调用

### 3. 并行抓取策略
- 市场数据脚本和新闻脚本可以并行执行（两个 `terminal()` 调用同时发起）
- 市场数据脚本：Yahoo Finance API 的 6 个品种逐一请求
- 新闻脚本：HN Algolia + dev.to + Reuters/Google News RSS，三个源合并在一个脚本中，各自 try/except 容错

### 4. 数据验证
- 执行后检查 Yahoo Finance 返回的时间戳，确认数据是最新的交易日
- 如果 Google News RSS 返回空结果或仅有 RSS 频道名，尝试备用查询词（如 `reuters+financial+markets`）

### 5. 翻译与本地化
- 所有新闻标题和内容必须翻译为中文，不允许出现英文
- 如果 LLM 本身翻译能力足够，直接在最终输出时翻译，无需额外调用翻译 API
