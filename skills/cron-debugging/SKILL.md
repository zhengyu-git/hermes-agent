---
name: cron-debugging
description: 排查定时任务(cron)失败问题的方法论
category: devops
---

# Cron 任务排错指南

## 触发条件

当用户反馈 cron 任务失败/没收到推送时，使用本流程。

## 排错步骤

### 1. 查看 cron 状态（仅供参考）

```
cronjob(action='list')
```

**注意**：`last_status: "ok"` 不代表任务真正执行成功！cron 系统只判断「是否成功调用了模型 API」，不判断业务逻辑是否完成。即使 Response 内容是错误信息，只要没抛异常，`last_status` 也会是 `ok`。**永远不要以 `last_status` 为依据判断任务结果，必须直接看 output 文件。**

### 2. 查看实际输出文件

```
ls ~/.hermes/cron/output/<job_id>/
read_file ~/.hermes/cron/output/<job_id>/<date>.md
```

重点关注 `## Response` 部分，这才是任务实际产出的内容。常见失败模式：

- `API call failed after 3 retries: HTTP 404: 404 page not found` → 模型端点问题（下线/改名/key失效）
- `API call failed after 3 retries: Connection error.` → API 服务端连不上（provider 宕机/网络中断/临时故障），通常过几小时自愈
- 网络超时 → WSL 网络限制
- 空响应 → 模型返回异常

**当多个 cron 任务在同一时段全部失败且报错相同（如全是 Connection error）**：说明是 provider 侧故障，不是任务配置问题。不需要逐一排查每个任务。直接看 output 文件开头几行确认，然后告知用户「API 当时挂了一会儿，后面自己恢复了」。

**注意**：当多个 cron 任务在同一时段全部失败且报错相同（如全是 Connection error），说明是 provider 侧故障，不是任务配置问题。不需要逐一排查每个任务。

### 3. 检查模型配置

```
read_file ~/.hermes/cron/jobs.json
```

查看每个 job 的 `model`/`provider`/`base_url` 字段：
- 如果是 `null`，走 config.yaml 的默认模型
- 如果显式指定了，检查该 provider 是否可用

### 4. 手动补跑

```
cronjob(action='run', job_id='<job_id>')
```

## 常见问题

| 现象 | 原因 |
|------|------|
| last_status=ok 但没收到推送，且 output 文件没有更新 | 任务实际执行失败，检查 output 文件 |
| HTTP 404 | 模型端点不可用 |
| 三个任务同时失败 | 大概率是 provider 侧问题，不是任务配置问题 |
| output 文件时间戳不更新（cron run 后仍无新文件） | `execute_code` 在 cron 流式环境里网络请求卡死，导致整轮静默消失。改用 `terminal` 执行 Python 脚本。详见 `news-push-cron-debug` 技能。 |