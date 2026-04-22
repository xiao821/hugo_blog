---
title: "Nuxt.js 上手学习笔记（二）：项目部署、Nginx 反向代理与 SQLite 排查"
date: 2026-04-22
tags: ["Nuxt", "Docker", "Nginx", "SQLite"]
slug: "Nuxt-2"
description: "记录 Nuxt 项目首次部署的复盘过程，包括 Docker 启动、宿主机 Nginx 反向代理、Cloudflare 访问链路，以及 better-sqlite3 的排查与解决。"
keywords: ["Nuxt 部署", "Docker", "Nginx 反向代理", "better-sqlite3"]
---

## Nuxt.js （day2） 的学习

前一天刚把 Nuxt 的页面、接口和 SQLite 跑起来，今天就顺势开始折腾部署。因为这个项目本身已经具备前后端和数据库的最小闭环，所以很适合拿来练习一次完整的上线流程。

这篇主要记录我在部署 Nuxt 项目时最终采用的方案、实际踩过的坑，以及排查 `Nginx`、`Cloudflare` 和 `better-sqlite3` 过程中得到的一些结论。整体折腾了小半天，最后算是把从容器启动到域名访问这一整条链路搞通了。感谢 GPT,感谢我的同事哈哈！

最终我决定统一采用这套方案：

- Docker 里只运行 `Nuxt`
- 宿主机 Nginx 负责对外入口
- 宿主机 Nginx 反向代理到 `127.0.0.1:3000`

## 一、我最终采用的部署方式

最后确定使用的是这套部署流程：

```text
浏览器
  -> Cloudflare
  -> 服务器公网 80/443
  -> 宿主机 Nginx
  -> 127.0.0.1:3000
  -> Docker 里的 Nuxt
  -> SQLite
```

也就是说：

- Docker 里只运行 Nuxt 应用
- Nuxt 暴露 `3000` 端口给宿主机
- 宿主机 Nginx 监听 `80/443`
- 宿主机 Nginx 再把请求转发到 `127.0.0.1:3000`
- SQLite 由 Nuxt 容器使用，并通过 volume 持久化

## 二、我的部署流程

### 1. 服务器准备

服务器上需要有：

- Docker
- Docker Compose 插件
- Nginx

对外通常只开放：

- `80`
- `443`

应用内部端口使用：

- `3000`

`3000` 只是给宿主机 Nginx 转发使用，不建议直接作为公网入口。

### 2. 上传项目

我把 Nuxt 项目和相关部署文件上传到服务器，例如：

```text
~/nuxt/nuxt-demo
```

这次部署中比较关键的文件有：

- [Dockerfile](/Users/xiao/vue-node-nginx-docker/nuxt/nuxt-demo/Dockerfile:1)
- [docker-compose.yml](/Users/xiao/vue-node-nginx-docker/nuxt/nuxt-demo/docker-compose.yml:1)
- [server/utils/db.ts](/Users/xiao/vue-node-nginx-docker/nuxt/nuxt-demo/server/utils/db.ts:1)
- 宿主机上的 `/etc/nginx/sites-available/nuxt.conf`

### 3. 启动 Docker 中的 Nuxt

进入项目目录后执行：

```bash
sudo docker compose down
sudo docker compose up -d --build
```

当前最终使用的 `docker-compose.yml` 核心思路是：

```yaml
services:
  nuxt:
    build: .
    restart: always
    ports:
      - "127.0.0.1:3000:3000"
    environment:
      - NODE_ENV=production
      - JWT_SECRET=your-secret-change-this
      - DB_PATH=/app/data/data.db
    volumes:
      - ./data:/app/data
```

这里的作用分别是：

- `127.0.0.1:3000:3000`：只把容器的 `3000` 暴露给宿主机本机，不直接暴露到公网
- `DB_PATH=/app/data/data.db`：让 SQLite 存到持久化目录
- `./data:/app/data`：把数据库文件保存到宿主机

### 4. 先验证容器本身是否正常

不要一上来就只看域名，先确认 Nuxt 本身在服务器中已经正常工作。

先执行：

```bash
sudo docker compose ps
sudo docker compose logs --tail=100 nuxt
curl -I http://127.0.0.1:3000
```

这一步如果返回 `200 OK`，说明：

- Docker 正常
- Nuxt 正常
- 应用端口正常

只有这一步通过之后，再去检查 Nginx、域名和 Cloudflare。

### 5. 配置宿主机 Nginx

因为最终采用的是“宿主机 Nginx + Docker Nuxt”，所以宿主机 Nginx 应该写成：

```nginx
server {
    listen 80;
    server_name xxxx.yyyy.xyz;

    location / {
        proxy_pass http://127.0.0.1:3000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

修改完宿主机的 `/etc/nginx/sites-available/nuxt.conf` 后执行：

```bash
sudo nginx -t
sudo systemctl reload nginx
```

### 6. 最后再看域名和 Cloudflare

等 Nuxt 和宿主机 Nginx 都通了，再访问域名：

```text
http://xxxx.yyyy.xyz
```

如果前面都正常，但域名还是不通，就继续检查：

- 域名解析
- Cloudflare 代理状态
- 服务器防火墙
- 云服务器安全组

## 三、试错过程

这次部署过程中，主要踩了下面几个坑。

### 1. `Dockerfile` 里错误执行了 `pnpm init`

一开始镜像构建失败，报错内容是：

```text
ERR_PNPM_PACKAGE_JSON_EXISTS
```

当时 `Dockerfile` 里有这一步：

```dockerfile
RUN cd .output/server && pnpm init && pnpm add better-sqlite3
```

问题在于：

- `pnpm build` 后 `.output/server` 里已经有 `package.json`
- 再执行 `pnpm init` 一定会报错

这个问题后来通过删除 `pnpm init` 解决。

### 2. 容器名冲突

后面又遇到了这种错误：

```text
The container name "/nuxt-demo" is already in use
```

原因是：

- 服务器上已经有旧容器
- 配置里用了固定容器名

解决方式：

- 删除旧容器
- 或者不再写固定 `container_name`

### 3. 一度把“宿主机 Nginx”和“Docker 里的 Nginx”混用了

这次最容易绕晕的地方就在这里。

前面曾经出现过这种写法：

```nginx
proxy_pass http://nuxt:3000;
```

它更像是“Docker 里的 Nginx -> Docker 里的 Nuxt”的用法。

但当前项目实际的 `docker-compose.yml` 只有 `nuxt` 服务，没有 `nginx` 服务，所以最终应该统一成：

```nginx
proxy_pass http://127.0.0.1:3000;
```

这样配置和实际架构才一致。

### 4. Cloudflare 小黄云开启后访问失败

前面我曾经用过：

```text
http://xxxx.yyyy.xyz:8081/
```

这在灰云模式下有可能通，但在黄云模式下就不稳定甚至不通。后面通过同事的指导，把小黄云关掉了。

原因是：

- 黄云模式下，请求会先进入 Cloudflare
- Cloudflare 更适合标准 Web 端口
- `8081` 这种调试端口不适合作为最终生产入口

最终思路改成：

- 对外只走 `80/443`
- 由 Nginx 再转发到内部的 `3000`

### 5. SQLite 运行时报 `Cannot find module 'better-sqlite3'`

这次最关键的运行时错误以及报错最多的就是：

```text
Error: Cannot find module 'better-sqlite3'
```

一开始尝试过在 Docker 构建阶段额外安装：

```dockerfile
RUN npm install --prefix .output/server better-sqlite3
```

但是这条路并不稳，原因有两个：

- 构建阶段还要联网访问 npm registry
- 服务器上实际出现过 `ECONNRESET`

所以最后没有采用“在 `.output/server` 里额外联网安装依赖”的方案。

## 四、最终解决方法

### 1. `Dockerfile` 保持最小化

最终的 [Dockerfile](/Users/xiao/vue-node-nginx-docker/nuxt/nuxt-demo/Dockerfile:1) 只做这些事：

- 安装构建依赖
- 执行 `pnpm install`
- 执行 `pnpm build`
- 启动 Nuxt

核心内容如下：

```dockerfile
FROM node:22-alpine

RUN apk add --no-cache python3 make g++
RUN npm install -g pnpm

WORKDIR /app

COPY package.json pnpm-lock.yaml pnpm-workspace.yaml ./
RUN pnpm install --frozen-lockfile

COPY . .
RUN pnpm build

EXPOSE 3000
ENV NODE_ENV=production

CMD ["node", ".output/server/index.mjs"]
```

### 2. SQLite 不再在 `.output/server` 里额外安装依赖

最终真正解决 SQLite 问题的方式，不是继续在 `.output/server` 中强行装包，而是修改模块解析方式。

当前 [server/utils/db.ts](/Users/xiao/vue-node-nginx-docker/nuxt/nuxt-demo/server/utils/db.ts:1) 中的关键代码是：

```ts
const appRequire = createRequire(join(process.cwd(), 'package.json'))
const Database = appRequire('better-sqlite3')
```

这段代码的作用是：

- 不从 `.output` 的运行时入口位置去解析模块
- 而是强制从应用根目录 `/app` 去解析 `better-sqlite3`

因为容器里的 `/app/node_modules` 已经在 `pnpm install` 时装好了 `better-sqlite3`，所以运行时就能正常找到它。

### 3. SQLite 数据落到挂载目录

通过：

```text
DB_PATH=/app/data/data.db
```

和：

```yaml
volumes:
  - ./data:/app/data
```

保证数据库文件保存到宿主机目录中，这样容器重建后数据不会丢。

### 4. 先通容器，再通 Nginx，最后通域名

排查的正确顺序应该是：

1. Nuxt 容器能否启动
2. `better-sqlite3` 是否正常加载
3. `127.0.0.1:3000` 是否能正常访问
4. 宿主机 Nginx 是否代理成功
5. 域名和 Cloudflare 是否正常

如果跳过前面的检查，直接盯着域名看，很容易发生判断的错误点不一致。

## 五、疑问点整理

### 1. 服务器是不是一般只开放 `80` 和 `443`

通常是这样：

- 公网只开放 `80` 和 `443`
- 业务应用跑在 `3000`、`8080` 之类的内部端口
- 由 Nginx 把公网请求转发到这些内部端口

所以不是“监听 `80/443` 再做端口映射”，而是：

- Nginx 对外监听 `80/443`
- Nginx 再做反向代理

### 2. `proxy_pass http://127.0.0.1:3000;` 为什么更适合当前项目

因为当前项目的 `docker-compose.yml` 只有一个 `nuxt` 服务：

```yaml
services:
  nuxt:
    ...
```

没有 `nginx` 服务，所以当前架构不是“Docker 里的 Nginx -> Docker 里的 Nuxt”，而是“宿主机 Nginx -> Docker 里的 Nuxt”。

在这种结构下，更合理的写法就是：

```nginx
proxy_pass http://127.0.0.1:3000;
```

### 3. 为什么 `location /` 能把整站交给 Nuxt 处理

这里的 `/` 不是“首页”，而是“路径前缀的默认匹配规则”。

例如：

- `/`
- `/admin`
- `/api/users`

这些路径都以 `/` 开头，所以都会命中：

```nginx
location / {
    proxy_pass http://127.0.0.1:3000;
}
```

这条规则更准确的意思是：

- 只要没有更具体的 `location`
- 这个站点的大多数请求都走这一条规则

而 Nuxt 会自己处理页面路由，所以 Nginx 不需要再一个个区分 `/admin`、`/users`。

### 4. 为什么有些示例会写 `server_name xxxx.yyyy.xyz _;`

这里真正有意义的是：

```nginx
server_name xxxx.yyyy.xyz;
```

后面的 `_` 只是很多示例里的占位写法，不是必须的。

### 5. 为什么 Cloudflare 小黄云关掉后能访问，打开后反而不行

因为：

- 灰云是浏览器直接访问服务器
- 黄云是先访问 Cloudflare，再由 Cloudflare 回源

最终的正确思路是：

- 域名入口使用 `80/443`
- Cloudflare 代理标准 Web 入口
- Nginx 再转发到内部 `3000`

### 6. 这次 SQLite 报错最终到底是怎么解决的

最终不是靠“重新安装包”解决的，而是靠“更新运行时模块解析位置”解决的。

也就是把：

```ts
createRequire(import.meta.url)
```

改成：

```ts
createRequire(join(process.cwd(), 'package.json'))
```

这样运行时会从应用根目录 `/app/node_modules` 去找 `better-sqlite3`，于是就能找到。

## 六、最终效果

最后统一一下最终的部署方案是：

- Docker 只跑 `Nuxt`
- 宿主机 Nginx 负责公网入口
- 宿主机 Nginx 通过 `proxy_pass http://127.0.0.1:3000;` 转发到 Nuxt
- 域名入口只使用 `80/443`
- Cloudflare 放在最前面
- SQLite 通过 volume 做持久化

## 七、自我感悟
因为我确实不是很懂 docker 和整个部署流程，因为我也是想要学习整个系统开发和部署，所以借用同事的服务器做练手，顺便也是学习 nuxt 和部署的内容。在这里非常非常感谢 GPT 的帮助，真的大部分都是 GPT 帮我解决的。还有一个非常感谢的就是同事，确实经验老道一眼就帮我打通了最后的链接流程。最后还是需要多实践才能成长！！！后面继续学习！！！
