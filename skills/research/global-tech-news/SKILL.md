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
| 360搜索 | `https://www.so.com/s?q=<query>` (curl + 浏览器) | **中文AI/科技资讯，国产模型覆盖** |

### ⚠️ 重要：HN Algolia 的局限性

HN Algolia API **严重偏向英文社区**，会遗漏：
- 国产大模型发布（Qwen、MiMo、GLM、DeepSeek 中文报道）
- 国内科技新闻（小米、阿里、智谱等）
- 中文技术社区的深度分析

**规则：涉及 AI/大模型/国产科技 的话题，必须同时搜索 360搜索 中文源。**
搜索方法：**curl 会被反爬返回空内容，必须用浏览器 navigate 打开360搜索**。命令：`browser_navigate(url='https://www.so.com/s?q=<关键词>')`，然后用 browser_snapshot 读取结果。

## Known Network Limitations (WSL)

- ✅ 可访问：HN Algolia API、dev.to API、360搜索(so.com)
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
