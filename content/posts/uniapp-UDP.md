---
title: "在 uni-app 中实现 UDP 扫描局域网 IP"
date: 2026-02-05
tags: ["uniapp", "UDP", "局域网扫描"]
slug: "uniapp-udp-lan-scan"
description: "在 uni-app 中使用 udp-client 插件实现局域网设备扫描功能，包括插件配置、自定义基座打包、UDP 广播通信等完整实现方案"
keywords: ["uniapp", "UDP", "局域网扫描", "udp-client", "广播通信"]
---

这两天由于事情不多，能有一点时间解决在 uniapp 中扫描局域网 IP 的功能，虽然这个后端也能做，但是这不恰好我前端也能做，也没啥其他问题。于是就开始琢磨起来。

一开始我去问 GPT，给出的回答是比较一般的，他跟我说要用 http 去轮询接口，这个不还是要后端配合，然后我说还有没有其他的。他给出的一个其他的方法，但是也是丝毫没效果的。于是我放弃了直接跟 AI 沟通就能解决的想法。

后面尝试古法搜索，直接上网页搜索，虽然也找了大半天，但是终究是找到了一个能用的解决方法，我看到有人推荐去使用 uniapp 中的插件 

> **参考文章**：[xl__qd - CSDN 博客](https://blog.csdn.net/lcc2001/article/details/134135172)

我找到了这个[udp-client](https://ext.dcloud.net.cn/plugin?id=6833) 的插件，这个插件完全免费，于是我下载下来，将里面的 `aar` 包放到我的项目中。

这个地方需要注意，放到我们项目中之后，我们需要在 Hbuildx 中将这个包在 mainfest.json中的安卓/IOS 原生插件配置页面中引入进来，

![](https://xssblog.211508.xyz/2026/02/556b4e251bbae1f4b2c4525f514f243e.png)

引入进来后，我们如果要想看到效果直接调试运行到 Android 基座是还不行的，此时的UDP 插件还是不可用，没有集成 udp-client 原生插件，一开始我还不明白为什么，后面问了一下 GPT 说因为我们改动了 mainfest.json文件，是需要我们重新打个 apk 包的，且在打包时需要选择打自定义调试基座，这里选这个主要是因为方便我们调试测试。

![](https://xssblog.211508.xyz/2026/02/cfb9c74986fbf4983fca84ce36195d04.png)

等待打包完成后就可以在运行到 Android APP 基座时选择本地基座，这里为什么要选择这个呢？还是因为我们引入了新的插件，如果是选择已安装的基座的话，这个地方是没办法去测试我们安装的插件效果的，这个也是需要注意的一点。

![](https://xssblog.211508.xyz/2026/02/8a5d44c5ff142b2c3b9fcd3346acece6.png)

当我们这些步骤完成后，我们看一下代码层面，这里因为我们是只说前端，后端部分我们是不管的。

**核心代码如下：**

```vue
<script setup>
/** ========== 扫描协议与参数 ========== */
const DISCOVERY_PORT = 39876      // 设备端监听发现端口（你往这发）
const UDP_LISTEN_PORT = 39878     // 本机监听端口（设备回包发到这）
const SCAN_TIMEOUT = 5000         // 扫描窗口（ms）
const UDP_ATTEMPTS = 2            // 广播次数
const UDP_INTERVAL = 300          // 广播间隔（ms）

<!-- 这部分是需要与后端约定好的数据格式 -->
const MAGIC_TYPE = 'edge_discovery'
const REQ_OP = 'whois'
const RESP_OP = 'iam'
const PROTO_VER = 1
const DISCOVERY_SERVICE = 'edge-mon'

/** ========== UDP 实例与扫描上下文 ========== */
let udpClient = null
let udpReady = false
let currentScan = null

/** ========== 工具：nonce ========== */
const generateNonce = () =>
  Math.random().toString(36).slice(2) + Math.random().toString(36).slice(2)

/** ========== 1) UDP 收包处理 ========== */
const onUdpMsg = (resData) => {
  if (!currentScan) return

  // 插件回调可能是 {data, host} 或字符串
  const payload = typeof resData === 'string' ? resData : resData?.data
  if (typeof payload !== 'string') return

  const cleaned = payload.replace(/\0/g, '').trim()

  let msg
  try {
    msg = JSON.parse(cleaned)
  } catch {
    return
  }

  // 协议过滤：只接收目标协议的回包
  if (!msg || msg.type !== MAGIC_TYPE || msg.op !== RESP_OP) return
  if (msg.service && msg.service !== DISCOVERY_SERVICE) return
  if (msg.nonce && msg.nonce !== currentScan.nonce) return

  const ip = msg.ip || resData?.host || msg.host
  if (!ip) return

  // 设备结构（你真正要展示/使用的字段）
  const device = {
    id: msg.id || 'unknown',
    hostname: msg.hostname || msg.host || '',
    ip,
    os: msg.os || '',
    arch: msg.arch || '',
    service: msg.service || DISCOVERY_SERVICE,
    version: msg.version || ''
  }

  // 去重：同一设备可能回多次
  const key = `${device.id}@${device.ip}`
  if (!currentScan.found.has(key)) {
    currentScan.found.set(key, device)
    devices.value = Array.from(currentScan.found.values())
  }
}

/** ========== 2) UDP 错误处理 ========== */
const onUdpError = (err) => {
  // 这里保持最简：需要你自己在 UI/日志里显示也可以
  console.error('[udp] error:', err)
}

/** ========== 3) 初始化 UDP ========== */
const ensureUdpClient = () => {
  // #ifdef APP-PLUS
  if (udpClient && udpReady) return true

  try {
    udpClient = uni.requireNativePlugin('udp-client')
  } catch {
    udpClient = null
  }
  if (!udpClient) return false

  // 防止热重载/异常导致端口未释放
  try {
    udpClient.release?.()
  } catch {}

  udpClient.setByteSize?.(4096)
  udpClient.setIsDebug?.(true)

  // 监听回包端口
  udpClient.init?.(UDP_LISTEN_PORT, onUdpMsg, onUdpError)
  udpReady = true
  return true
  // #endif

  // #ifndef APP-PLUS
  return false
  // #endif
}

/** ========== 4) 扫描主流程：广播 + 等待窗口 ========== */
const scanNetwork = async () => {
  // #ifdef APP-PLUS
  if (!ensureUdpClient()) {
    throw new Error('UDP 插件不可用（请确认已集成 udp-client 且使用自定义基座/打包）')
  }

  // 广播地址：最核心最通用的一个
  const broadcastHosts = ['255.255.255.255']

  const nonce = generateNonce()
  const payload = JSON.stringify({
    type: MAGIC_TYPE,
    op: REQ_OP,
    service: DISCOVERY_SERVICE,
    nonce,
    ver: PROTO_VER,
    replyPort: UDP_LISTEN_PORT
  })

  return new Promise((resolve) => {
    currentScan = {
      nonce,
      found: new Map(),
      done: false,
      timer: null,
      broadcastTimers: [],
      resolve
    }

    const sendBroadcast = () => {
      if (!udpClient) return
      if (!currentScan || currentScan.nonce !== nonce) return

      broadcastHosts.forEach((host) => {
        try {
          udpClient.send({
            host,
            port: DISCOVERY_PORT,
            data: payload,
            useHex: false
          })
        } catch (e) {
          console.error('[udp] send failed:', e)
        }
      })
    }

    // 立即发送一次 + 间隔补发
    sendBroadcast()
    for (let i = 1; i < UDP_ATTEMPTS; i++) {
      const t = setTimeout(sendBroadcast, i * UDP_INTERVAL)
      currentScan.broadcastTimers.push(t)
    }

    // 扫描窗口结束
    currentScan.timer = setTimeout(() => {
      if (!currentScan || currentScan.nonce !== nonce) return resolve()
      currentScan.done = true
      devices.value = Array.from(currentScan.found.values())
      currentScan = null
      resolve()
    }, SCAN_TIMEOUT)
  })
  // #endif

  // #ifndef APP-PLUS
  throw new Error('仅 App-Plus 支持 UDP 扫描')
  // #endif
}

/** ========== 5) 对外入口：开始扫描 ========== */
const startScan = async () => {
  if (isScanning.value) return
  isScanning.value = true
  devices.value = []

  try {
    await scanNetwork()
  } finally {
    isScanning.value = false
  }
}

/** ========== 6) 清理资源 ========== */
const cleanup = () => {
  if (currentScan?.timer) clearTimeout(currentScan.timer)
  currentScan?.broadcastTimers?.forEach((t) => clearTimeout(t))

  // 如果 scanNetwork 的 Promise 还没结束，主动结束它
  if (currentScan && !currentScan.done && typeof currentScan.resolve === 'function') {
    try {
      currentScan.done = true
      currentScan.resolve()
    } catch {}
  }
  currentScan = null

  // #ifdef APP-PLUS
  try {
    udpClient?.release?.()
  } catch {}
  // #endif

  udpClient = null
  udpReady = false
  isScanning.value = false
}

defineExpose({ startScan, cleanup })

onBeforeUnmount(() => {
  cleanup()
})
</script>
```

**代码中还有一个获取 Android 定位权限**,这一步是说需要获取 WIFI 信息，因为我的运行环境是 pad，所以写了部分，但是目前我测试就算是不开定位权限也是可以正常使用的。可以选择性忽略。

```js
const ensureLocationPermission = async () => {
  // #ifdef APP-PLUS
  if (locationPermissionResult === true) return true
  if (plus.os.name !== 'Android') {
    locationPermissionResult = true
    return true
  }
  return new Promise((resolve) => {
    try {
      const perms = [
        'android.permission.ACCESS_FINE_LOCATION',
        'android.permission.ACCESS_COARSE_LOCATION'
      ]
      plus.android.requestPermissions(perms, (result) => {
        const deniedAlways = result.deniedAlways || []
        const deniedPresent = result.deniedPresent || []
        if (deniedAlways.length) {
          addLog('定位权限被永久拒绝，请在系统设置中开启定位权限')
          locationPermissionResult = false
          resolve(false)
          return
        }
        if (deniedPresent.length) {
          addLog('定位权限被拒绝，可能无法获取 Wi-Fi IP')
          locationPermissionResult = false
          resolve(false)
          return
        }
        locationPermissionResult = true
        resolve(true)
      }, (error) => {
        addLog('定位权限请求失败: ' + (error && error.message ? error.message : '未知错误'))
        locationPermissionResult = false
        resolve(false)
      })
    } catch (error) {
      addLog('定位权限请求异常: ' + (error && error.message ? error.message : '未知错误'))
      locationPermissionResult = false
      resolve(false)
    }
  })
  // #endif
  return true
}
```

**碰到的问题：**

1. udp-client插件运行出现UDP异常，bind failed:EADDRINUSE(address already in use)错误  
   这个问题应该是我在开发时启动了多个扫描页面，导致地址端口被占用了。
2. 出现 socket close  
   这个问题暂时没搞懂为什么，因为我重启运行测试后莫名其妙就好了，有点摸不着头脑。

**最后比较重要的一个点就是这个 udp-client 插件是需要有这些权限的**

```
"android.permission.ACCESS_NETWORK_STATE", "android.permission.CHANGE_NETWORK_STATE", "android.permission.ACCESS_WIFI_STATE", "android.permission.CHANGE_WIFI_STATE"
```

然后这个使用范围是只能是 APP，且 minSdkVersion在 21 以上
