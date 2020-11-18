# vue3 源码分析步骤 及 createApp 方法分析

## 前言

vue 3.0 相较于 vue2.0 主要针对 config api 改为 composition api、typescript 的支持、vue 的内部结构被重写成了一个个可以解耦的模块（允许编译后 tree-shaking），从而减小体积加快运行速度。

## v3.0 新特性介绍

### Composition-API

#### setup

一旦 props 被解析，新的 setup 组件选项在创建组件之前执行，并被当作 Composition API 的入口点，但在执行 setup 时尚未创建组件实例，因此在 setup 选项中没有 this。setup 选项应该是一个接受 props 和 context 的函数。

#### ref

过使用一个新的 ref 函数使任何变量在任何地方变成响应

```javascript
import { ref } from 'vue'

const counter = ref(0)

console.log(counter) // { value: 0 }
console.log(counter.value) // 0

counter.value++
console.log(counter.value) // 1
```

#### watch

#### computed

## 下载项目

[vue3.0 项目地址](https://github.com/vuejs/vue-next)

```shell
git clone https://github.com/vuejs/vue-next.git
```

## 如何调试源码

项目下载后打开项目

![/imgs/start/1.png](/imgs/start/1.png)

vue 项目主要包含 3 个文件夹： packages、scripts、test-tds

- packages 源码主目录 存放 vue 的核心代码
- scripts 脚本文件 主要用来编译打包
- test-tds 测试 ts 的文件

## packages 文件夹结构分析

![/imgs/start/2.png](/imgs/start/2.png)

- compiler-core 编译-vue 核心
- compiler-dom 编译-dom 部分（浏览器）
- compiler-sfc 编译-单文件组件
- compiler-ssr 编译-服务端渲染
- reactivity 响应式
- runtime-core 运行时核心 包含声明周期、vnode、watch 等
- runtime-dom 运行时 dom 相关 包含 createApp 实现等
- runtime-test 运行时测试代码
- server-renderer 服务端渲染代码
- shared 工具类等
- size-check 测试 vue 包大小体积使用
- template-explorer vue 内部的编译文件浏览工具
- vue vue 的主入口文件

## createApp 使用方法

vue3.0 中是用 createApp 方法生成 vue 实例，回顾下 vue 2.0 版本中是如何生成实例的

### vue 2.0

```javascript
// 方式一
new Vue({
  el: '#app',
  data: {
    message: 'Hello Vue!',
  },
})
// 方式二
const App = {
  data() {
    return {
      message: 'Hello Vue!',
    }
  },
}
new Vue({
  // h 为 createElement 方法
  render: (h) => h(App),
}).$mount('#app')
```

### vue3.0

```javascript
const App = {
  data() {
    return {
      message: 'Hello Vue!',
    }
  },
}
Vue.createApp(App).mount('#app')
```

## createApp 源码分析

createApp 的相关源码放置在 `/packages/runtime-dom/src/index.ts` 中

### createApp

```typescript
export const createApp = ((...args) => {
  const app = ensureRenderer().createApp(...args)

  if (__DEV__) {
    injectNativeTagCheck(app)
  }

  const { mount } = app
  app.mount = (containerOrSelector: Element | string): any => {
    const container = normalizeContainer(containerOrSelector)
    if (!container) return
    const component = app._component
    if (!isFunction(component) && !component.render && !component.template) {
      component.template = container.innerHTML
    }
    // clear content before mounting
    container.innerHTML = ''
    const proxy = mount(container)
    container.removeAttribute('v-cloak')
    container.setAttribute('data-v-app', '')
    return proxy
  }

  return app
}) as CreateAppFunction<Element>
```

- 上述代码中 `ensureRenderer` 方法顾明思议确保 renderer 对象是否存在，如果没有则生成一个（调用 createRenderer），生成 app 实例对象

```javascript
function ensureRenderer() {
  return (
    renderer || ((renderer = createRenderer < Node), Element > rendererOptions)
  )
}
```

- 在 `__DEV__` 开发模式下, 在 app.config 对象上添加 `isNativeTag` 属性

```javascript
function injectNativeTagCheck(app: App) {
  // Inject `isNativeTag`
  // this is used for component name validation (dev only)
  Object.defineProperty(app.config, 'isNativeTag', {
    value: (tag: string) => isHTMLTag(tag) || isSVGTag(tag),
    writable: false,
  })
}
```

- 从 app 对象中解构 mount 方法暂存，然后重写 `app.mount`
-

### createRenderer

### mount
