---
title: "ElementUI 级联组件实现自定义输入和模糊搜索"
date: 2024-12-23
tags: ["vue.js", "elementui", "前端"]
---

记录一下工作中碰到的需求，对于新手小白碰到这种情况可以参考，高手的话还有很多更好的办法。哈哈。

对于这个需求一开始是很懵的，想到 Element UI 里面有个级联组件，但是这个级联组件不支持自定义输入数据，这个就很烦。于是我在网上搜索，看到了这样一篇文章给了我思路。文章链接在下面可以进行查看。

> **参考文章**：[接世界毁灭 - CSDN 博客](https://blog.csdn.net/weixin_44569565/article/details/131783147)

我试着用该博主的思路，用文本框套一个级联框，然后触发级联组件的点击事件，这样既能有级联效果也有自定义效果。

```html
<li @click="toggleCascaderExpansion" style="width:90%">
  <el-input
    placeholder="请选择原因"
    type="textarea"
    :row="1"
    style="width:100%;font-size:14px;display:inline-block;z-index:99;"
    v-model.sync="addOrEditForm.biztype"
    @input="onBizTypeInput">
  </el-input>
  <el-cascader
    :props="{ multiple: true, checkStrictly: true, value:'id', label:'label' }"
    filterable
    :filter-method="filterMethod"
    placeholder="请选择原因"
    style="width:100%;font-size:14px;display:flex;margin-top: -40px;opacity: 0;height:40px"
    v-model.sync="addOrEditForm.firstbiztype"
    :options="biztypeList"
    @change="handleBizTypeChange"
    ref="cascaderRef">
  </el-cascader>
</li>
```

一开始我用 `ref` 去触发级联组件的下拉框展开事件，发现没效果，后面我测试了很多方法，用下面这个就可以触发。大家可以参考我这个写法。

```javascript
// 触发级联组件
toggleCascaderExpansion() {
  const cascaderTrigger = this.$refs.cascaderRef.$el.querySelector('.el-input--suffix');
  cascaderTrigger.click();
},
```

然后对于怎么区分内容节点，一级内容放在第一个文本框内，第二级内容放在第二个文本框内，需要用到组件中的 `getCheckedNodes` 方法。用这样的方法可以拿到一级节点和二级节点的内容，然后分别赋值就行了。

```javascript
getSelectedNodes() {
  // 通过 $refs 访问 el-cascader 组件实例
  const cascaderInstance = this.$refs.cascaderRef;
  // 使用 checkedNodes 属性获取选中的节点
  const checkedNodes = cascaderInstance.getCheckedNodes();

  // 处理 checkedNodes 数据
  checkedNodes.forEach(node => {
    // console.log(node.value, node.label);
  });

  // 返回选中的节点，或者根据需要处理这些节点
  return checkedNodes;
},
```

后面对于怎么获取级联框的数据，我用的是一种比较粗暴的方法，我只能想到这样，我是先获取到选中的节点进行存储，然后再对一级节点和二级节点分别过滤，然后再进行拼接赋值给数据呈现在级联框里面，这也算是实现了级联框自选和自定义输入已经自动赋值给二级输入框的功能，或者点击第二级节点可以把第一级的赋值给级联框内，二级节点内容赋值给文本输入框。（部分代码）

```javascript
const firstSelectedBizSourceText = firstselectLabels.join(', ');
const selectedBizSourceText = selectLabels.join(', ');

// 追加到现有数据后面
this.addOrEditForm.firstselectedBizSourceText = this.addOrEditForm.biztype + firstSelectedBizSourceText;
this.addOrEditForm.biztype = '';
this.addOrEditForm.biztype = this.addOrEditForm.firstselectedBizSourceText;

// 追加到现有数据后面
this.addOrEditForm.selectedBizSourceText = this.addOrEditForm.bizsource + selectedBizSourceText;
this.addOrEditForm.bizsource = this.addOrEditForm.selectedBizSourceText;
```

最后还需要实现模糊搜索的功能的话，需要在组件上添加 `:filter-method="filterMethod"` 这个属性，官方文档上也有，可以进行参考。我这边是使用了 `watch` 进行监听输入框的变化，然后呈现搜索出来的值。

```javascript
filterMethod(value, arr) {
  if (!arr) {
    return [];
  }
  const newarr = [];
  arr.forEach(element => {
    // 检查当前节点的 label 是否包含 value
    if (element.label.toLowerCase().includes(value.toLowerCase())) {
      // 如果当前节点匹配，则递归处理子节点
      // const filteredChildren = this.filterMethod(value, element.children);
      const obj = {
        ...element,
        children: element.children || []
      };
      if (obj !== null && obj !== undefined) { // 确保对象不是 null 或 undefined
        newarr.push(obj);
      }
    } else if (element.children && element.children.length > 0) {
      // 如果当前节点不匹配但有子节点，则递归处理子节点
      const filteredChildren = this.filterMethod(value, element.children);
      if (filteredChildren.length > 0) {
        // 如果子节点中有匹配的，则将当前节点添加到结果中
        const obj = {
          ...element,
          children: filteredChildren
        };
        if (obj !== null && obj !== undefined) { // 确保对象不是 null 或 undefined
          newarr.push(obj);
        }
      }
    }
  });
  // console.log(newarr);
  return newarr;
},
```

> **(2024/12/23: 更新：支持多选节点后还能进行模糊搜索)**

```javascript
// 来电原因模糊搜索
filterMethod(value, arr) {
  if (!arr) {
    return [];
  }
  const keywords = value.split(/[,，]/).map(keyword => keyword.trim().toLowerCase());
  const newarr = [];
  arr.forEach(element => {
    if (keywords.some(keyword => element.label.toLowerCase().includes(keyword))) {
      const obj = {
        ...element,
        children: element.children || []
      };
      newarr.push(obj);
    } else if (element.children && element.children.length > 0) {
      const filteredChildren = this.filterMethod(value, element.children);
      if (filteredChildren.length > 0) {
        const obj = {
          ...element,
          children: filteredChildren
        };
        newarr.push(obj);
      }
    }
  });
  return newarr;
},

onBizTypeInput(event) {
  // console.log('event',event);
  // && event.match(/[\u4e00-\u9fa5]+/)
  if (event) {
    this.biztypeList = this.filterMethod(event, this.originalBiztypeList); // 使用过滤后的数据更新 biztypeList
  } else {
    this.biztypeList = this.originalBiztypeList; // 如果没有输入值，则显示全部数据
  }
},
```

最后该功能也是能基本实现，但是在我自己的测试下还是会有一些不完善的地方。

> （比如如果当我自定义输入的时候就没有办法去选中级联框里面已有的数据，因为会默认自定义输入的内容是搜索效果，如果没有匹配就会显示无数据。）  
> ----- **该功能已完善更新**

初次写文章，可能表达不是很好，敬请见谅，希望能对大家有帮助。