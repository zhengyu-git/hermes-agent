---
name: chinese-news-scraping
description: Fetch and compile Chinese domestic and international news headlines from websites that render content via JavaScript
triggers:
  - daily news push
  - news aggregation
  - Chinese news scraping
---

# Chinese News Scraping Skill

Fetch headlines from Chinese news websites using Python requests.

## Key Discovery (2026-05-12)
Most major Chinese news sites use JavaScript rendering and do NOT expose headline links in static HTML. This includes:
- `people.com.cn` (人民网) - returns only nav/footer links
- `xinhuanet.com` (新华网) - heavily JavaScript-rendered
- `thepaper.cn` (澎湃) - JS-rendered, returns only ICP info
- `cctv.com` (央视) - most sections return empty
- `sina.com.cn` - homepage returns nav elements, international section works
- `ifeng.com` (凤凰网) - returns mostly nav/metadata

## Working Sources

### Reliable static HTML sources:
- **证券时报** `https://www.stcn.com/` - returns actual news titles
- **财新** `https://www.caixin.com/` - financial news works well
- **第一财经** `https://www.yicai.com/` - business news
- **BBC中文** `https://www.bbc.com/zhongwen/simp/` - international news
- **观察者网** `https://www.guancha.cn/` - domestic/international sections
- **百度新闻** `https://news.baidu.com/` - aggregation works

### Partial success:
- **新浪国际** `https://news.sina.com.cn/world/` - international works
- **央视国际** `https://news.cctv.com/world/` - sometimes has real titles
- **环球时报** `https://www.huanqiu.com/` - some domestic headlines

## Code Template

```python
import requests, re

headers = {'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36'}

def fetch_titles(url):
    try:
        r = requests.get(url, headers=headers, timeout=15)
        r.encoding = 'utf-8'
        text = re.sub(r'<script[^>]*>.*?</script>', '', r.text, flags=re.DOTALL)
        text = re.sub(r'<style[^>]*>.*?</style>', '', text, flags=re.DOTALL)
        titles = re.findall(r'<a[^>]+href=[^>]+>([^<]{10,80})</a>', text)
        nav = {'首页','新闻','财经','体育','娱乐','军事','科技','社会','视频','图片','更多','相关','推荐'}
        titles = [t.strip() for t in titles if t.strip() not in nav and 10 < len(t.strip()) < 80]
        return titles[:30]
    except:
        return []
```

## Deduplication Strategy
When fetching news for multiple pushes per day:
1. Read previous `.md` files from `/home/v-zhengyu002/.hermes/cron/output/d7786a07b9cf/`
2. **Extract the `## Response` section only** — the rest of the `.md` file contains cron metadata and prompt text. Use:
   ```python
   import re
   m = re.search(r'## Response\s*\n+(.*)', full_text, re.DOTALL)
   prev_news = m.group(1) if m else ""
   ```
   Do NOT compare against the entire file contents, or keyword lists will produce false positives (e.g., "马化腾" appearing in cron prompt → false duplicate alert).
3. Extract key terms (names, places, events) from prior pushes
4. Filter out any fetched news containing those key terms — but keep terms that appear in the prompt/metadata but NOT in the actual `## Response` news content
5. Aim for <30% overlap with prior content

## Critical: Summit Coverage Dominance (2026-05-13 observation)

When a major 元首外交 summit is ongoing (e.g., 中美最高层会晤), the **homepages of 人民网/新华网/中新社 are flooded with summit coverage**. Running `fetch_titles()` on these portals during summit periods returns 10-15 identical 元首外交/中美关系 headlines — nearly zero news diversity for hours.

**Workaround**: After fetching homepage titles, if >60% are the same story/family of stories, switch to **section-specific or alternative sources**:
- 网易国际 `news.163.com/world/` — returns genuinely diverse international stories
- 凤凰网 `www.ifeng.com/` — social, international, breaking news
- 新浪财经 `finance.sina.com.cn/` — domestic/international financial news
- 央视国际 `news.cctv.com/world/` — international coverage
- 人民网国际 `world.people.com.cn/` — international from People.com

**Verified empty/minimally-useful sources** (avoid during summit periods):
- 澎湃新闻 `thepaper.cn` — JS-rendered, near-empty via requests
- 今日头条 `toutiao.com` — JS-rendered, near-empty via requests
- 观察者网 `guancha.cn` — returns highly repetitive content
- 凤凰国际 `ifeng.com/c/...` — often returns empty

## Verification Checklist
- At least 10 domestic news items (10+ chars each)
- At least 10 international news items (10+ chars each)
- Emoji tags present: 🇨🇳 / 🌍
- Title time-of-day indicator matches cron run time (午间/晚间/夜间)
