# 如何解决 JavaScript 大整数精度丢失问题

关于这个大精度丢失问题，一开始我还没在意，直到上班碰到了这个问题，当时记得在处理后端返回的大整数拿到一个超过 js 精度的 id 时，怎么传值都对不上，后面通过打印检查对比发现，我拿到的 id 跟我所获取到的不一致，当时还不知道是什么情况，后面上网搜索才知道这是属于一个精度丢失问题。

那原因是什么呢？  
经过查询知道 JS 的 Number 类型遵循 IEEE754 标准，最大安全整数是 `2^53 - 1`。  
再大的数字会被强制四舍五入，导致精度丢失。

那我为了避免这种情况，后续在 JSON 解析和对象处理阶段，手动把大整数转换为字符串。下面是我这边写的一个工具方法。

---

## 问题示例：精度丢失

后端返回：

```json
{
  "nxcallid": 1234567890123456789
}
```

如果直接 `JSON.parse`：

```js
JSON.parse('{"nxcallid":1234567890123456789}')
// => { nxcallid: 1234567890123456800 } 精度已丢失
```

---

## 核心思路：将大整数字段转为字符串再解析

只要能让 JSON 解析器把这些字段当作字符串读取，精度就能保留。

---

## 方法代码

### 1. 安全解析 JSON（处理大整数字段）

```ts
export function safeJsonParse(jsonString: string, bigIntFields: string[] = ['nxcallid']): any {
  if (!jsonString) return null;
  
  let processedJson = jsonString;
  
  // 将大整数字段强制替换成字符串
  bigIntFields.forEach(field => {
    const regex = new RegExp(`"${field}":\\s*(\\d+)`, 'g');
    processedJson = processedJson.replace(regex, `"${field}":"$1"`);
  });
  
  try {
    return JSON.parse(processedJson);
  } catch (error) {
    console.error('JSON解析失败:', error);
    throw error;
  }
}
```

---

### 2. 处理数组中的大整数字段

```ts
export function processBigIntFields(data: any[], bigIntFields: string[] = ['nxcallid']): void {
  if (!Array.isArray(data)) return;
  
  data.forEach(item => {
    if (item && typeof item === 'object') {
      bigIntFields.forEach(field => {
        if (item[field] !== undefined && item[field] !== null) {
          item[field] = String(item[field]);
        }
      });
    }
  });
}
```

---

### 3. 处理对象中的大整数字段

```ts
export function processBigIntFieldsInObject(data: any, bigIntFields: string[] = ['nxcallid']): void {
  if (!data || typeof data !== 'object') return;
  
  bigIntFields.forEach(field => {
    if (data[field] !== undefined && data[field] !== null) {
      data[field] = String(data[field]);
    }
  });
}
```

---

## 使用示例

### 解析 JSON 字符串：

```js
const raw = '{"nxcallid": 1234567890123456789}';

const result = safeJsonParse(raw);
console.log(result.nxcallid);
// "1234567890123456789" ✔ 精度完好
```

---

### 处理数组：

```js
processBigIntFields(listData, ['userid', 'nxcallid']);
```

---

## 总结

JavaScript 精度问题不可避免，但可以通过简单规则避开，比如：

- 避免让大整数进入 JS number 体系  
- 在解析 JSON 时，将大整数统一转为字符串  
- 对对象和数组中的大整数字段做二次处理