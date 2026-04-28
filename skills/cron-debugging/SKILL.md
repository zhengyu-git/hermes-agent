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

**注意**：`last_status: "ok"` 不代表任务真正执行成功！cron 系统只判断「是否成功调用了模型 API」，不判断业务逻辑是否完成。即使 Response 内容是错误信息，只要没抛异常，`last_status` 也会是 `ok`。

### 2. 查看实际输出文件

```
ls ~/.hermes/cron/output/<job_id>/
read_file ~/.hermes/cron/output/<job_id>/<date>.md
```

重点关注 `## Response` 部分，这才是任务实际产出的内容。常见失败模式：

- `API call failed after 3 retries: HTTP 404: 404 page not found` → 模型端点问题（下线/改名/key失效）
- 网络超时 → WSL 网络限制
- 空响应 → 模型返回异常

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
| last_status=ok 但没收到推送 | 任务实际执行失败了，检查 output 文件 |
| HTTP 404 | 模型端点不可用 |
| 三个任务同时失败 | 大概率是 provider 侧问题，不是任务配置问题 |