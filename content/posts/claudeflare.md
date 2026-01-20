---
title: "Cloudflare Pages + GitHub 部署静态站点实战"
date: 2026-01-20
tags: ["Cloudflare", "GitHub", "静态页面部署"]
slug: "claudeflare"
description: 本文演示如何在 Cloudflare Pages 结合 GitHub 部署静态站点，并处理构建配置细节。
keywords: ["Cloudflare Pages", "Cloudflare Workers", "GitHub 部署", "静态网站"]
---

之前一直想写一篇 Cloudflare Pages 部署静态站点的记录，今天正好用一个 Vue 小项目做示例，把过程整理下来。

## 1. 登录 Cloudflare，进入 Workers & Pages

在侧栏 **Compute & AI → Workers & Pages**。

![image.png](https://xssblog.211508.xyz/2026/01/4a1438a2195238a93c6a19fa87ced6e7.png)

## 2. 创建应用

![image.png](https://xssblog.211508.xyz/2026/01/44bb09f9df835d84f0267dee9c71eafe.png)

## 3. 绑定 GitHub 仓库

![image.png](https://xssblog.211508.xyz/2026/01/81b79d395870e31c3683fda86f44c12f.png)

![image.png](https://xssblog.211508.xyz/2026/01/5cadf00bf16dedf6a19e924976636a40.png)

## 4. 设置构建命令

![image.png](https://xssblog.211508.xyz/2026/01/1a18df57370e06eacd3818d497403f27.png)

Vue 项目需要 `npm run build`，纯静态站点可跳过。

### 关键点：Deploy command

若 Deploy command 填了 `npx wrangler deploy`，需在项目根目录添加 `wrangler.jsonc`（或 `wrangler.toml`），示例：

```
{
  "name": "mini-projects-hub",
  "compatibility_date": "2026-01-20",
  "assets": {
    "directory": "./dist"
  }
}
```

否则构建阶段会报错。补上后重新构建即可。

![image.png](https://xssblog.211508.xyz/2026/01/8a8f3c444a2bfe5b888e47509fcd336b.png)

## 5. 获取页面地址

![image.png](https://xssblog.211508.xyz/2026/01/1c5c655e41b2853161c6666c742c9162.png)

## 6. 绑定自定义域名

在项目设置页点击 **Add → Custom domain**，输入自己的域名即可完成。

![image.png](https://xssblog.211508.xyz/2026/01/d481bb9c41e94db851f5c433526078d8.png)
![image.png](https://xssblog.211508.xyz/2026/01/a836e683feda7b01f28c6463034fc50d.png)
![image.png](https://xssblog.211508.xyz/2026/01/d481bb9c41e94db851f5c433526078d8.png)


### 最后输出
- 新手记录，如有疏漏欢迎留言交流。谢谢！

