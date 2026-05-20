---
name: cloudflare-tunnel-wsl
description: 在 WSL 中用 Cloudflare Tunnel 搭建公网可访问的网站（不需要公网 IP）
version: 2.0.0
triggers:
  - cloudflare tunnel
  - 内网穿透
  - 公网访问 wsl
  - 域名 + wsl
  - cloudflared
---

# Cloudflare Tunnel + WSL 搭建公网网站

## 前提条件

1. 域名 DNS 托管在 Cloudflare（或能从原注册商改 NS 到 Cloudflare）
2. WSL 能访问外网
3. 不需要公网 IP、不需要开防火墙端口

## 步骤

### 1. 域名 DNS 切到 Cloudflare

- 阿里云/其他注册商后台，把 NS 改为 Cloudflare 提供的两个地址（如 bingo.ns.cloudflare.com / woz.ns.cloudflare.com）
- 等待 NS 全球生效（几分钟到几小时）

### 2. WSL 安装 cloudflared

```bash
# 下载 deb（无需 sudo）
curl -LO https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-amd64.deb
dpkg-deb -x cloudflared-linux-amd64.deb /tmp/cloudflared_extract
cp /tmp/cloudflared_extract/usr/bin/cloudflared ~/.local/bin/
chmod +x ~/.local/bin/cloudflared
```

### 3. 创建隧道（推荐用 Cloudflare Dashboard 方式，避免本地回调问题）

- 访问 https://one.dash.cloudflare.com → Networks → Tunnels
- Create a tunnel → 选 Cloudflared → 命名
- 复制 token

### 4. 添加 Public Hostname（路由）

- 在隧道详情页 → Routes → Add route → Published application
- Subdomain: myapp, Domain: zhengyy.com → myapp.zhengyy.com
- Service URL: http://localhost:5000（改成实际端口）
- Save（DNS 自动创建 CNAME 记录）

### 5. 快速验证本地服务（无需配置 Cloudflare 账户）

在正式配隧道之前，先用无配置模式验证本地服务能否被外网访问：

```bash
# 启动临时隧道，生成一个 trycloudflare.com 临时地址
cloudflared tunnel --url http://localhost:8080
```

成功后会显示类似 `https://xxxx.trycloudflare.com` 的地址，用手机 4G 打开这个地址验证。如果能访问到本地页面，说明本地服务正常，只是隧道配置问题。

**注意**：trycloudflare.com 是临时地址（每次重启都变），不能替代正式隧道。

### 6. 启动本地服务和正式隧道

**⚠️ 注意 flag：要用 `--log-level`，不是 `--loglevel`，且要放在 `run` 之前！**

```bash
# 终端1: 启动本地服务（Python HTTP 示例）
cd /home/v-zhengyu002/test-site
python3 -m http.server 8080 &
# 或 Flask 示例：
/home/v-zhengyu002/venv/bin/python /home/v-zhengyu002/myapp/app.py >> /home/v-zhengyu002/myapp/flask.log 2>&1 &

# 终端2: 启动 cloudflared（⚠️ --log-level 在 run 前面！）
~/.local/bin/cloudflared tunnel --log-level info run --token '<token>'
```

也可以用 token 文件（避免 token 暴露在命令行）：
```bash
echo '<token>' > ~/.cloudflared/tunnel.token
~/.local/bin/cloudflared tunnel --log-level info run --token-file ~/.cloudflared/tunnel.token
```

### 6. 验证

- 用手机 4G/5G 打开 https://myapp.zhengyy.com（不要用公司 WiFi，内网 DNS 可能没刷新）
- 或 curl -H "Host: myapp.zhengyy.com" http://<cloudflare_ip>/

### Token 管理（重要）

**⚠️ 不要硬编码 token！** 每次 token 失效/重置后都要更新启动脚本，很容易漏掉。

推荐做法：token 统一存一份在 `~/.cloudflared/tunnel.token`，启动脚本直接引用：
```bash
TOKEN_FILE=~/.cloudflared/tunnel.token
TOKEN=$(cat "$TOKEN_FILE")
~/.local/bin/cloudflared tunnel run --token "$TOKEN"
```

如果硬编码了旧 token，症状是：
- `cloudflared tunnel run --token '<old>'` 前台跑 → 直接报错 `Provided Tunnel token is not valid`
- 后台跑 → 进程立即退出，网站报 1033

### 常见问题

#### Tunnel 报 "control stream encountered a failure while serving" 并持续重试
**原因**：token 失效或过期，但错误信息不是直接报"无效 token"，而是表现为控制流失败。
**排查**：用 `timeout 10 cloudflared tunnel --log-level info run --token <token>` 在前台跑，看是否在 10 秒内成功注册连接（看到 `Registered tunnel connection`）。如果一直重试不注册，换新 token。

#### cloudflared 没有任何日志输出就退出了
**原因**：`--log-level`（或错误的 `--loglevel`）flag 位置错了，cloudflared 把它当成未知 flag 静默忽略并退出（exit code 0）。
**正确**：`cloudflared tunnel --log-level info run`（`--log-level` 在 `run` 之前），不是 `cloudflared tunnel run --log-level info`。

#### Named tunnel 502 — 光有 token 不够，还需要 credentials 文件
**表现**：`cloudflared tunnel run --token '<token>'` 进程在跑，日志里有 `Registered tunnel connection connIndex=0`，但访问域名返回 502。
**根因**：Named tunnel（用 `cloudflared tunnel create` 创建的那种）除了需要 token，还需要 **credentials 文件**（`~/.cloudflared/<uuid>.json`），这是 `cloudflared tunnel create` 时自动生成的。token 负责识别"哪个 tunnel"，credentials 文件负责加密握手。如果 credentials 文件不存在或路径不对，Cloudflare edge 能看到 tunnel 连上来了，但无法建立安全连接，返回 502。
**排查**：
```bash
ls ~/.cloudflared/
# 应该看到 tunnel.token 和 <uuid>.json 两个文件
# 如果只有 tunnel.token，说明缺少 credentials 文件
```
**解决**：
1. **方案 A（推荐）**：去 Cloudflare Dashboard → Networks → Tunnels → 点击 tunnel 名称（不是 Actions 按钮）→ 详情页 → 底部有 `Download credentials file` → 下载 json 文件 → 放到 `~/.cloudflared/<uuid>.json`
2. **方案 B（更干净）**：删掉旧 tunnel，在 Dashboard 重新 Create a tunnel → 选 Cloudflared → 命名（如 myapp2）→ Download credentials file → 新 token 存到 `~/.cloudflared/tunnel.token` → credentials json 存到 `~/.cloudflared/<new-uuid>.json`
   - 然后更新 DNS CNAME 记录指向新 tunnel ID
   - 更新 systemd 服务里的 token
3. 重启 cloudflared 进程即可

**注意**：Dashboard UI 可能不显示 Download credentials 按钮（取决于 Cloudflare 版本）。如果方案 A 找不到按钮，走方案 B 重建 tunnel。

**临时方案**：用 `cloudflared tunnel --url http://localhost:8080`（快速隧道）绕过 named tunnel 的 credentials 要求。快速隧道不需要任何凭证，每次重启地址会变，但能用。

#### Tunnel 状态显示 "Down" 但实际能访问
**表现**：Dashboard 里 tunnel 卡片显示红色 "Down"，但 `myapp.zhengyy.com` 实际能正常访问。
**原因**：Cloudflare Dashboard 的 "Down" 状态检测的是**最后一条连接**的时间戳，如果 tunnel 长期不断开（QUIC 长连接），Dashboard 的健康检查会误判为离线（实际上 tunnel 在跑，只是没有新建连接来刷新状态）。
**判断方法**：
```bash
# 看 tunnel 日志里有没有 "Registered tunnel connection"
cat ~/.hermes/myapp/tunnel.log | grep "Registered tunnel connection"
# 有记录 = tunnel 实际在运行
```
**结论**：以实际访问为准，不要以 Dashboard 状态为准。

#### cloudflared tunnel --url（快速隧道）和 tunnel run（命名隧道）的区别
- `cloudflared tunnel --url http://localhost:8080`：快速隧道，**不需要** credentials/token，Cloudflare 随机分配 `.trycloudflare.com` 地址，适合测试，每次重启地址都变
- `cloudflared tunnel run --token '<token>'`：命名隧道，**需要** credentials 文件 + token，绑定到你在 Dashboard 创建的 tunnel，适合正式使用

两者不能混用：快速隧道不需要任何凭证；命名隧道光有 token 不够，必须有 credentials 文件。

#### DNS 不解析
- WSL 用公司内网 DNS（/etc/resolv.conf 里 nameserver 10.x.x.x），新记录可能没缓存
- 公网 DNS（Cloudflare/Google DoH）通常几分钟内生效
- 手机切 4G/5G 验证

### tunnel login 回调失败
- `cloudflared tunnel login` 需要浏览器回连本地端口，WSL 无公网 IP 时可能失败
- 替代方案：用 Cloudflare Dashboard 创建隧道拿 token

### "control stream encountered a failure while serving" 错误
- tunnel 连上了 Cloudflare edge 但握手失败，持续重试（2s→4s→8s→...）
- **原因**：token 失效或被撤销
- **排查**：看进程日志 `grep -i error ~/.cloudflared/*.log` 或 `process log` 会话
- **解决**：需要去 Dashboard 重新生成 token（见下文）

### Token 失效后无法通过 CLI 重新获取
- `cloudflared tunnel token <name>` 需要 cert.pem（origin certificate）
- 如果没有 cert.pem，命令报错：`Cannot determine default origin certificate path`
- **解决**：必须去 Dashboard 手动重新生成 token
  1. 访问 https://dash.cloudflare.com → Networks → Tunnels
  2. 找到目标 tunnel → Actions → Regenerate token
  3. 复制新 token，更新启动命令

### Cloudflare Dashboard 本身被安全检查拦截
- 浏览器打开 dash.cloudflare.com 时出现 "Just a moment..." 验证页面
- 浏览器也被当作机器人拦了，无法管理 tunnel
- 备用方案：用手机 4G 热点访问，或从浏览器开发者工具手动过验证

#### Error 1033 — Argo handshake failed
**表现**：`cloudflared` 进程不在跑，或进程启动后立即退出（exit 0 无日志）
**原因**：token 失效（记忆里存的旧 token）
**排查**：`~/.local/bin/cloudflared tunnel run --token "$(cat ~/.cloudflared/tunnel.token)"` 前台跑，看是否报 `Provided Tunnel token is not valid`
**解决**：去 Dashboard 重新生成 token（Networks → Tunnels → Actions → Regenerate token），新 token 存回 `~/.cloudflared/tunnel.token`

#### Error 502 from Cloudflare edge — 两种完全不同的原因

**原因 A**：隧道进程根本没跑起来（之前记录的 1033/旧token情况）
- 排查：`ps aux | grep cloudflared` 确认进程在
- `curl localhost:<port>` 确认本地服务在

**原因 B**：隧道进程在跑，但 Dashboard 里没配 Public hostname 路由
- 表现：`ps aux | grep cloudflared` 有进程，`curl localhost:<port>` 正常返回，DNS 也解析到 Cloudflare IP，但域名访问返回 502
- 根因：Cloudflare Dashboard → Networks → Tunnels → 找到对应 tunnel → Routes 页面里，Public hostname 列表为空——隧道本身连上了 Cloudflare edge，但 Cloudflare 不知道要把哪个域名路由到这个 tunnel
- 排查：
  1. `nslookup myapp.zhengyy.com` 确认 DNS 已指向 Cloudflare IP（如 172.67.x.x）
  2. cloudflared 日志里有 `Registered tunnel connection`（说明 tunnel 连上了）
  3. 但 dashboard 里该 tunnel 没有 Public hostname 路由记录
- 解决：去 Dashboard 给 tunnel 添加 Public hostname 路由（Add route → Published application → subdomain + domain + service URL），保存后 DNS 自动生效，无需重启 tunnel 进程

**先快速验证法**（`cloudflared tunnel --url http://localhost:PORT`）确认本地服务正常，再判断是原因 A 还是 B。

#### 502 的另一个原因：重启 cloudflared 时本地服务也断了

重启 cloudflared 进程时，如果本地 HTTP 服务没有守护进程，它会跟着终端一起退出，导致 502。

**排查**：先确认本地服务还在——`ss -tlnp | grep :5000`（或实际端口）

**原则**：重启 cloudflared 前先确保本地服务活着，或者用 supervisor/pm2 守护两个进程

### 端口改动后不生效
- 改完路由的 Service URL 后，需要重启 cloudflared 进程

### WSL 无 sudo/pip 安装 Flask
- 用 `apt download` 下载 deb 包
- `dpkg-deb -x` 手动解压依赖到 site-packages
- 手动创建 venv：`mkdir -p venv/bin && cp /usr/bin/python3 venv/bin/python`

## 文件位置

- cloudflared: ~/.local/bin/cloudflared
- token 保存在启动脚本中
- zhengyy.com 项目: ~/myapp/（Flask + SQLite 个人网站）
- 正式配置（推荐）: `~/.cloudflared/config.yml` + credentials JSON，参考 `templates/cloudflared-config.yml`

## 开机自启（WSL）

WSL 没有 systemd，用 Windows 任务计划程序代替：

1. 创建启动脚本 `~/start_tunnel.sh`：
```bash
#!/bin/bash
~/.local/bin/cloudflared tunnel run --token "$(cat ~/.cloudflared/tunnel.token)" --log-level info
```
2. 添加 Windows 任务计划程序：触发器选"登录时"，操作选"启动程序"填 `wsl.exe -e bash /home/v-zhengyu002/start_tunnel.sh`

> 注意：token 一旦失效（"control stream" 错误），即使重启进程也无效，必须重新从 Dashboard 获取新 token。