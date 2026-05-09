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
- 入口: app.py（Flask）
- 数据库: site.db（SQLite，自动创建）
- 虚拟环境 Python: **/home/v-zhengyu002/venv/bin/python**（不是 hermes-agent 的 venv！）
- 模板: templates/index.html（前台）、admin.html、admin_login.html、**news.html、market.html**
- 静态文件: static/favicon.svg
- 新闻历史输出: /home/v-zhengyu002/.hermes/cron/output/d7786a07b9cf/

## 启动/重启

**关键：必须用正确的 Python 路径重启 Flask：**
```bash
kill $(lsof -ti:5000 2>/dev/null)
cd /home/v-zhengyu002/myapp && /home/v-zhengyu002/venv/bin/python app.py >> flask.log 2>&1 &
```

或者用 start.sh（但它内部没有用正确 python 路径）。

## 页面列表

| 路由 | 说明 |
|------|------|
| / | 首页（个人介绍） |
| /news | 📰 新闻页——读取 cron 推送历史，卡片展示最近7天 |
| /market | 📈 行情页——全球指数/A股/加密货币 |
| /admin | 后台管理 |

## 行情页数据源（2026-05 实测，经多次试错确认）

**重要经验：**
- WSL 出口 IP 对 Yahoo Finance 单股票查询返回 "Too Many Requests"，但**指数查询（%5EGSPC）加标准 UA 可用**
- 腾讯财经（qt.gtimg.cn）对国内股票/指数可用，美股指数用 usNDX（纳斯达克100）、usDJI（道琼斯）
- 腾讯财经**不支持** usSPX（标普500），始终返回空

| 类型 | 数据源 | 备注 |
|------|--------|------|
| 标普500 | Yahoo Finance `%5EGSPC` | 必须用 `%5E` 转义，加 User-Agent |
| 纳斯达克100 | 腾讯财经 `usNDX` | |
| 道琼斯 | 腾讯财经 `usDJI` | |
| 恒生指数 | 腾讯财经 `hkHSI` | |
| A股/上证 | 东方财富 `push2.eastmoney.com` | secids=1.000001 |
| 加密货币 | Gate.io `/api/v4/spot/tickers?currency_pair=XXX_USDT` | **每次单独调用**一个币种，不能合并 |

**Yahoo Finance 标普500请求示例：**
```python
r = requests.get(
    'https://query1.finance.yahoo.com/v8/finance/chart/%5EGSPC?interval=1d&range=2d',
    headers={'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36'},
    timeout=8
)
meta = r.json()['chart']['result'][0]['meta']
price = meta['regularMarketPrice']
prev = meta.get('previousClose')  # 注意：可能为 None，要用 .get()
```

**Gate.io 加密货币调用示例：**
```python
for pair in ['BTC_USDT', 'ETH_USDT', 'XRP_USDT', 'DOGE_USDT', 'XLM_USDT', 'LUNC_USDT']:
    r = requests.get(f'https://api.gateio.ws/api/v4/spot/tickers?currency_pair={pair}', timeout=8)
    item = r.json()[0]
    price = float(item['last'])
    chg = float(item['change_percentage'])
```

**腾讯财经解析（GBK编码）：**
```python
r = requests.get(f'https://qt.gtimg.cn/q=usNDX,usDJI,hkHSI', timeout=8)
r.encoding = 'gbk'
# v_usNDX="200~纳斯达克100~.NDX~28406.80~28015.06~..."
# 字段3=当前价，字段4=昨收价
fields = line.split('~')
price = float(fields[3])
prev = float(fields[4])
```

## 新闻页面逻辑

/news 路由读取 `/home/v-zhengyu002/.hermes/cron/output/d7786a07b9cf/` 目录下 .md 文件，
提取 `## Response` 之后的内容，解析 markdown 风格（emoji 标题、- 列表行）转为 HTML 渲染。

**7天自动清理**：每次访问 /news 时，自动删除超过7天的 .md 文件（按文件修改时间 mtime 判断）。

### ⚠️ 新闻解析已知坑（2026-05-07 更新）

**坑1：Python 字符串不可变**
在循环中拼接 HTML 时，**不能用临时变量引用再修改**：
```python
# ❌ 错误！target 是本地副本，+= 修改变量而非原字符串
target = domestic_html if current_section == 'domestic' else global_html
target += '<ul...>'  # domestic_html/global_html 纹丝不动！

# ✅ 正确：直接修改原变量
if current_section == 'domestic':
    domestic_html += '<ul...>'
```

**坑2：`---` 分隔线后的 `📰` 国际标题被跳过（2026-05-07 修复）**
旧代码把 `📰` 行放进"跳过"列表，导致分隔后的国际 section header 被忽略，
`current_section` 一直是 `None`，所有 `🌍` 开头的新闻列表项都被丢弃。

**修复**：在 `app.py` 的 `/news` 路由解析逻辑中，`---` 后遇到 `📰` 开头的行，
要判断其内容是"国内"还是"国际"，相应设置 `current_section`，而不是直接跳过：
```python
# 旧（bug）：
if line_stripped.startswith('---') or line_stripped.startswith('📰'):  # ← 📰被跳过了！
    continue

# 新（修复）：
if line_stripped.startswith('📰'):
    if '国际' in line_stripped or '全球' in line_stripped:
        current_section = 'global'
    elif '国内' in line_stripped:
        current_section = 'domestic'
    continue
if line_stripped.startswith('---'):
    continue
```

**坑3：Cron 双消息只有第一条存文件（重要！）**
cron 任务有时要求模型"发两条消息"（国内+国际），但 cron 系统只把第一条
输出的内容写入 .md 文件，第二条直接送到 QQ，文件里没有。

**修复**：改 cron prompt，强制要求"单条消息包含国内和国际两部分"，禁止用 `---` 分隔成多条。
更新 jobs.json 里 d7786a07b9cf 的 prompt，把"发两条消息"改成：
```
**格式：单条消息包含国内和国际两部分，禁止用 --- 分隔**
📰 X月X日（周X）[午间/晚间/夜间]速报

🇨🇳 中国国内
- [国内新闻...]
...

🌍 国际
- [国际新闻...]
...
```
**全部内容放在一条消息里输出，不要用 --- 拆分成多条。**

### ⚠️ 新闻解析已知坑：Python 字符串不可变

在循环中拼接 HTML 时，**不能用临时变量引用再修改**：
```python
# ❌ 错误！target 是本地副本，+= 修改变量而非原字符串
target = domestic_html if current_section == 'domestic' else global_html
if not in_list:
    target += '<ul class="news-list">\n'  # domestic_html/global_html 纹丝不动！
    in_list = True
target += f'<li>{item}</li>\n'  # 同上

# ✅ 正确：直接修改原变量
if current_section == 'domestic':
    if not in_list:
        domestic_html += '<ul class="news-list">\n'
        in_list = True
    domestic_html += f'<li>{item}</li>\n'
elif current_section == 'global':
    if not in_list:
        global_html += '<ul class="news-list">\n'
        in_list = True
    global_html += f'<li>{item}</li>\n'
```

**调试方法**：在 Flask 目录下直接用 venv python 运行解析逻辑，对单个 .md 文件输出 domestic_html/global_html 长度，验证是否为 0（0 = bug）。

### 新闻空内容过滤

Cron 任务有时会产生空的 .md 文件（API 失败），Flask 解析时通过检查 `domestic_html` 和 `global_html` 去掉所有 HTML 标签后是否只剩空白来过滤：
```python
domestic_stripped = domestic_html.replace('<ul class="news-list">','').replace('</ul>','') \
    .replace('<h3 class="news-section">','').replace('</h3>','') \
    .replace('<li>','').replace('</li>','').replace('<p>','').replace('</p>','').strip()
if not domestic_stripped and not global_stripped:
    continue  # 跳过空条目
```

## 导航栏

index.html 顶部 nav 包含：首页、技能、经历、新闻、行情、联系。

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

## 性能动画页面

路由: `/template/perf` → 加载 `templates/index_perf.html`

Three.js 动画页面，曲线+柱状图，全屏模式（无导航/无内容区块）。通过 URL 参数 `?v=时间戳` 控制缓存刷新。

**启动服务器**（必须用 venv python）：
```bash
ps aux | grep app.py | grep -v grep  # 检查是否在跑
kill $(cat /home/v-zhengyu002/myapp/flask.pid) 2>/dev/null  # 杀旧进程
/home/v-zhengyu002/venv/bin/python /home/v-zhengyu002/myapp/app.py >> /home/v-zhengyu002/myapp/flask.log 2>&1 &
echo $! > /home/v-zhengyu002/myapp/flask.pid
# 验证
curl -s -o /dev/null -w "%{http_code}" http://localhost:5000/template/perf
```

**start.sh**（在 `/home/v-zhengyu002/myapp/start.sh`）会同时启动 Flask 和 cloudflared tunnel，可用。但 start.sh 内部 python 路径写死了 `/home/v-zhengyu002/venv/bin/python`，是正确路径。

## 常见问题

### 网站打不开
1. 检查 Flask 进程: `curl localhost:5000`
2. 检查 cloudflared: `ps aux | grep cloudflared`
3. 检查 DNS: Cloudflare Dashboard → DNS Records 确认 myapp 记录存在
4. 手机切 4G/5G 验证（公司内网 DNS 可能延迟）
5. 如 Flask/cloudflared 不在，运行 `/home/v-zhengyu002/myapp/start.sh`

### Three.js 动画页面报 "THREE is not defined"
**原因**：通过 Cloudflare Tunnel 访问时，外部 CDN（cdnjs.cloudflare.com）的 SSL 握手失败（`ERR_SSL_BAD_RECORD_MAC_ALERT`），Three.js 未加载。

**修复**：把外部 CDN 的 JS 库下载到本地 static 目录，改成本地引用：
```bash
# 下载
curl -s https://cdnjs.cloudflare.com/ajax/libs/three.js/r128/three.min.js \
  -o /home/v-zhengyu002/myapp/static/three.min.js

# 模板里改引用
# 旧: <script src="https://cdnjs.cloudflare.com/ajax/libs/three.js/r128/three.min.js"></script>
# 新: <script src="/static/three.min.js"></script>
```

### 动画页刷新后没反应 / 改代码没生效
强制刷新（Ctrl+Shift+R），因为浏览器缓存导致 JS 不更新。也可以在 URL 后面加时间戳参数绕过缓存：
```
https://myapp.zhengyy.com/template/perf?v=1699999999
```

### 改端口
1. 改 app.py 里的 port=5000
2. 去 Cloudflare Dashboard → Tunnels → myapp → Routes 改 Service URL
3. 重启: `/home/v-zhengyu002/myapp/start.sh`

### 修改管理密码
编辑 app.py 里的 ADMIN_PASSWORD 变量