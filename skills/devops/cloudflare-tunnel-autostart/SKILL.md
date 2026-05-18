---
name: cloudflare-tunnel-autostart
title: Cloudflare Tunnel 自动启动 (systemd 用户服务)
description: 在 WSL 环境下配置 systemd 用户服务让 Cloudflare Tunnel 自动开启，解决重启后网站 1033 错误。
authors: Hermes Agent
summary: 步骤具体帮助、常见坑点和验证误误等。
---

# 触发条件
- WSL 环境，不得用 sudo，由 systemd \`--user\` 管理服务。
- 需要一个達到 Cloudflare 的 tunnel (myapp) 和本地的 HTTP 端口 (e.g., 8080)。

# 步骤概览
1. **获取完整 token**
   ```bash
   cat ~/.cloudflared/tunnel.token   # 应有 248 字符，完整不被截断
   ```
2. **编写启动脚本 (optional)**
   ```bash
   cat > ~/cloudflared-run.sh <<'EOS'
   #!/usr/bin/env bash
   # tunnel --token 方式不需要 tunnel 名称参数，也不需要 --url（已在 config.yml 里配了）
   /home/v-zhengyu002/.local/bin/cloudflared tunnel run \
       --token "$(cat /home/v-zhengyu002/.cloudflared/tunnel.token)"
   EOS
   chmod +x ~/cloudflared-run.sh
   ```
3. **创建 systemd 服务** `~/.config/systemd/user/cloudflared.service`
   ```ini
   [Unit]
   Description=Cloudflare Tunnel for myapp.zhengyy.com
   After=network-online.target
   Wants=network-online.target

   [Service]
   # 用 token-file 方式避免 shell 参数里有敏感信息
   ExecStart=/home/v-zhengyu002/.local/bin/cloudflared tunnel run \
       --token-file /home/v-zhengyu002/.cloudflared/tunnel.token
   # 或者用脚本：
   # ExecStart=/home/v-zhengyu002/cloudflared-run.sh
   Restart=on-failure
   RestartSec=5
   StartLimitInterval=0

   [Install]
   WantedBy=default.target
   ```
4. **重读 systemd 配置并启动服务**
   ```bash
   systemctl --user daemon-reload
   systemctl --user enable cloudflared.service
   systemctl --user start cloudflared.service
   ```
5. **验证**
   ```bash
   systemctl --user status cloudflared.service   # 应显示 Active: active (running)
   curl -I https://myapp.zhengyy.com/          # 应返回 200 或对应页面
   ```

# 常见坑点 & 解决方案
- **Token 被截断**：在编辑或 `echo` 输出时常看到 `eyJhIj...eCJ9` 。必须使用 `cat` 完整读取，或将 `~/.cloudflared/tunnel.token` 内容直接复制到 service 中。
- **本地 HTTP 服务不存在**：隧道在没有应用端口运行时，会在服务尝试连接后自动关闭并出现 Error 1033。启动一个简单的 HTTP 服务，例如 `python -m http.server 8080`，或指向实际应用端口。
- **ICMP proxy 警告**：`cloudflared` 将警告 GID 不在 `ping_group_range`，这不会影响隧道使用，可忽略。若需消除警告，需 sudo 权限更改 `/proc/sys/net/ipv4/ping_group_range`。
- **systemd 用户服务未启动**：WSL 默认不使用 user linger，可以 `loginctl enable-linger $(whoami)` 开启之后重启会自动启动服务。
- **服务持续在 `activating (auto-restart)`**
  - 常见原因是 token 写入错误，或本地端口不可达。可使用 `journalctl --user -u cloudflared.service` 查看详细日志。

# 验证步骤
1. `systemctl --user status cloudflared.service` 如果显示 `Active: active (running)` 则成功。
2. **核心验证** — 看 tunnel 日志确认实际建立了连接：
   ```bash
   journalctl --user -u cloudflared.service -f &
   # 或直接看 cloudflared 进程日志
   # 找到 "Registered tunnel connection" = tunnel 真正在跑
   # 如果只有 "Requesting new quick Tunnel" = 没连上
   ```
3. **外网访问测试**：
   ```bash
   curl -s --connect-timeout 10 https://myapp.zhengyy.com/ -L | head -5
   # 返回 HTML = 成功
   # 返回 502 = credentials 文件缺失或路由未配（参考 cloudflare-tunnel-wsl 技能）
   ```
4. 重启 WSL 后，重新 `systemctl --user status cloudflared.service` 进行检查。

# 参考文件
- `references/token_handling.md` – 如何安全查看完整 token，避免省略。
- `templates/cloudflared.service.ini` – 可直接复制的 systemd service 模板文件。
