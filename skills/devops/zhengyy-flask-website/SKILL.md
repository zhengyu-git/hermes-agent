---
name: zhengyy-flask-website
description: Zhengyu 个人网站 — Flask + SQLite 后台管理，Apple 风格，跑在 WSL + Cloudflare Tunnel
version: 1.0.0
triggers:
  - 个人网站
  - myapp.zhengyy.com
  - 网站后台
  - 网站挂了
  - 网站连不上
  - zhengyy 网站
---

# Zhengyu 个人网站

## 访问地址

- 前台: https://myapp.zhengyy.com
- 后台: https://myapp.zhengyy.com/admin（密码: admin123）

## 项目位置

- 代码: /home/v-zhengyu002/myapp/
- 入口: app.py（Flask 3.0.2）
- 数据库: site.db（SQLite，自动创建）
- 虚拟环境: /home/v-zhengyu002/venv/bin/python
- 模板: templates/index.html（前台）、admin.html、admin_login.html
- 静态文件: static/favicon.svg

## 启动/重启

```bash
/home/v-zhengyu002/myapp/start.sh
```

start.sh 会：
1. 杀掉旧的 Flask 和 cloudflared 进程
2. 启动 Flask（端口 5000）
3. 启动 cloudflared tunnel

## 后台管理功能

- 修改 Hero 区文字（姓名/职位/描述）
- 修改各区域标题和副标题
- 增删技能标签
- 增删工作经历
- 增删联系方式

## 数据库初始化

首次运行 app.py 时自动创建 site.db 并写入默认数据。如需重置，删除 site.db 后重启 Flask。

## 隧道信息

- 名称: myapp
- ID: 83172e60-ae0a-410f-903f-243d29be1781
- 路由: myapp.zhengyy.com → http://localhost:5000
- cloudflared: ~/.local/bin/cloudflared v2026.3.0

## 常见问题

### 网站打不开
1. 检查 Flask 进程: `curl localhost:5000`
2. 检查 cloudflared: `ps aux | grep cloudflared`
3. 检查 DNS: Cloudflare Dashboard → DNS Records 确认 myapp 记录存在
4. 手机切 4G/5G 验证（公司内网 DNS 可能延迟）
5. 如 Flask/cloudflared 不在，运行 start.sh

### 改端口
1. 改 app.py 里的 port=5000
2. 去 Cloudflare Dashboard → Tunnels → myapp → Routes 改 Service URL
3. 重启: `/home/v-zhengyu002/myapp/start.sh`

### 修改管理密码
编辑 app.py 里的 ADMIN_PASSWORD 变量