---
name: news-push-cron-debug
description: 每日新闻推送 cron 任务的调试技巧 — 文件结构、去重逻辑、采集技巧
category: devops
---

# 每日新闻推送 Cron 调试技巧

## 文件路径结构
Cron 输出目录：`/home/v-zhengyu002/.hermes/cron/output/d7786a07b9cf/`

**重要**：有两类文件：
- `2026-05-11_12-05-58.md` 等 — 包含 task prompt + metadata，**不是**新闻内容
- `news_20260511_noon.md` — **真正的新闻输出文件**，包含 📰 标题和具体新闻条目

读取历史推送时，**必须读 `news_YYYYMMDD_noon.md` 而不是 cron run 的 `.md` 文件**。

## 去重逻辑
- 午间推送：选 8-12 点新闻
- 晚间推送：排除午间，12-17 点新闻
- 夜间推送：排除午间+晚间，17 点后新闻

判断重叠时：
- 精确关键词匹配（如"3家中国公司"是重复）
- 但同一事件的不同阶段报道不算重复（如"汉坦病毒"午间报了邮轮，晚间报WHO 7例 — 这是新进展，关键词相同但内容不同）

## 新闻采集技巧
```python
# 很多新闻站用JS渲染，直接requests.get拿不到标题
# 人民/新华/中新/凤凰/环球 较好采集
# 澎湃/腾讯/网易 常返回空，需要多试几个URL
# 央视 r.encoding='utf-8' 配合正则 <a[^>]+href=[^>]+>([^<]{10,80})</a> 效果较好
```

## 已知问题

### execute_code 在 cron 环境里流式调用网络请求会静默卡死

**现象**：`last_status: ok`（cron 系统标），但 `~/.hermes/cron/output/<job_id>/` 目录下没有新的输出文件，或文件是旧的。手动 `cronjob(action='run')` 触发后，过 2 分钟仍无输出。

**原因**：`execute_code` 走流式输出（Server-Sent Events），cron 环境里如果网络请求耗时长（如多个新闻源串行抓取），流可能会在 tool-call 执行中间断，但 agent 循环没有重试，导致整轮任务静默消失。cron 系统只检查「模型 API 是否调用成功」，不检查业务逻辑是否有输出，所以 `last_status` 仍为 `ok`。

**判断方法**：手动触发后等 2 分钟，再看 `ls -la ~/.hermes/cron/output/<job_id>/` — 文件时间戳没有更新就是这个问题。

**解法（按稳定性从高到低）**：

1. **最佳：no_agent + script（本次采用的方案）**
   - 把完整逻辑写成 Python 脚本存到 `~/.hermes/scripts/news_push.py`
   - cron job 配置 `no_agent: true`，`script: "news_push.py"`
   - 脚本 stdout 直接作为消息发送，无需 LLM 介入
   - 支持 `[SILENT]` 输出（脚本 print `[SILENT]` 则静默不发送）
   - 脚本内完成：抓取 → 去重 → 生成报告 → 保存 output 文件 → print 报告
   - 脚本路径必须是 `~/.hermes/scripts/<name>.py`（其他路径会被拒绝）

2. **次选：prompt + terminal 工具（仍有 streaming 风险）**
   - prompt 里写 `python3 /home/v-zhengyu002/.hermes/scripts/news_fetch.py`
   - 但 terminal 工具调用本身也走流式，长网络请求仍可能 stall
   - 不推荐用于网络请求密集型任务

3. **已废弃：execute_code**
   - execute_code 在 cron 环境里流式调用网络请求会静默卡死
   - 原因：SSE stream 在 tool-call 执行中间断，agent 无重试，任务静默消失
   - cron 只检查「模型 API 是否调用成功」，不检查业务逻辑是否有输出
   - 所以 `last_status: ok` 并不代表任务真正完成了

**调试方法**：
```bash
# 手动触发后等待，然后检查输出文件时间戳
cronjob(action='run', job_id='d7786a07b9cf')
sleep 120 && ls -la ~/.hermes/cron/output/d7786a07b9cf/ | sort -r | head -3
# 文件时间戳没有更新 → streaming stall 问题
```

**no_agent 模式的限制**：
- 没有 LLM，所以无法做复杂判断（去重、新闻筛选等）
- 脚本必须是全自主逻辑，输出什么就发什么
- 如果脚本需要联网+LLM 判断两个步骤，考虑拆成两个 cron：第一个 no_agent 脚本抓数据，第二个用 LLM 整理（用 `context_from` 关联）

## 验证步骤
1. 国内新闻 ≥ 10 条，每条 ≥ 20 字
2. 国际新闻 ≥ 10 条，每条 ≥ 20 字
3. 标题 emoji 🇨🇳 / 🌍 正确
4. 标题时间词（午间/晚间/夜间）与运行时间匹配
5. 本次内容与历史推送重复率 < 30%
