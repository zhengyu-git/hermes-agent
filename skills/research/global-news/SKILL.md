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

## Notes
- WSL 企业网络封锁 Reddit、BBC RSS、feeds.reuters.com
- Google News RSS 是绕过封锁的好方法
- Yahoo Finance 直接 API 请求比 RSS 更可靠
- HN Algolia API 支持按时间过滤：`dateRange=all` 或 `numericFilters=points>300`
- Reuters 通过 Google News RSS 访问时，中文环境需去掉 `hl=zh-CN&gl=CN` 参数
