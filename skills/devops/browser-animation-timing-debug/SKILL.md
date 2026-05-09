---
name: browser-animation-timing-debug
description: 排查浏览器动画速度异常 + CDN缓存导致的更新不生效问题
---

# Browser Animation Timing Debug

## Problem
动画速度比预期快很多，即使设置了很大的时间间隔/阈值也不生效。HTML/JS改了但浏览器看不到变化。

## Root Causes (两个往往同时存在)

### 1. Cloudflare CDN 缓存
Cloudflare 缓存了 HTML 响应，文件更新后浏览器仍然看到旧版本。

**修复：** 清除 CDN 缓存
- Cloudflare Dashboard → Caching → Configuration → **Purge Everything**
- 或用 API：`curl -X POST "https://api.cloudflare.com/client/v4/zones/{zone}/purge_cache" -H "Authorization: Bearer {token}" -H "Content-Type: application/json" -d '{"purge_everything":true}'`

### 2. 浏览器标签节流 + requestAnimationFrame
标签页在后台时，`requestAnimationFrame` 降到约 1fps，但 `performance.now()` 照常走。累加时间的做法会失败，因为切回前台时 `dt` 一下跳几千毫秒。

**错误示例（会中招）：**
```javascript
let curveAcc = 0;
function animate() {
  requestAnimationFrame(animate);
  const now = performance.now();
  const dt = now - lastT; // 切回标签时 dt 可能 = 3000+ ms
  lastT = now;
  curveAcc += dt;
  if (curveAcc >= 2000) { // 一次触发多次
    curveAcc = 0;
    // 更新曲线
  }
}
```

**正确做法 — 帧数驱动（推荐）：**
```javascript
let f = 0;
const CURVE_FRAMES = 150; // 每 N 帧更新一次

function animate() {
  requestAnimationFrame(animate);
  f++;
  if (f % CURVE_FRAMES === 0) {
    // 更新曲线
  }
  if (f % 20 === 0) {
    // 更新柱状图
  }
}
```
帧数驱动完全不依赖真实时间，标签节流不影响速度。

**备选 — 限制 dt 上限：**
```javascript
const dt = Math.min(Math.max(1, now - lastT), 50); // 每帧最多累加 50ms
lastT = now;
curveAcc += dt;
```

### 3. FPS 显示 infinity / NaN
`1000 / dt` 中 `dt = 0` 会导致 infinity。

**修复：** `const dt = Math.max(1, now - lastT)`

## 验证方法
始终用 `grep` 或 `read_file` 直接确认文件内容——浏览器/网络缓存可能让你误以为改动没生效。

## 测试时绕过缓存
用 query param 强制不走缓存：
```
https://example.com/page?t=1234567890
```
