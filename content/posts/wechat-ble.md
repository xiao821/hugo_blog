---
title: "微信小程序 BLE 扫描变慢与 allowDuplicatesKey 排查"
date: 2026-05-12
tags: ["微信小程序", "BLE", "蓝牙"]
slug: "wechat-ble-allowduplicateskey"
description: "记录 UniApp 开发微信小程序蓝牙通信时，BLE 设备断开后重新扫描缓慢的问题，以及 allowDuplicatesKey 设置为 true 后的排查结论。"
keywords: ["微信小程序", "BLE", "蓝牙扫描", "allowDuplicatesKey", "onBluetoothDeviceFound"]
---

今天在做微信小程序获取蓝牙权限 ble 扫描对应的无线设备时，发现一个这样的情况，我初次启动小程序扫描蓝牙是很快能扫描到，也能很快建立连接，但是我一旦断开蓝牙，重新打开，我就发现这个扫描无线设备非常缓慢，一直处于扫描中的状态。我就不是很明白，为什么会这样。于是我上网查了一下“UniApp 开发微信小程序蓝牙通信”的问题。我就找到了一篇参考文章

> **参考文章**：[wx66ece9f42611c - UniApp 开发微信小程序蓝牙通信：从流程到避坑](https://blog.51cto.com/u_17028302/14274837)

看到了一个关键数据“allowDuplicatesKey”，我在代码中是设置的 false，我就在想是不是这个问题，但是我问了 GPT，它说是跟这个参数无影响，

```text
allowDuplicatesKey: false 只是表示本次扫描中同一个设备不重复回调。
它不会导致“第一次连接过的设备以后扫描不到”。
```

但是我想了想，还是死马当活马医，我改一下试试看。

```js
// 持续扫描：开启后不会自动停止，需显式 stopDiscovery()
export async function startDiscovery(onDeviceFound = null, options = {}) {
  uni.onBluetoothDeviceFound(res => {
    emitScannedDevices(res.devices, onDeviceFound, options, sid);
  });

await startDiscovery(onDeviceFound, {
    allowDuplicatesKey: true,   // 允许重复上报同一设备（保持信号更新）
    scanWithServiceFilter: false,
    interval: 0,
    restartDelayMs: 300,        // 停旧扫描后等 300ms 再启
    cacheRefreshDelayMs: 1200   // 启动后 1.2s 再读一次缓存
})
```

我发现改成了 true 后居然解决了这个问题，我就很好奇，claude 给出的回答是这样的

> allowDuplicatesKey:false（默认）：同一设备只上报一次，之后即使收到广播包也不再触发回调。如果第一次扫描时设备信号弱、广播包没被接收到，就永远匹配不到了。
>
> ⏺ 原因在于微信小程序的蓝牙缓存机制：
>
> false 时慢的原因：
>
> 关闭蓝牙再开启后，系统缓存被清空。allowDuplicatesKey: false 下，每个设备只上报一次。如果这"一次"上报时我的手机恰好没收到设备的广播包（信号抖动、时序问题），这个设备就再也不会被上报了，只能等下一轮扫描重启（10秒后）才有机会再匹配一次。
>
> true 时快的原因：
>
> 每次收到广播包都触发回调。设备通常每隔几百毫秒就广播一次，只要设备在范围内，很快就会有一次广播包被手机接收到并触发回调，立即匹配成功。
>
> 本质上：false 是"只给你一次机会"，true 是"持续给机会直到匹配成功"。关闭蓝牙后缓存清空，false 模式下那"一次机会"很容易错过，true
> 模式下错过了还有下一次广播包。

但是如果改为 true 需要注意的一点就是会造成设备每广播一次（通常每 100-500ms 一次），onBluetoothDeviceFound 就触发一次，我的匹配函数就被调用一次。false 时同一设备只触发一次，true 时可能每秒触发几次。但是因为我匹配逻辑里有 if (this.bleConnected || this.bleConnecting) return 保护，找到设备开始连接后就不再处理重复上报，所以没有啥副作用，可以这样写。

```js
if (this.bleConnected || this.bleConnecting) return
 const targetId = this.bleTargetDeviceId
 if (!targetId) return
 const matched = device?.deviceId === targetId || device?.name === targetId
 if (matched) {
   this.clearBleDiscoveryRestartTimer()
   this.bleConnectTarget(device)
 }
```
