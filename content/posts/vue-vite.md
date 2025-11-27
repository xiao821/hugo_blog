---
title: "关于 Vite 做跨域代理的那些事"
date: 2025-11-27
tags: ["vue.js", "Vite", "Proxy代理"]
slug: "vue-vite"
description: 在开发环境下使用 Vite 做跨域代理的配置及原理 
keywords: ["Vue3","Vite"]
---

# 使用 Vite 的 Proxy / WebSocket 代理：配置 + 原理 + 源码机制

当我们在开发 Web 应用时，我们经常遇到这样的问题：

- 前端代码运行在本地（如 `http://localhost:5173`，由 Vite dev server 提供），而后端 API 或 WebSocket 服务部署在另一个域名／端口（如 `http://api.example.com`，或 `ws://backend:8080`），而面对这种问题，我们往往需要解决跨域问题，要么就是后端设置允许跨域，要么就是在本地启动 nginx。这样都通常还是比较麻烦。但是有了 Vite 之后，只需要简单配置一下即可解决，这是非常神奇的，那么这个原理与配置是要怎么做呢？

- 首先是为什么会有跨域问题？跨域是由于浏览器的同源策略 (Same-Origin Policy) 会阻止不同源的的 HTTP 请求或 WebSocket 连接，从而导致请求失败或 WebSocket 无法连接。

而为了绕过这个限制，并简化开发环境配置，希望由“前端 → 统一 origin (localhost) → 后端 (API / WebSocket)” —— 即通过一个 **代理 (proxy)** 中间层来转发请求。

`http-proxy`（Node.js 上的 HTTP / WebSocket 代理库）能做到这个功能，而且支持 HTTP 和 WebSocket 协议。  
而 Vite 本身就内置了 proxy 功能（通过 `server.proxy` 配置），并且支持 WebSocket 转发／代理。

因此，我们可以通过 Vite 做类似于生产环境中 Nginx + reverse-proxy + WebSocket 转发那样的事情 —— 但这只仅限于开发环境 (development) 使用。生产环境还是需要同步配置 nginx 来做转发的。

---

## Vite Proxy 的配置与基本使用

### 基本配置

在 `vite.config.js`（或 `.ts`）中，你可以这样配置：

```js
export default defineConfig({
  server: {
    proxy: {
      '/api': {
        target: 'http://backend.example.com',
        changeOrigin: true,
        rewrite: (path) => path.replace(/^\/api/, ''),
      },
      '/socket': {
        target: 'ws://backend.example.com',
        ws: true,
        changeOrigin: true,
      }
    }
  }
});
```

- `target`：目标服务器地址 (HTTP / HTTPS / WS / WSS)  
- `changeOrigin`：是否将请求头中的 `Host` / `Origin` 等修改为目标服务器所属 origin（有助于避免后端因为 origin 校验拒绝请求）  
- `rewrite` (可选)：重写路径，如去掉前端使用的路径前缀 `/api`、`/socket` 等，让后端接收到原始期望路径。  
- `ws: true` — `ws` 为 `true` 是维持 websocket 长连接的一个重要字段。

---

## 原理

那为什么 Vite 能像 Nginx 那样支持 HTTP + WebSocket 呢？

以下是我查到的对 Vite 的了解：

### Vite 的本质：一个开发服务器 (Dev Server)

- Vite 是一个真正的本地开发服务器 (development server)，它监听某个端口（默认 5173），响应客户端 (浏览器) 的请求。  
- 它不仅负责静态资源 (HTML / JS / CSS / 模块加载 / HMR)，也可以充当一个中间层 (proxy)，把部分请求转发到后端 (HTTP API 或 WebSocket 服务)。  

实际上，Vite 本身就是一个 “服务器 + 可配置中间件 (middleware)” 的框架 — 所以它可以像 Nginx / Apache / lighttpd 那样做反向代理 (reverse proxy)。

### 内部使用 http-proxy，支持 HTTP & WebSocket 转发

- Vite 的 proxy 功能是基于 `http-proxy` — 一个成熟且支持 HTTP + WebSocket (upgrade/ws/wss) 的代理库。  
- 当浏览器发起普通 HTTP 请求 (fetch / xhr / ajax)，Vite proxy 会将其转发 (`proxy.web`) 给 `target`；
- 当浏览器发起 WebSocket 请求 (带 `Upgrade: websocket` / `Connection: upgrade` header)，Vite proxy 会识别出这是 WebSocket 升级请求 (upgrade)，通过 `proxy.ws` 转发 — 从而建立 WebSocket 连接 (双向 / 长连接) —— 后续客户端与后端之间的数据帧 (frames) 会被 “透明转发 (tunnel)”。

---

## 源码

从源码／机制角度看 Vite 的 Proxy + WebSocket 支持：

```js
on server start:
  if server.proxy 配了规则:
    for each (pathPattern → proxyOptions):
      create 一个 http-proxy 实例 (proxy) with options { target, changeOrigin, ws, secure, ... }
      register middleware:
        app.use(pathPattern, (req, res, next) => {
          // 如果普通 HTTP request:
          proxy.web(req, res, options)
        })
      listen upgrade 事件:
        server.on('upgrade', (req, socket, head) => {
          if req.url matches pathPattern:
            proxy.ws(req, socket, head, options)
        })
```

---

## 总结

- Vite 本身是一个开发服务器 (dev server)，不仅负责静态资源 + HMR，还支持通过 `server.proxy` 配置做 HTTP / WebSocket 代理。  
- Vite 的 proxy 功能基于 `http-proxy`，实现对 HTTP 和 WebSocket (upgrade/ws/wss) 的请求转发。

功力浅薄，可能理解不是很深，写的也不够深入，敬请见谅，感谢观看