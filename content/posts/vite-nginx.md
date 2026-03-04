---
title: "Nginx 反向代理导致上游 Vite 服务 403 的排查与解决"
date: 2026-03-04
tags: ["Nginx", "Vite", "Vue3", "部署"]
slug: "vite-nginx"
description: "记录一次 Nginx 反向代理因 proxy_set_header Host 配置不当，触发上游 Vite 服务 allowedHosts 校验导致 403 的排查过程与解决方案"
keywords: ["Nginx 反向代理", "Vite allowedHosts", "proxy_set_header Host", "Nginx 403"]
---

## 背景

昨天在公司写完项目后，本来就是一个很正常的新增一个接口，然后修改 nginx，然后部署项目，但是出现了一个非常奇怪的事情。我的接口居然出现 403，报错还是 vite ???? 可是我已经部署了怎么可能还报错 vite 的问题呢????

## Vite 跨域配置

以下是我的 vue3 代码的 vite的跨域配置，其实就是端口号不一样，所以我就参考了之前写的 Botapi，nginx 也是这样写的。

```js
"/Botapi/": {
    target: "http://111.222.333.444:9088",
    changeOrigin: true,
    rewrite: path => path.replace(/^\/Botapi\//, ""),
},
"/Formapi/": {
    target: "http://111.222.333.444:3000",
    changeOrigin: true,
    rewrite: path => path.replace(/^\/Formapi\//, ""),
},
```

## 问题出现

可是就是因为这样，我以为没啥问题，但是在部署完测试后，发现这个 Formapi 接口莫名其妙出现了 403 错误。
错误信息为

```text
Blocked request. This host ("dev209988.szmckj.cn") is not allowed.
To allow this host, add "dev209988.szmckj.cn" to `server.allowedHosts` in vite.config.js.
```

## 排查过程

### 第一次尝试：修改 Vite 配置

我百思不得其解，为什么会这样呢？明明都是一样的逻辑，但是为什么会出现这样的问题，于是我开始排查原因，首先就是看到它说我的 vite 有问题，但是我都是打包 build 后上传到服务器上了，哪来的 vite 事情啊？但是我出于谨慎的情况，将我的 vite 配置给到了个 GPT ，经过一通分析，说需要我在 vite 中在 server 中新增 allowedHosts 为 true 或者写需要映射到外网的域名。

```js
server: {
    // 端口号
    port: VITE_PORT,
    host: "0.0.0.0",
    allowedHosts: ["xyz.abc.cn"],
}
```

我按照这个逻辑写上了，但是还是有问题，我继续搜索。后面与同事一起讨论。应该可能是 nginx 代理出现了问题。

### 第二次尝试：排查 Nginx 配置

我原本的 nginx 是这样写的

```nginx
location /Formapi/{
    root   html;
    proxy_pass http://111.222.333.444:3000/;
    proxy_http_version 1.1;
    proxy_connect_timeout 4s;
    proxy_read_timeout 60s;
    proxy_send_timeout 12s;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "Upgrade";
    proxy_set_header Host $host:$server_port;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
}
```

发现是我的 nginx 中的`proxy_set_header Host $host:$server_port;`这个写法有问题，GPT 给出的解释是这个报错是上游 `http://111.222.333.444:3000/` 做了 Host 校验拦下来的；`pnpm build` 出来的 dist 静态文件托管本身是不受 Vite `allowedHosts` 影响，这点也验证了我本身的想法是没错的，打包后是不受到 Vite 的影响。

## 解决方案

后面把 `/Formapi/` 这一段改成让上游看到 IP Host 就能绕过：

```nginx
location /Formapi/ {
    proxy_pass http://111.222.333.444:3000/;
    proxy_http_version 1.1;

    proxy_connect_timeout 4s;

    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "Upgrade";

    # 关键：别再传 $host:$server_port，改传上游 host（111.222.333.444:3000）
    proxy_set_header Host $proxy_host;

    # 可选：保留原始域名给上游（如果它需要用）
    proxy_set_header X-Forwarded-Host $host;

    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
}
```

## 原理分析

对于上述的 nginx 的解释如下（参考 GPT 老师给的解释，AI 上大分）：
Vite 报这个错时，它检查的是"到达它那一跳的 HTTP Host 请求头"，不是你浏览器地址栏看到的域名。

### 为什么原来的配置会 403

这行把 外网映射的域名 强行塞给了上游：`proxy_set_header Host $host:$server_port;`

- `$host` 来自客户端的 Host（xyz.abc.cn）
- `$server_port` 是这台 Nginx 接收到请求的端口（我这里是 9988，不一定是外网的 443）
- 所以上游实际收到的是类似：`Host: xyz.abc.cn:9988`

`但是请注意这里的细节情况，因为我这个接口是非本公司提供的，不清楚那边是什么情况`，这边猜测这个上游 `111.222.333.444:3000` 跑的是 Vite（dev/preview），它有个防 DNS rebinding 的中间件，所以会按 `req.headers.host` 做白名单校验；不在 `allowedHosts` 里就直接 403。

### 为什么改成 `$proxy_host` 就好了

- `proxy_set_header Host $proxy_host;` 会让上游收到：`Host: 111.222.333.444:3000`
- Vite 的 Host 校验对 IP 是默认放行的（它认为用局域网 IP 访问 dev server 是正常场景），所以不会再拦截

### `X-Forwarded-Host` 的作用

- 改完以后，上游看到的 Host 变成 IP 了，如果上游业务还想知道用户最初访问的是哪个域名，就用 `X-Forwarded-Host: xyz.abc.cn` 传过去
- 这个头不会影响 Vite 的拦截逻辑（Vite只看 Host），只是给上游应用做参考
