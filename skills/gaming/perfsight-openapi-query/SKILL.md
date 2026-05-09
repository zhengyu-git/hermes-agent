---
name: perfsight-openapi-query
description: "PerfSight OpenAPI 查询 — 条件查询对局、获取单局详情（FPS曲线/CPU/GPU/内存）。支持按 UID/场景/时间筛选。"
category: gaming
---

# PerfSight OpenAPI 查询

## ⚠️ 用户偏好（输出模版）

1. **用"场景结束时间"标识对局**，不要用开始时间
2. **所有指标（FPS、Jank 等）必须取 PerfSight 平台算好的值**，不准自己算
   - 平均 FPS → 从 conditionalSearch 返回的 `fps_mean` 字段取
   - 最低 FPS → `fpsMin`
   - 卡顿 → `janks`、`pdbigjank`、`seriousFrozen`
   - CPU/GPU → `cpu_mean`、`gpuUsage`
   - 自己从 `fps_frame_data.FpsData` 算的值仅供参考，不放进模版

## 凭证

| 参数 | 值 |
|------|-----|
| APP_KEY | `116a83402fa27c74fe0e571f4a32a69d` |
| APP_ID | `348901537` |
| env | `v2`（国内环境） |
| username | `traviszeng`（三个可选：Shen, traviszeng, dannyzhou，traviszeng 已验证可用） |
| BASE_URL | `https://api.perfsight.qq.com` |

## ⚠️ 已知坑点

1. **detail 接口参数名不同**: 用 `Appid` 而不是 `app_id`，且**不需要 `env` 参数**
2. **大世界场景名**: `/Game/Maps/World/World_P.World_P`（不是 "大世界"）
3. **FPS 数据位置**: 在 `fps_frame_data.FpsData` 里，不是 `fps` 字段
4. **JWT token**: 每次查询都要重新生成，exp = time.time() + 120

## 通用 JWT 生成

```python
import time
import jwt

APP_KEY = "116a83402fa27c74fe0e571f4a32a69d"

def generate_token(payload):
    exp = int(time.time() + 120)
    payload["exp"] = exp
    headers = {"alg": "HS256", "typ": "JWT"}
    return jwt.encode(payload=payload, key=APP_KEY, algorithm="HS256", headers=headers)
```

## 接口1: 条件查询 `/openapi/single/conditionalSearch`

用于查找对局记录，获取 `performance_id`。

```python
import requests
import json

APP_ID = "348901537"
USERNAME = "traviszeng"
BASE_URL = "https://api.perfsight.qq.com"

payload = {
    "stime": "2026-04-01 00:00:00",
    "etime": "2026-04-24 23:59:59",
    "version": [],              # 不限版本
    "platform": "5",            # 0=Android, 1=iOS, 5=Windows
    "userName": USERNAME,
    "size": 100,                # 最大 10000
    "source": 1,                # 1=国内, 2=国际
    "timeSortField": 2,         # 1=场景开始时间, 2=场景上传时间
    "requestid": f"{USERNAME}_conditionalSearch_openapi",
    "model": [],                # 不限机型
    "userIdList": ["454827175589891"],  # UID 列表
    "scene": ["/Game/Maps/World/World_P.World_P"]  # 可选，筛选场景
}

token = generate_token(payload)
params = {
    "app_id": APP_ID,   # 注意：这里是 app_id
    "env": "v2",
    "username": USERNAME,
    "token": token
}

resp = requests.post(
    f"{BASE_URL}/openapi/single/conditionalSearch",
    json=params,
    headers={"Content-Type": "application/json"}
)

result = resp.json()
# 数据路径: result['data']['ret']['data']['data']['result']
records = result['data']['ret']['data']['data']['result']
```

### 返回字段（关键）

| 字段 | 说明 |
|------|------|
| `performance_id` | 对局ID，用于 detail 接口 |
| `scene_name` | 场景名称 |
| `scene_time` | 场景时间 |
| `fps_mean` | 平均FPS |
| `fpsMin` | 最低FPS |
| `frame_count` | 帧数 |
| `scene_last_time` | 持续时间（毫秒） |
| `valid` | 是否有效 |

## 接口2: 单局详情 `/openapi/single/detail`

获取 FPS 曲线、CPU、GPU、内存等秒级数据。

**⚠️ 参数名与 conditionalSearch 不同！**

```python
perf_id = "1497155318955202560"  # 从条件查询获取

payload = {}  # detail 接口的 JWT payload 只需要 exp
token = generate_token(payload)

params = {
    'username': USERNAME,
    'Appid': APP_ID,    # ⚠️ 是 Appid 不是 app_id！
    'token': token,
    'Idstr': perf_id,   # performance_id
    # ⚠️ 不需要 env 参数！
}

resp = requests.post(
    f"{BASE_URL}/openapi/single/detail",
    json=params,
    headers={"Content-Type": "application/json"}
)

result = resp.json()
# 数据路径: result['data']['ret']
data = result['data']['ret']
```

### FPS 数据提取（仅供参考，模版中用 conditionalSearch 的 fps_mean）

```python
# FPS 曲线数据（仅用于画曲线图，不算平均值）
fps_data = data['fps_frame_data']['FpsData']
# 每个元素: {'Time': '0.00', 'Fps': 2.17, 'FrameIndex': [...], 'CpuInfo': {...}, 'GpuInfo': {...}, 'MemInfo': {...}}

# ⚠️ 模版中的平均FPS用 conditionalSearch 返回的 fps_mean，不要自己算！
# fps_mean 来自 records[0]['fps_mean']，是平台排除异常帧后算的
```

### 返回数据结构（关键字段）

```
data
├── scene: {Sname, mark, LaunchTime, EndTime, Tsp(时长秒), Ldt(加载耗时秒)}
├── fps_frame_data
│   ├── FpsData: [{Time, Fps, FrameIndex, CpuInfo, GpuInfo, MemInfo}, ...]  ← FPS曲线
│   ├── JankIndex: [{FpsIndex, Count}, ...]
│   ├── BigJankIndex: [...]
│   └── FpsSwingIndex: [...]
├── cpu: {arr, avg, top}
├── gpuUsage: {arr, avg, top}
├── workingSet: {arr, avg, top}  (MB)
├── onlineTime: int (秒)
├── seriousFrozen: int
├── Sochardware: string
└── ... (100+ 其他字段)
```

## 常见场景名

| 场景 | scene_name |
|------|-----------|
| 大世界 | `/Game/Maps/World/World_P.World_P` |
| 室内 | `/Game/Maps/WorldIndoor/...` (多个变体) |
| 洞穴 | `/Game/Maps/SeamLessCave/...` |
| 活动 | `/GF_EM_*/Maps/EventMap/...` |
| 初始化 | `PerfSight.Initialize` |

## 完整查询流程

1. `conditionalSearch` → 获取 `performance_id` 列表和**平台算好的指标**（fps_mean, fpsMin, janks 等）
2. 用**场景结束时间**标识对局（不用开始时间）
3. `detail(Idstr=performance_id)` → 获取 FPS 曲线（用于画图）和其他详细数据
4. 模版中所有指标取平台值，不自己算
5. FPS 曲线数据（`fps_frame_data.FpsData`）仅用于可视化，不参与指标计算

## 示例脚本参考

- 条件查询: https://perfsight-docs-1258344700.cos.ap-nanjing.myqcloud.com/open/PS_conditionalSearch.py
- 单局详情: https://perfsight-docs-1258344700.cos.ap-nanjing.myqcloud.com/open/PS_Detail.py
