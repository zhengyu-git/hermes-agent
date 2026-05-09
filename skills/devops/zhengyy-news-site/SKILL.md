---
name: zhengyy-news-site
description: 新闻网站 myapp.zhengyy.com/news 运维 — cron 任务、解析逻辑、已知问题
---

# 新闻网站运维 (myapp.zhengyy.com/news)

## 背景
新闻网站从 cron 任务读取 JSON 文件渲染。cron 每天生成两条 QQ 消息（国内+国际），用 `---` 分隔。

## 已知问题：国际新闻丢失

**根因**：cron Prompt 原来用 `---` 分隔两条消息，系统只把第一条存文件，第二条直接推 QQ 丢了。文件里只有国内段落。

**修复**：
1. **cron Prompt** 改为单条消息包含国内+国际内容（去掉 `---` 分隔符）
2. **app.py 解析逻辑** line 211：`---` 后的 `📰` 标题现在能正确设置 `current_section = 'global'`，后续 `🌍 国际` 段落不再丢失

**临时补救**：用历史完整数据（12:04那次）手动补充到最新文件。

## 文件路径
- `/home/v-zhengyu002/myapp/app.py`
- `/home/v-zhengyu002/myapp/cron/daily_news.cron`
- `/home/v-zhengyu002/myapp/templates/news.html`
- JSON 数据：`/home/v-zhengyu002/myapp/news_data/`

## 验证
- `https://myapp.zhengyy.com/news` 返回 200
- JSON 文件里有 `global` 段落且有内容
