---
title: "Vite 代理接口偶发超时的排查与解决"
date: 2026-06-04
tags: ["Vite", "Vue3"]
slug: "vite-proxy"
description: "记录一次本地 Vite 代理接口偶发超时的排查过程，以及通过关闭 keep-alive 连接复用解决问题的处理方式"
keywords: ["Vite proxy", "keep-alive", "Connection close"]
---

在开发时我发现这样的问题，我前端调用一个接口会造成接口返回异常直到这个接口超时。但是偶尔又是正常的。一开始因为是接口问题，查询速度慢，返回有问题。后面我去接口工具中调用接口发现很正常。但是唯独使用本地 vite 代理的接口就会出现这种情况。

我的 vite 代理是这样的

```js
proxy: {
  "^/csm-(auth|base|olcs|call|ors|stats|voice)": {
    target: "http://xxx.xxx.xxx.xx:xxx",
    changeOrigin: true,
    rewrite: path => path
  }
},
```

后面我发现可能是 vite中的代理出现了问题。一番查询之下，得出结论可能是 Vite 代理复用了某个后端或中间网络状态不好的连接。因为这部分我没有去深究是什么情况。后面我将这个 vite 代理单独把这个 csm-voice 这个规则重新写了一个。

```js
proxy: {
  "/csm-voice": {
    target: "http://xxxx.xxx.xxx.xxx:xxx",
    changeOrigin: true,
    rewrite: path => path,
    headers: {
      Connection: "close"
    },
    agent: new http.Agent({ keepAlive: false }),
    timeout: 120000,
    proxyTimeout: 120000
  },
  "^/csm-(auth|base|olcs|call|ors|stats)": {
    target: "http://xxxx.xxx.xxx.xxx:xxxx",
    changeOrigin: true,
    rewrite: path => path
  }
},
```

主要是新增了这样的规则，后面接口请求就正常返回了，没有说出现很慢的情况。

```js
headers: {
  Connection: "close"
},
agent: new http.Agent({ keepAlive: false }),
```

这两条规则的作用就是 关闭 keep-alive 连接复用
Connection: close 和 keepAlive: false 表示
每次请求都新建/关闭连接，不复用旧连接。所以每次请求都是新的干净的请求，就不会再出现那种情况了。
