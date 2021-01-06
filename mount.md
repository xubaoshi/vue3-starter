# vue3 源码分析-mount

## 前言

上一篇主要学习了 createApp api 的相关源码，其方法内部涉及到的 mount 方法其实是 createApp 方法内部最核心的操作，本文主要针对 mount 方法进行深入分析。

## 下载源码开启调试模式

下载项目

```shell
git clone https://github.com/vuejs/vue-next.git
```

更改 package.json scripts 选项

```json
{
  "dev": "node scripts/dev.js"
}
```

```json
{
  "dev": "node scripts/dev.js --sourcemap --environment TARGET:web-full-dev"
}
```

更改此配置的目的为执行 `yarn run dev` 时能够获取编译后的 source-map 方便打断点调试源码。

开启两个命令行终端，先后运行

```shell
yarn run dev
```

![/imgs/mount/1.png](/imgs/mount/1.png)

```shell
yarn run serve
```

![/imgs/mount/2.png](/imgs/mount/2.png)

`yarn run dev` 命令运行后打包的文件放置在 `/packages/vue/dist/vue.global.js` 可以通过 `http://localhost:5000/packages/vue/dist/vue.global.js`。

注：调试时确保 js 已经加载，可以将 script 引入头部（阅读源码时）。

```html
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>Document</title>
    <script src="http://localhost:5000/packages/vue/dist/vue.global.js"></script>
  </head>
  <body>
    <div id="demo"></div>
    <script src="./hello.js"></script>
  </body>
</html>
```

## mount 方法

```javascript
Vue.createApp(App).mount('#app')
```

上期中主要分析了 createApp 过程，本文只要涉及 mount 方法的实现。

**packages/runtime-core/src/apiCreateApp.ts**

```javascript
mount(rootContainer: HostElement, isHydrate?: boolean): any {
  if (!isMounted) {
    // 1. 调用 `createVNode` 方法获取 vnode, 其中 `rootComponent` 即为调用 `createApp(config)` 数据, rootProps 为其传入的对应组件的属性。
    const vnode = createVNode(
      rootComponent as ConcreteComponent,
      rootProps
    )
    // 2. 保存 context 在根节点上
    vnode.appContext = context

    if (isHydrate && hydrate) {
      hydrate(vnode as VNode<Node, Element>, rootContainer as any)
    } else {
      // 3.调用 rander 渲染函数
      render(vnode, rootContainer)
    }
    // 4.isMounted 设置为 true
    isMounted = true
    // 5. 实例的_container保存为当前rootContainer； mount('#app')
    app._container = rootContainer
    // 6. rootContainer增加属性__vue_app__，置为当前app实例；
    ;(rootContainer as any).__vue_app__ = app

    return vnode.component!.proxy
  }
}
```

上述代码的核心渲染代码为 render 函数。

### render

render 的函数目前只是负责任务分发的工作， 分发两个工作 unmount 和 patch

**packages/runtime-core/src/renderer.ts**

```javascript
// 1. vnode，是要更新到页面上的vnode，通过上面createVNode获得；container为展现的容器 ('#app')
const render: RootRenderFunction = (vnode, container) => {
  if (vnode == null) {
    // 2.如果 vnode 为空，并且container._vnode有值，也就是有之前的dom渲染，则进行unmount操作；
    if (container._vnode) {
      unmount(container._vnode, null, null, true)
    }
  } else {
    // 3. 如果vnode不为空，则进行patch操作，dom diff和渲染
    patch(container._vnode || null, vnode, container)
  }
  // 4. 执行flushPostFlushCbs函数，回调调度器，使用Promise实现，与Vue2的区别是Vue2是宏任务或微任务来处理的
  flushPostFlushCbs()
  // 5. 把container的_vnode存储为当前vnode，方便后面进行dom diff操作
  container._vnode = vnode
}
```

本文主要讲述的渲染，vnode 不会为空，肯定会走到 patch 函数部分。

**packages/runtime-core/src/renderer.ts**

```javascript
const patch: PatchFn = (
    n1, // old
    n2, // new
    container, // 容器
    anchor = null,
    parentComponent = null,
    parentSuspense = null,
    isSVG = false,
    optimized = false
) => {
    // 如果type不相同，则把n1直接卸载掉
    if (n1 && !
    (n1, n2)) {
        anchor = getNextHostNode(n1)
        unmount(n1, parentComponent, parentSuspense, true)
        n1 = null
    }

    if (n2.patchFlag === PatchFlags.BAIL) {
        optimized = false
        n2.dynamicChildren = null
    }

    const {type, ref, shapeFlag} = n2
    switch (type) {
        case Text:
            processText(n1, n2, container, anchor)
            break
        case Comment:
            processCommentNode(n1, n2, container, anchor)
            break
        case Static:
            if (n1 == null) {
                mountStaticNode(n2, container, anchor, isSVG)
            } else if (__DEV__) {
                patchStaticNode(n1, n2, container, isSVG)
            }
            break
        case Fragment:
            // 处理片段(dom数组)的函数
            processFragment(
                n1,
                n2,
                container,
                anchor,
                parentComponent,
                parentSuspense,
                isSVG,
                optimized
            )
            break
        default:
            if (shapeFlag & ShapeFlags.ELEMENT) {
                // 处理 dom 元素
                processElement(
                    n1,
                    n2,
                    container,
                    anchor,
                    parentComponent,
                    parentSuspense,
                    isSVG,
                    optimized
                )
            } else if (shapeFlag & ShapeFlags.COMPONENT) {
                // 处理组件
                processComponent(
                    n1,
                    n2,
                    container,
                    anchor,
                    parentComponent,
                    parentSuspense,
                    isSVG,
                    optimized
                )
            } else if (shapeFlag & ShapeFlags.TELEPORT) {
                ;(type as typeof TeleportImpl).process(
                    n1 as TeleportVNode,
                    n2 as TeleportVNode,
                    container,
                    anchor,
                    parentComponent,
                    parentSuspense,
                    isSVG,
                    optimized,
                    internals
                )
            } else if (__FEATURE_SUSPENSE__ && shapeFlag & ShapeFlags.SUSPENSE) {
                ;(type as typeof SuspenseImpl).process(n1, n2, container, anchor, parentComponent, parentSuspense, isSVG, optimized, internals)
            } else if (__DEV__) {
                warn('Invalid VNode type:', type, `(${typeof type})`)
            }
    }

    if (ref != null && parentComponent) {
        setRef(ref, n1 && n1.ref, parentComponent, parentSuspense, n2)
    }
}
```

![/imgs/mount/3.png](/imgs/mount/3.png)

## 示例代码分析

**hello.js**

```javascript
const app = Vue.createApp({
  data() {
    return {
      list: ['a', 'b', 'c', 'd'],
    }
  },
})
app.mount('#demo')
```

**hello.html**

```html
<!DOCTYPE html>
<html>
  <head>
    <meta
      name="viewport"
      content="initial-scale=1.0,maximum-scale=1.0,minimum-scale=1.0,
    user-scalable=no,target-densitydpi=medium-dpi,viewport-fit=cover"
    />
    <title>Vue3.js hello example</title>
    <script src="../../dist/vue.global.js"></script>
  </head>
  <body>
    <div id="demo">
      <ul>
        <li v-for="item in list" :key="item">{{item}}</li>
      </ul>
    </div>
    <script src="./hello.js"></script>
  </body>
</html>
```

代码运行后：

![/imgs/mount/4.png](/imgs/mount/4.png)

以下分析 patch 函数代码运行过程：

1. 当运行到 patch 方法时 `patch(n1, n2, container, anchor = null, parentComponent = null, parentSuspense = null, isSVG = false, optimized = false)` 其中 n1 即为 null，n2 即为要更新的 vnode，container 为 `#demo` 容器元素。
2. 初始化运行时 n1 为 null, n2 为要更新的 vnode

![/imgs/mount/5.png](/imgs/mount/5.png)
