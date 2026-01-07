---
title: "Nginx 中 proxy_pass 路径配置的写法详解"
date: 2026-01-07
tags: ["nginx", "proxy_pass", "反向代理"]
slug: "nginx-proxy-pass-path-configurations"
description: 详解 Nginx 中 location 与 proxy_pass 的路径匹配规则，特别是如何正确去掉前缀路径（如 /kb）转发到后端服务。
keywords: ["nginx", "proxy_pass", "location", "反向代理", "路径重写"]
---

# Nginx 中 proxy_pass 路径配置的四种写法的区别

今天在服务器上做接口代理时，感觉完全迷糊了，我前端接口是这样的 `https://example.com/kb/xxx`，然后我在 nginx 中这样配置。

```nginx
location /kb/ {
    proxy_pass http://172.16.99.20:8070/kb/;
}
```

但是最后结果是转发错误，我以为这样配置是可以不携带/kb 的，于是进行调整，为了方便理解，我做了四个不同方法的测试。


## 需求场景

我希望是这样访问：

- 访问 `https://example.com/kb/xxx`
- Nginx 将请求代理到 `http://172.16.99.20:8070/xxx`
- **后端服务不要看到 `/kb`**

要实现这一点，关键在于 `location` 的匹配方式与 `proxy_pass` 是否以 `/` 结尾。

---

## 四种配置对比

### 配置一：保留 `/kb` 前缀（错误）

```nginx
location /kb/ {
    proxy_pass http://172.16.99.20:8070/kb/;
}
```

- 请求：`/kb/abc`
- 转发：`http://172.16.99.20:8070/kb/abc`
- **问题**：后端仍然会收到 `/kb`，不符合需求。

> 这种写法适用于后端服务本身也以 `/kb` 为根路径的场景，但不适用于我们目前想要剥离前缀的情况。

---

### 配置二：正确剥离前缀（正确）

```nginx
location /kb/ {
    proxy_pass http://172.16.99.20:8070/;
}
```

- 请求：`/kb/abc`
- 转发：`http://172.16.99.20:8070/abc`
- **效果**：`/kb/` 被成功移除，仅将剩余路径传递给后端。

> ✅ 这正是我们想要的行为。

> **原理**：当 `location` 以 `/kb/` 匹配（带尾斜杠），且 `proxy_pass` 以 `/` 结尾时，Nginx 会自动将匹配前缀以外的部分拼接到 `proxy_pass` 后。

---

### 配置三：未以 `/` 结尾，路径不重写 (错误)

```nginx
location /kb/ {
    proxy_pass http://172.16.99.20:8070;
}
```

- 请求：`/kb/abc`
- 转发：`http://172.16.99.20:8070/kb/abc`
- **问题**：尽管 `location` 是 `/kb/`，但 `proxy_pass` 没有尾部 `/`，Nginx 不会进行路径重写。

> 需要注意注意：`proxy_pass http://...`（无尾 `/`）表示使用原始 URI，不触发前缀替换。

---

### 配置四：location 不带尾斜杠，匹配范围过宽 (不太正确)

```nginx
location /kb {
    proxy_pass http://172.16.99.20:8070/;
}
```

- 行为：路径重写正确（`/kb/abc` → `/abc`）
- **但**：`location /kb` 会匹配：
  - `/kb`
  - `/kb/`
  - `/kb/abc`
  - **甚至 `/kbc`、`/kbox` 等**（因为是前缀匹配）

> 这可能导致意外代理的预期路径范围过大，存在安全隐患或逻辑错误。

---

## 最佳实践总结

经过四种不同结果测试，要实现 **“去掉 `/kb` 前缀，仅将子路径转发给后端”**，是需要使用：

```nginx
location /kb/ {
    proxy_pass http://172.16.99.20:8070/;  # 注意结尾的 /
}
```

### 关键点：

| 要素 | 要求 |
|------|------|
| `location` | 必须写成 `/kb/`（带尾部 `/`），避免匹配 `/kbc` |
| `proxy_pass` | 必须以 `/` 结尾，才能触发路径重写 |
| 转发结果 | `/kb/xxx` → `/xxx` |

---

## 附AI总结的 URI 重写规则速查

| location | proxy_pass | 请求 URI | 转发 URI |
|--------|------------|--------|--------|
| `/prefix/` | `http://backend/` | `/prefix/path` | `/path`  |
| `/prefix/` | `http://backend` | `/prefix/path` | `/prefix/path`  |
| `/prefix` | `http://backend/` | `/prefix/path` | `/path`（但可能误匹配 `/prefixX`） |
| `/prefix/` | `http://backend/prefix/` | `/prefix/path` | `/prefix/path`（双重前缀）|


