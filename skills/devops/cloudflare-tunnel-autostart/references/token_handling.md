# Token 完整性检查

本地 token 存放于 `~/.cloudflared/tunnel.token`，内容应为一行完整的 JWT 字符串（约 248 字符）。

**常见误区**
- 使用 `cat ~/.cloudflared/tunnel.token | wc -c` 确认长度为 248。
- 在编辑器或终端直接复制粘贴时，可能只显示 `eyJhIj...eCJ9`（被省略），导致服务启动失败。
- 解决办法：
  ```bash
  cat ~/.cloudflared/tunnel.token   # 直接查看完整字符串
  # 如需在 service 中使用，手动替换占位符 <YOUR_FULL_TOKEN>
  ```

**建议**：在 `cloudflared.service` 中直接使用完整 token（如示例所示），或写一个包装脚本（`cloudflared-run.sh`），在脚本内部通过 `$(cat ...)` 读取，以避免手动复制错误。