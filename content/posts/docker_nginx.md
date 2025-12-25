当然可以！以下是你原文内容的**格式优化版**——**完全保留所有文字内容**，仅对排版、结构、代码块和标题层级进行了规范化和美化，使其更清晰易读，同时保持技术细节一字不差。

---

# Docker 场景下 Nginx 部署 Vue 项目的实践

**日期**：2025-12-25  
**标签**：`nginx`, `docker`, `Vue`, `反向代理`  
**摘要**：在 Docker 中使用 Nginx 部署 Vue 前端，并同时代理 API 与图片等静态资源的完整实践，总结端口冲突、502 问题等  
**关键词**：`nginx`, `docker`, `vue`, `proxy`

---

## 背景

今天在公司有点时间，想要试试前端部署的一个完整流程，当然这只是一个简单的 docker+nginx 部署流程。

- **前端**：Vue build 后的 dist 静态文件  
- **后端接口**：运行在 `172.xx.xx.xx:66xx`  
- **图片资源路径**：`/uploads/**`  
- **对外访问端口**：不占用现有服务，选择一个不常用端口  

---

## 运行环境说明

```
宿主机
└── Docker
    └── nginx 容器
        ├── 提供 Vue 静态页面
        ├── 反向代理 API
        └── 反向代理图片资源
```

---

## 部署环境

### 一、基础准备（Docker 环境）

#### 1. 确认 Docker 是否安装

```bash
docker -v
```

输出类似：

```
Docker version 27.x.x
```

说明 Docker 已可用。

---

#### 2. 确认 Docker Compose（v2）

```bash
docker compose version
```

若能正常输出版本号，即可继续。

---

### 二、创建项目目录结构

```bash
mkdir -p /data/xiaoss/qywx
cd /data/xiaoss/qywx
```

最终目录规划如下：

```
qywx/
├── docker-compose.yml
├── html/
│   └── qywx_fe/        # Vue build 后的文件
├── conf/
│   └── nginx.conf
└── logs/
```

---

### 三、创建 `docker-compose.yml`（核心）

#### 1. 新建文件

```bash
nano docker-compose.yml
```

#### 2. 编写内容

```yaml
version: "3.9"

services:
  nginx:
    image: nginx:latest
    container_name: nginx_qywx
    ports:
      - "9110:80"
    volumes:
      - ./html:/usr/share/nginx/html
      - ./conf/nginx.conf:/etc/nginx/nginx.conf
      - ./logs:/var/log/nginx
    restart: unless-stopped
```

**说明**：
- `image: nginx:latest` → 自动从 Docker Hub 拉取官方 nginx 镜像  
- `9110:80` → 宿主机 9110 → 容器 80  
- `volumes` → 挂载前端、配置、日志目录  

---

### 四、拉取 Nginx 并启动容器

#### 1. 启动（首次会自动拉镜像）

```bash
docker compose up -d
```

等价于：
- `docker pull nginx:latest`
- 创建容器
- 后台运行

---

#### 2. 查看运行状态

```bash
docker ps
```

应看到：

```
nginx_qywx   nginx:latest   Up   0.0.0.0:9110->80/tcp
```

---

### 五、准备 Vue 前端静态文件

将 Vue 打包产物放入：

```
/data/xiaoss/qywx/html/qywx_fe
```

例如：

```
qywx_fe/
├── index.html
├── js/
├── css/
└── assets/
```

---

### 六、Nginx 配置（关键部分）

#### 1. 创建 `nginx.conf`

```bash
mkdir -p conf
nano conf/nginx.conf
```

#### 2. 最终配置内容

```nginx
worker_processes 1;

events { worker_connections 1024; }

http {
    include       mime.types;
    default_type  application/octet-stream;

    add_header Access-Control-Allow-Origin *;
    add_header Access-Control-Allow-Methods "GET, POST, OPTIONS";
    add_header Access-Control-Allow-Headers "Authorization,Origin, X-Requested-With, Content-Type, Accept";

    sendfile on;
    keepalive_timeout 65;

    server {
        listen 80;
        server_name _;

        # Vue 前端
        root /usr/share/nginx/html/qywx_fe;
        index index.html;

        location / {
            try_files $uri $uri/ /index.html;
        }

        # 图片资源代理
        location /uploads/ {
            proxy_pass http://172.xx.xx.xx:66xx;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
            proxy_read_timeout 60s;
        }

        # API 代理
        location /api {
            proxy_pass http://172.xx.xx.xx:66xx;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
            proxy_connect_timeout 60s;
            proxy_send_timeout 60s;
            proxy_read_timeout 60s;
        }
    }
}
```

---

### 七、重启容器使配置生效

```bash
docker compose down
docker compose up -d
```

---

## 问题与排查过程

### 1. 端口冲突问题

最开始使用的时候选的默认 80 端口，但是服务器中已经有使用的了：

```yaml
ports:
  - "80:80"
```

直接报错：`bind: address already in use`

**排查后发现**：
- 宿主机已有系统 nginx 占用 80  
- 多个 docker-proxy 已占用 8000+ 端口  

**后续选择更换端口**，随便找了一个没用过的端口号：

```yaml
ports:
  - "9110:80"
```

---

### 2. 当我一切准备完毕后，发现项目中需要匹配域名端口的 `/uploads/**` 图片无法访问

在前端代码中使用的是：

```
/uploads/excellentDisplay/good2.jpg
```

打包并部署后发现：
- 页面正常  
- 图片访问失败或返回 HTML  

**经过 AI 的原因分析是因为**：

Vue 使用了 history 模式，nginx 中有：

```nginx
location / {
    try_files $uri $uri/ /index.html;
}
```

该配置会：
- 吞掉所有未被单独匹配的路径  
- `/uploads/**` 被当成前端路由处理  
- fallback 到 `index.html`  

---

### 关键解决方案

#### 1. 必须为 `/uploads` 单独配置 `location`

这是整个问题中 **最关键的一点**。

```nginx
location /uploads/ {
    proxy_pass http://172.16.99.32:6690;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_read_timeout 60s;
}
```

---

### 2. 完整 Nginx 配置（最终版）

```nginx
worker_processes 1;

events { worker_connections 1024; }

http {
    include       mime.types;
    default_type  application/octet-stream;

    add_header Access-Control-Allow-Origin *;
    add_header Access-Control-Allow-Methods "GET, POST, OPTIONS";
    add_header Access-Control-Allow-Headers "Authorization,Origin, X-Requested-With, Content-Type, Accept";

    sendfile on;
    keepalive_timeout 65;

    server {
        listen 80;
        server_name _;

        # Vue 前端
        root /usr/share/nginx/html/qywx_fe;
        index index.html;

        location / {
            try_files $uri $uri/ /index.html;
        }

        # 图片资源代理
        location /uploads/ {
            proxy_pass http://172.16.99.32:6690;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
            proxy_read_timeout 60s;
        }

        # API 代理
        location /api {
            proxy_pass http://172.16.99.32:6690;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
            proxy_connect_timeout 60s;
            proxy_send_timeout 60s;
            proxy_read_timeout 60s;
        }
    }
}
```

---

### Docker Compose 配置（最终版）

```yaml
version: "3.9"

services:
  nginx:
    image: nginx:latest
    container_name: nginx_qywx
    ports:
      - "9110:80"
    volumes:
      - ./html:/usr/share/nginx/html
      - ./conf/nginx.conf:/etc/nginx/nginx.conf
      - ./logs:/var/log/nginx
    restart: unless-stopped
```

---

### 日志处理

在最后我发现 nginx 是没有日志输出，不好判断问题的情况，于是我加上 nginx 的日志输出。

为了便于排查 502、代理错误等问题：
- 将容器内日志目录映射到宿主机  
- 直接查看真实 `access.log` / `error.log`

```yaml
- ./logs:/var/log/nginx
```

最后经过一系列操作完成部署流程。

---

## 关键经验总结

1. **Docker 场景下需要判断端口的占用情况**，尤其是在服务器中可能存在多个 docker 的情况下。  
2. **Vue history 模式一定要注意 `try_files` 的副作用**  
3. **`location` 的顺序非常重要**  
4. **日志一定要持久化，否则排错非常痛苦**  
5. **页面正常 ≠ 资源与接口正常**，有可能接口资源出现网络 502 错误导致不正常  

---