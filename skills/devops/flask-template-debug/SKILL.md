---
name: flask-template-debug
description: Flask 模板调试技巧 — 缓存问题、Three.js 本地化、nav 居中、z-index 覆盖
---

# Flask 模板调试经验

## 模板缓存问题
Flask 默认会缓存模板文件，每次修改后需要**重启 Flask 服务**才能生效。
调试模板时流程：
```
1. 修改模板 → 保存
2. kill $(cat flask.pid) 停止服务
3. start.sh 重启
4. curl 验证文件已更新
5. 浏览器 Ctrl+F5 刷新
```

## Three.js 本地化
通过 Cloudflare Tunnel 访问时，外部 CDN（cdnjs.cloudflare.com）会报 `ERR_SSL_BAD_RECORD_MAC_ALERT`。
解决：下载到本地 `static/` 目录，引用改为 `/static/three.min.js`。

## nav 居中问题
Flexbox 居中：`justify-content: center` 要加在**具体那个 flex 容器**上。
- 父级 `.nav { justify-content: space-between; }` 默认两端对齐
- 子级 `.nav-links { justify-content: center; }` 才能让链接居中
- 两者同时加更保险

## nav z-index 被 3D canvas 覆盖
Three.js canvas 覆盖整个页面时，FPS 等 HTML 元素被挡住。
- 3D FPS sprite 的 z 轴位置再高也无效（canvas 2D 层 vs 3D 场景层）
- 解决方案：FPS 改用 HTML overlay，z-index 200+ 即可覆盖
