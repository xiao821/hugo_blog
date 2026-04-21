---
title: "Nuxt.js 上手学习笔记（一）：项目结构、自动路由与 API 搭建"
date: 2026-04-21
tags: ["Nuxt", "Vue3", "SSR"]
slug: "Nuxt-1"
description: "记录初次上手 Nuxt.js 的学习过程，包括项目目录结构、组件自动注册、server和api 路由约定以及开发中踩到的坑"
keywords: ["Nuxt.js", "server/api 路由", "Vue3 SSR"]
---

## Nuxt.js （day1） 的学习

最近看到一个新技术就折腾一下，前面看到 qiankun，昨天刷 L 站看到了 Nuxt.js，这不就开始上手边看文档边写项目了。
目前就根据我自己起的项目来做个小总结。

## 1. 项目结构
目前跟普通的 Vue 的区别就是多了一个 `server`，专门用于放与后端 db、api 相关的内容，其他的没太多变化。

| 目录 | 用途 |
|---|---|
| `app.vue` | 根组件，所有页面的入口 |
| `pages/` | 文件即路由，`index.vue` → `/`，`about.vue` → `/about` |
| `components/` | 自动注册的 Vue 组件，无需 import |
| `layouts/` | 页面布局模板，如统一的 header/footer |
| `composables/` | 自动导入的组合式函数，封装可复用逻辑 |
| `plugins/` | 应用启动时执行的插件，如注册第三方库 |
| `middleware/` | 路由守卫，页面跳转前执行，如权限验证 |
| `server/` | 服务端代码，`server/api/` 创建后端接口 |
| `public/` | 静态资源，直接通过 URL 访问，不经过构建 |
| `assets/` | 需要构建处理的资源，如 CSS、图片（会被 Vite 处理） |

## 2. 组件自动注册
Nuxt.js 中的组件创建后无需手动引入，直接使用即可。比如在 `components/` 下新建了 `example.vue`，在 `pages/` 的其他 vue 文件中直接 `<example />` 就能用。

## 3. 外部依赖配置
外部依赖直接在 `nuxt.config.ts` 中配置即可，比如引入 `element-plus`：
```ts
export default defineNuxtConfig({
  compatibilityDate: '2025-07-15',
  devtools: { enabled: false },
  css: ['~/assets/css/main.css'],
  modules: ['@element-plus/nuxt'],
  nitro: {
    experimental: {
      wasm: true
    },
    externals: {
      external: ['better-sqlite3']
    }
  },
  runtimeConfig: {
    jwtSecret: process.env.JWT_SECRET || 'dev-secret-change-in-production'
  }
})
```

## 4. server/api 路由约定
这个项目使用的是 SQLite，无需配置额外环境，简洁方便。
Nuxt 写接口有一个很特别的地方：**接口路径与文件路径直接对应**。比如新增接口放在 `server/api/notes/index.post.ts`，在 Vue 文件中调用就是：

```ts
const note = await $fetch('/api/notes', { method: 'POST', body: {} })
```

如果需要携带 id 参数，文件名写成 `[id].delete.ts` 即可。

### 路由映射规则
`server/routes/` 和 `server/api/` 都是 Nuxt/Nitro 的特殊目录，框架自动处理路由映射，**目录名本身不会出现在 URL 里**。

类比 `pages/` 目录：
```
pages/about.vue  → /about  （不是 /pages/about）
pages/user.vue   → /user   （不是 /pages/user）
```
同理：
```
server/api/notes.get.ts    → /api/notes  （api 是约定前缀）
server/routes/notes.get.ts → /notes      （routes 不加前缀）
```
这是 Nitro 框架的约定：`api/` 自动加 `/api` 前缀，`routes/` 直接映射，两个目录名都不会出现在 URL 里。

### 自定义目录名
如果想自定义目录名，需要在 `nuxt.config.ts` 中通过 `nitro.scanDirs` 配置，但这样做比较麻烦，需要手动定义每个路由。所以一般直接用 `api/` 或 `routes/` 即可。
```ts
// nuxt.config.ts
nitro: {
  scanDirs: ['server/fast']
}
// server/fast/notes.get.ts 的 URL 还是 /notes，不会带 /fast 前缀
// 如果想要 /fast/notes，需要在文件里手动定义路由：
export default defineEventHandler({
  route: '/fast/notes',
  handler: (event) => {
    return { data: 'hello' }
  }
})
```

## 踩坑记录
开发时遇到一个奇怪的报错：
```
Error: IPC connection closed
 ⁃ at Socket.onClose (node_modules/.pnpm/@nuxt+vite-builder@4.4.2_.../vite-node.mjs:140:101)
```
因为确实没有找到错误的头绪，于是询问 GPT 后发现是某个 Vue 文件的语法问题：`<script setup>` 没有加 `lang="ts"`，导致 `catch (e: any)` 这种 TypeScript 写法无法被 Vue 编译器识别。**没有 `lang="ts"` 就不能在 script 中使用 TypeScript 语法**，需要注意。

## 最后 day1 小结
目前学习进度到路由和简单 API 的搭建，后续会继续基于当前项目做进一步学习。