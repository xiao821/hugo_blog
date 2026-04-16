---
title: "微前端 qiankun 的学习"
date: 2026-04-16
tags: ["前端", "微前端", "qiankun", "Vue3"]
slug: "qiankun-1"
description: "记录初学微前端 qiankun 的理解，以及主应用注册子应用、Vue 子应用接入 qiankun 的基础代码和注意点"
keywords: ["微前端", "qiankun", "Vue3", "vite-plugin-qiankun", "主应用", "子应用"]
---

其实在最初找工作的时候就跟一些同为前端的朋友交流过，那个时候了解到的一个概念是叫“大前端”，当时其实是有点懵的，刚毕业感觉自己什么都不会，这个概念也是第一次听到，后面了解到的就是前端不仅仅只是做web开发。还需要涉及更多领域，比如说微信小程序，APP,桌面端（Electron），甚至是 Node.js,SSR 这种。后面上班后视野逐渐更加开阔，了解到了一点点 docker 的皮毛，熟悉了 nginx等这种工程化的内容。对于这个大前端也有了一定的概念。

随着工作时间的增加，偶然看到了微前端的这个概念。这个虽然在之前也听过，可是我并没有去了解，但是今天我在空闲的时间中，依靠如今的 AI 帮我做了一个深入的了解。我让 AI 帮我搭建了一个基础的项目框架，然后我看了一下它写的代码，大概就了解了这个概念。微前端无非就是一个大容器中我可以随时放任何内容，想放哪个就放哪个，不要哪个就拿出来即可。且这些内容可以由多个不同的人来开发。彼此之间不会受到任何干扰。这大概就是我所了解到的。虽然可能看起来比较浅显。但是我想应该就是这样了。

---

## 下面我输出一些基础的代码来看看

首先是主应用，也就是所谓的放子应用的容器,其实就是一个独立的 Vue3 项目

### App.vue中的核心部分是这样的

```vue
<template>
  <div>
    <nav style="padding:10px;background:#f0f0f0">
      <a href="/sub-vue">Vue3 Sub</a> |
      <a href="/sub-react">React Sub</a> |
      <a href="/sub-vanilla">Vanilla Sub</a>
    </nav>
    <div id="subapp-container"></div>
  </div>
</template>
```

因为我初识这个微前端，搭建的内容可能简单粗浅。这个地方可以看到我有三个子应用，我选择了 vue, react, 和原生的 js.
这个地方的重要点是 a 链接中的 href 和下面 div 中的 id,这些内容需要跟在主应用中注册微应用中的内容对上,记得需要在 main.js 中需要引入前端和对应用进行注册。

### micro-apps.js

```js
export const microApps = [
  {
    name: 'sub-vue',
    entry: '//localhost:3001',
    container: '#subapp-container',
    activeRule: '/sub-vue',
  },
  {
    name: 'sub-react',
    entry: '//localhost:3002',
    container: '#subapp-container',
    activeRule: '/sub-react',
    props: {},
    sandbox: { loose: true },
  },
  {
    name: 'sub-vanilla',
    entry: '//localhost:3003',
    container: '#subapp-container',
    activeRule: '/sub-vanilla',
  },
]
```

### main.js

```js
import { createApp } from 'vue'
import { registerMicroApps, start } from 'qiankun'
import App from './App.vue'
import { microApps } from './micro-apps'

registerMicroApps(microApps)
start()

createApp(App).mount('#app')
```
---

## 下面我举个微前端的例子，我本人是熟悉 vue 的，我就从 vue 中来看看子应用是什么情况。

首先看看 子应用 vue 项目中的main.js 文件，这个里面多了这些内容，

- 这个renderWithQiankun是主应用加载子应用时调用的一些生命周期

- bootstrap: 子应用首次初始化

- mount: 子应用挂载到 #subapp-container

- unmount: 子应用从页面卸载

- update: 主应用传入 props 更新时触发

而下面那个 if 条件是相当于子应用自己单独打开，独立运行访问地址时是直接立即调用 render()

```js
import { renderWithQiankun, qiankunWindow } from 'vite-plugin-qiankun/dist/helper'

renderWithQiankun({
  bootstrap() {},
  mount(props) { render(props) },
  unmount() { app.unmount() },
  update() {},
})

if (!qiankunWindow.__POWERED_BY_QIANKUN__) {
  render()
}
```

还有一个 vite.config.js 文件，如果要被 qiankun 主应用所加载使用就需要在plugins中写入qiankun('sub-vue', { useDevMode: isDev })这个内容。

```js
export default defineConfig({
  plugins: [vue(), qiankun('sub-vue', { useDevMode: isDev })],
})
```

目前看起来微前端做开发很好的点就是真的很独立，互不影响，一个人一个子应用，随便怎么开发和部署。但是不好的点在于如果需要联动，比如说如果不同页面之间共享大量状态、方法等，我感觉就会比普通项目更加难以操作。
这些内容就是我初学微前端 qiankun 所了解到的一些知识点和需要注意地方。后面有时间还会继续研究和写一个新的完整的微前端项目。
