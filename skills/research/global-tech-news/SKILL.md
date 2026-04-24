---
name: global-tech-news
description: 获取全球科技热点要闻（过去一周），以步骤式单独消息发送
triggers:
  - 全球要闻
  - 科技热点
  - 全球科技新闻
  - 最近一周科技热点
  - tech news
---

# Global Tech News Summary

Get weekly global tech news highlights in Chinese, delivered as individual step-by-step messages.

## Data Sources (WSL Network Validated 2026-04)

| Source | API/URL | Coverage |
|--------|---------|----------|
| Hacker News | `hn.algolia.com/api/v1/search?tags=front_page&hitsPerPage=30` | 全球开发者热点，投票制 |
| dev.to | `dev.to/api/articles?tag=ai&per_page=20&top=1` | AI/科技圈文章 |
| dev.to | `dev.to/api/articles?tag=technology&per_page=20&top=1` | 技术文章 |

## Known Network Limitations (WSL)

- ✅ 可访问：HN Algolia API、dev.to API
- ❌ 不可访问：Google Search、Bing Search、Gitee
- 📌 Gitee push在WSL里无法完成，需Windows或换网络环境

## Output Format

每条步骤单独发一条消息，最后汇总发一条完整要闻。

步骤格式：
```
📌 步骤N
[操作内容]
→ [具体命令/请求]
→ [结果] ✓/❌
```

汇总格式：
```
✅ [主题]汇总完毕（日期范围）

🔴 大事件
- ...

🤖 AI & 科技
- ...

🔐 安全 & 隐私
- ...

💻 开发者热点
- ...
```

## Filtering Rules
- 日期范围：默认过去7天
- HN热度过滤：≥50票
- dev.to过滤：reactions≥1 或重要事件
- 去重后按热度排序输出

## Notes
- 每步单独发消息，不要合并
- 最后发完整汇总
- 中文输出，简洁有力
- Windows操作必须先问用户
