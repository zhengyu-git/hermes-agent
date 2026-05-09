---
name: qqbot-channel-troubleshooting
description: QQBot 频道问题诊断 — 涵盖 11263 错误、WebSocket 重连、以及 DM/群频道架构差异导致的 send_message 失败
triggers:
  - QQBot send failed 11263
  - 频道不存在
  - WebSocket closed code=4009 Session timed out
  - gateway instance already running
  - send_message 在私聊正常但发送失败
---

# QQBot 频道失效 / 恢复流程

## 症状
- `send_message` 返回 `11263: 频道不存在`
- 日志显示 `WebSocket closed: code=4009 reason=Session timed out`（每30分钟一次）
- `QQBOT_HOME_CHANNEL` 中保存的 channel ID 失效

## 诊断

1. **查看 gateway 日志**
   ```
   tail -30 ~/.hermes/logs/agent.log
   journalctl --user -u hermes-gateway.service --since="10 min ago"
   ```
2. **检查 gateway 进程状态**
   ```
   ps aux | grep "gateway run" | grep -v grep
   hermes gateway status
   ```
3. **查看 errors.log**
   ```
   cat ~/.hermes/logs/errors.log
   ```

## 核心修复

**不要用 `systemctl restart hermes-gateway.service`** — 容易和其他 hermes 进程冲突。改用：

```bash
hermes gateway stop
sleep 2
hermes gateway start
```

**清除失效的 channel ID：** 用文本编辑器将 `QQBOT_HOME_CHANNEL=` 后面的值删掉（保留等号和空值），然后重启：

```bash
hermes gateway restart
```

**重新建立 channel：** bot 重连后，用户需在 QQ 上主动给 bot 发一条消息，bot 才能获得新的有效 channel ID。

## 根本原因
QQ WebSocket session 每 30 分钟超时断开，旧 channel ID 在 bot 重连后失效，但 `sessions.json` 和 `QQBOT_HOME_CHANNEL` 仍保留旧值。

## 验证
```
# 检查 gateway 是否 Running + QQ 是否 Ready
ps aux | grep "gateway run" | grep -v grep
grep "Ready, session_id" ~/.hermes/logs/agent.log

# 发送测试消息（通过 CLI 的 send_message 工具，不是命令行）
# 用 target qqbot 或 qqbot:42E7A93D2B848DB7776DB63BAE50FE2E
```

## 关键架构发现（2026-04-23）

**DM 频道 vs HTTP API 频道是两套独立的通道：**
- bot 私聊（DM）收到的消息走 WebSocket，gateway 回复也走 WebSocket，**完全正常工作**
- `send_message` 工具走的是 QQ Open Platform HTTP API，**不认 WebSocket DM 频道**，返回 11263
- `send_message` 的 target list 显示的 DM ID 实际上是 WebSocket 会话中的临时 channel，对 HTTP API 无效

**只有群频道（Group Channel）才能同时支持 WebSocket 和 HTTP API。**

解决方案：把 bot 加入一个 QQ 群，在群里发消息，然后将 `QQBOT_HOME_CHANNEL` 设为该群频道 ID（格式类似 `-100XXXXXXXX` 的数字 ID）。私聊无法用于 send_message。

## 额外发现
- `send_message` 工具 list 目标时显示的 channel ID 不一定有效（session 重连后可能已变）
- 即使 gateway 日志显示 "Ready, session_id=..."，旧 channel ID 仍可能失效（11263）
- 此时清空 `QQBOT_HOME_CHANNEL=` 后，sessions.json 里的旧 session 映射也会在新消息来时自动更新，不需要手动删
- `sessions.json` 的 `updated_at` 如果没有更新，说明 bot 还没收到新消息
- DM 场景下 gateway 能正常回复（"Sending response" 日志可见），但 send_message HTTP API 永远失败 — 这是架构问题，不是配置问题
