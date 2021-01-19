# vue3 源码分析-domdiff

## 前言

上一篇文章主要分析了 vue3 mount（初始化挂载）流程, 大概的流程图如下：

![/imgs/domdiff/1.png](/imgs/domdiff/1.png)

以上现在数据已经呈现到页面上， 第一次渲染运行完成。当渲染依赖的数据更新时， 则会执行了更新渲染流程，本文主要针对数据更新后 vue3 内部所做的操作尤其是 dom diff。

## 示例代码

```javascript
const app = Vue.createApp({
  data() {
    return {
      list: ['a', 'b', 'c', 'd'],
    }
  },
  methods: {
    change() {
      this.list = ['a', 'd', 'e', 'b']
    },
  },
})
app.mount('#demo')
```

```html
<!DOCTYPE html>
<html>
  <head>
    <meta
      name="viewport"
      content="initial-scale=1.0,maximum-scale=1.0,minimum-scale=1.0,user-scalable=no,target-densitydpi=medium-dpi,viewport-fit=cover"
    />
    <title>Vue3.js hello example</title>

    <script src="../../dist/vue.global.js"></script>
  </head>
  <body>
    <div id="demo">
      <ul>
        <li v-for="item in list" :key="item">{{item}}</li>
      </ul>
      <button @click="change">change</button>
    </div>
    <script src="./hello.js"></script>
  </body>
</html>
```
