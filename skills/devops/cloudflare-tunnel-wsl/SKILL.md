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

### 5. 启动本地服务和隧道

```bash
# 终端1: 启动本地服务
python3 -m http.server 8080 --directory /path/to/site &

# 终端2: 启动 cloudflared
~/.local/bin/cloudflared tunnel run --token '<token>'
```

### 6. 验证

- 用手机 4G/5G 打开 https://myapp.zhengyy.com（不要用公司 WiFi，内网 DNS 可能没刷新）
- 或 curl -H "Host: myapp.zhengyy.com" http://<cloudflare_ip>/

## 常见问题

### DNS 不解析
- WSL 用公司内网 DNS（/etc/resolv.conf 里 nameserver 10.x.x.x），新记录可能没缓存
- 公网 DNS（Cloudflare/Google DoH）通常几分钟内生效
- 手机切 4G/5G 验证

### tunnel login 回调失败
- `cloudflared tunnel login` 需要浏览器回连本地端口，WSL 无公网 IP 时可能失败
- 替代方案：用 Cloudflare Dashboard 创建隧道拿 token

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