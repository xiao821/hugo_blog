---
title: "在 uni-app 中实现稳定的中文拼音排序（兼容 App/小程序）"
date: 2026-01-28
tags: ["uniapp", "localeCompare"]
slug: "uniapp-chinese-pinyin-sort"
description: "解决 uni-app 中 localeCompare 在 App 端排序失效的问题，使用 pinyin-pro 库实现稳定的中文拼音排序方案"
keywords: ["uniapp", "localeCompare", "pinyin-pro", "拼音排序"]
---

今天在做开发时，同事提出前端能不能直接对一些内容做排序，不需要依靠后端返回的拼音来做。我一开始还不太明白怎么做，那么就需要交给万能的 claude 大神，果不其然，给出了一个"完美"的方案，那就使用localeCompare方法，这个方法能支持实现排序，然后我又去查询了解这个方法，在 MDN 这个网站中找到了对这个 
[localeCompare](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/String/localeCompare?utm_source=chatgpt.com) API 的解释描述。

在仔细看完后我发现了一个问题，这个方法在` webview `中不一定适配，恰好我的运行环境是 app，需要在手机上运行，虽然我在电脑上运行过了是正常的，于是我打包一个 wgt 包进行更新，果然有问题，这个名称排序乱七八糟的。根本不能使用。

---

## 为什么不能直接用 localeCompare？

那是为什么不能直接用 localeCompare 呢？这是因为浏览器内置了完整的 Intl / 本地化数据，可以进行语言敏感的比较（也可直接用 Intl.Collator）。

但是在部分 App 运行时环境：

- JS 引擎内可能没有完整的 Intl 国际化支持；
- localeCompare 或 Intl.Collator 即使存在，也可能不支持拼音排序；
- 导致排序结果看起来"失效"或按 Unicode 顺序排序。

这个问题在 uni-app 的 Android / iOS 打包场景中尤其常见 —— 因为移动端 JS 引擎（如部分 WebView、JSC）并不会像桌面浏览器那样内置完整的 Intl 数据。

---

## 使用 pinyin-pro 库

于是我又去搜索，看看还有没有其他更可靠的方法，在网上果然也看到了有人出现了这样的问题，我看到可以使用 pinyin 库来把中文转成拼音再去排序 ，于是我下载了这个库 "pinyin-pro": "^3.0.0"。

[pinyin-pro](https://github.com/zh-lx/pinyin-pro) 是一个高性能、准确度极高的 汉字转拼音工具，它支持：

- 返回标准拼音（带音调）
- 返回不带声调拼音
- 支持数组形式或字符串形式输出

属于专为中文处理设计的 JS 库。

我们通过它把汉字转换成拼音，用于排序键（key）。

### 核心好处

>  不依赖运行环境是否支持 Intl  
>  适合 App + Web 通用  
>  可以处理多音字（拼音-pro 具有更好的准确率）

### 代码实现

```javascript
import { pinyin } from 'pinyin-pro'

const pinyinCache = new Map()

const buildPinyinKey = name => {
	const raw = String(name || '').trim()
	if (!raw) return ''
	if (pinyinCache.has(raw)) return pinyinCache.get(raw)
	let key = ''
	try {
		const value = pinyin(raw, { toneType: 'none' })
		key = Array.isArray(value) ? value.join('') : String(value || '')
	} catch (e) {
		key = ''
	}
	key = key.replace(/\s+/g, '').toLowerCase()
	pinyinCache.set(raw, key)
	return key
}
```

然后代码这样写，之后继续打包测试，测试时看到果然排序正常了。

---

## 总结

最后总结在 uni-app 或其他跨平台环境中：

1. **不要依赖**浏览器标准的 Intl 本地化排序
2. **优先使用**拼音转换 + 自定义排序键
3. **可选使用**标准 Collator 优化体验，但需兼容快速退化
