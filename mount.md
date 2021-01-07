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

3. 此时判断符合 `shapeFlag & ShapeFlags.COMPONENT` 执行 `processComponent(n1, n2, container, anchor, parentComponent, parentSuspense, isSVG, optimized)` 方法。 n1、n2 等参数同 patch 方法。

4. processComponent 方法

**packages/runtime-core/src/renderer.ts**

```javascript
const processComponent = (
    n1: VNode | null,
    n2: VNode,
    container: RendererElement,
    anchor: RendererNode | null,
    parentComponent: ComponentInternalInstance | null,
    parentSuspense: SuspenseBoundary | null,
    isSVG: boolean,
    optimized: boolean
  ) => {
    // 对 n1 的判断，n1不为null，则证明是更新操
    if (n1 == null) {
      if (n2.shapeFlag & ShapeFlags.COMPONENT_KEPT_ALIVE) {
        // 为 keepAlive 的类型的组件，走activate逻辑
        ;(parentComponent!.ctx as KeepAliveContext).activate(
          n2,
          container,
          anchor,
          isSVG,
          optimized
        )
      } else {
        // 首次渲染执行此方法
        mountComponent(
          n2,
          container,
          anchor,
          parentComponent,
          parentSuspense,
          isSVG,
          optimized
        )
      }
    } else {
      updateComponent(n1, n2, optimized)
    }
  }
```

由于是首次渲染我们调用 `mountComponent` 方法， 以下为 `mountComponent` 源码：

**packages/runtime-core/src/renderer.ts**

```javascript
const mountComponent: MountComponentFn = (
    initialVNode,
    container,
    anchor,
    parentComponent,
    parentSuspense,
    isSVG,
    optimized
  ) => {
    const instance: ComponentInternalInstance = (initialVNode.component =
    // 1. 调用createComponentInstance生成对当前n2的实例
    createComponentInstance(
      initialVNode,
      parentComponent,
      parentSuspense
    ))

    if (__DEV__ && (__BROWSER__ || __TEST__) && instance.type.__hmrId) {
      registerHMR(instance)
    }

    if (__DEV__) {
      pushWarningContext(initialVNode)
      startMeasure(instance, `mount`)
    }

    // inject renderer internals for keepAlive
    if (isKeepAlive(initialVNode)) {
      ;(instance.ctx as KeepAliveContext).renderer = internals
    }

    // resolve props and slots for setup context
    if (__DEV__) {
      startMeasure(instance, `init`)
    }
    // 2. 初始化 props 和 slots
    setupComponent(instance)
    if (__DEV__) {
      endMeasure(instance, `init`)
    }

    // setup() is async. This component relies on async logic to be resolved
    // before proceeding
    if (__FEATURE_SUSPENSE__ && instance.asyncDep) {
      parentSuspense && parentSuspense.registerDep(instance, setupRenderEffect)

      // Give it a placeholder if this is not hydration
      // TODO handle self-defined fallback
      if (!initialVNode.el) {
        const placeholder = (instance.subTree = createVNode(Comment))
        processCommentNode(null, placeholder, container!, anchor)
      }
      return
    }

    // 3. instance为上面生成的实例，initialVNode还是为上图n2,container为#demo，其他为默认值
    setupRenderEffect(
      instance,
      initialVNode,
      container,
      anchor,
      parentSuspense,
      isSVG,
      optimized
    )

    if (__DEV__) {
      popWarningContext()
      endMeasure(instance, `mount`)
    }
  }
```

setupComponent 实现逻辑如下

**packages/runtime-core/src/component.ts**

```javascript
export function setupComponent(
  instance: ComponentInternalInstance,
  isSSR = false
) {
  isInSSRComponentSetup = isSSR

  const { props, children, shapeFlag } = instance.vnode
  const isStateful = shapeFlag & ShapeFlags.STATEFUL_COMPONENT
  initProps(instance, props, isStateful, isSSR)
  initSlots(instance, children)

  const setupResult = isStateful
    ? setupStatefulComponent(instance, isSSR)
    : undefined
  isInSSRComponentSetup = false
  return setupResult
}
```

setupRenderEffect 实现逻辑如下：

setupRenderEffect 函数是一个非常核心的函数， 此函数将会为当前实例挂载上 update 方法，update 方法是通过 effect 生成的， effect 在 Vue3 中的作用就相当于 Vue2 中的 observe；update 生成后，挂载之前会先运行一下生成的 effect 方法，最后返回当前 effect 方法给 update；运行 effect 函数就相当于 Vue2 中 watcher 调用 get 的过程.effect 接受两个参数，第一个参数就是 componentEffect 函数，也就是监听变化调用此函数；上面讲到先运行一下生成的 effect 方法，生成的 effect 方法内部就会调用这个 componentEffect 函数。

**packages/runtime-core/src/renderer.ts**

```javascript
const setupRenderEffect: SetupRenderEffectFn = (
    instance,
    initialVNode,
    container,
    anchor,
    parentSuspense,
    isSVG,
    optimized
  ) => {
    // create reactive effect for rendering
    instance.update = effect(function componentEffect() {
      // instance.isMounted；如果已经渲染，则走更新逻辑；则走未渲染的逻辑
      if (!instance.isMounted) {
        let vnodeHook: VNodeHook | null | undefined
        const { el, props } = initialVNode
        const { bm, m, parent } = instance

        // beforeMount hook
        // 1. 先调用了当前实例的beforeMount钩子函数
        if (bm) {
          invokeArrayFns(bm)
        }
        // onVnodeBeforeMount
        // 2.调用n2的父类的BeforeMount钩子函数
        if ((vnodeHook = props && props.onVnodeBeforeMount)) {
          invokeVNodeHook(vnodeHook, parent, initialVNode)
        }

        // render
        if (__DEV__) {
          startMeasure(instance, `render`)
        }
        // 3. 调用renderComponentRoot函数进行渲染组件的根元素
        const subTree = (instance.subTree = renderComponentRoot(instance))
        if (__DEV__) {
          endMeasure(instance, `render`)
        }

        if (el && hydrateNode) {
          if (__DEV__) {
            startMeasure(instance, `hydrate`)
          }
          // vnode has adopted host node - perform hydration instead of mount.
          hydrateNode(
            initialVNode.el as Node,
            subTree,
            instance,
            parentSuspense
          )
          if (__DEV__) {
            endMeasure(instance, `hydrate`)
          }
        } else {
          if (__DEV__) {
            startMeasure(instance, `patch`)
          }
          // 4. 调用patch  subtree 为上面生成的根 node
          patch(
            null,
            subTree,
            container,
            anchor,
            instance,
            parentSuspense,
            isSVG
          )
          if (__DEV__) {
            endMeasure(instance, `patch`)
          }
          initialVNode.el = subTree.el
        }
        // mounted hook
        // 5.调用当前实例的mounted钩子函数；；
        if (m) {
          queuePostRenderEffect(m, parentSuspense)
        }
        // onVnodeMounted
        // 6.调用n2的父类的mounted钩子函数；调用当前实例的activated钩子函数；不是直接调用，而是通过queuePostRenderEffect放到队列中去调用
        if ((vnodeHook = props && props.onVnodeMounted)) {
          queuePostRenderEffect(() => {
            invokeVNodeHook(vnodeHook!, parent, initialVNode)
          }, parentSuspense)
        }
        // activated hook for keep-alive roots.
        // #1742 activated hook must be accessed after first render
        // since the hook may be injected by a child keep-alive
        const { a } = instance
        if (
          a &&
          initialVNode.shapeFlag & ShapeFlags.COMPONENT_SHOULD_KEEP_ALIVE
        ) {
          queuePostRenderEffect(a, parentSuspense)
        }
        // 最终把实例的isMounted置为true
        instance.isMounted = true
      } else {
        // updateComponent
        // This is triggered by mutation of component's own state (next: null)
        // OR parent calling processComponent (next: VNode)
        let { next, bu, u, parent, vnode } = instance
        let originNext = next
        let vnodeHook: VNodeHook | null | undefined
        if (__DEV__) {
          pushWarningContext(next || instance.vnode)
        }

        if (next) {
          next.el = vnode.el
          updateComponentPreRender(instance, next, optimized)
        } else {
          next = vnode
        }

        // beforeUpdate hook
        if (bu) {
          invokeArrayFns(bu)
        }
        // onVnodeBeforeUpdate
        if ((vnodeHook = next.props && next.props.onVnodeBeforeUpdate)) {
          invokeVNodeHook(vnodeHook, parent, next, vnode)
        }

        // render
        if (__DEV__) {
          startMeasure(instance, `render`)
        }
        const nextTree = renderComponentRoot(instance)
        if (__DEV__) {
          endMeasure(instance, `render`)
        }
        const prevTree = instance.subTree
        instance.subTree = nextTree

        if (__DEV__) {
          startMeasure(instance, `patch`)
        }
        patch(
          prevTree,
          nextTree,
          // parent may have changed if it's in a teleport
          hostParentNode(prevTree.el!)!,
          // anchor may have changed if it's in a fragment
          getNextHostNode(prevTree),
          instance,
          parentSuspense,
          isSVG
        )
        if (__DEV__) {
          endMeasure(instance, `patch`)
        }
        next.el = nextTree.el
        if (originNext === null) {
          // self-triggered update. In case of HOC, update parent component
          // vnode el. HOC is indicated by parent instance's subTree pointing
          // to child component's vnode
          updateHOCHostEl(instance, nextTree.el)
        }
        // updated hook
        if (u) {
          queuePostRenderEffect(u, parentSuspense)
        }
        // onVnodeUpdated
        if ((vnodeHook = next.props && next.props.onVnodeUpdated)) {
          queuePostRenderEffect(() => {
            invokeVNodeHook(vnodeHook!, parent, next!, vnode)
          }, parentSuspense)
        }

        if (__DEV__ || __FEATURE_PROD_DEVTOOLS__) {
          devtoolsComponentUpdated(instance)
        }

        if (__DEV__) {
          popWarningContext()
        }
      }
    }, __DEV__ ? createDevEffectOptions(instance) : prodEffectOptions)
  }
```

上面 componentEffect 函数中调用 patch 才是正式渲染的开始，前面大部分都是相当于数据的整理：

1. 按照上面 componentEffect 函数的运行参数传递到 patch 函数：patch(n1, n2, container, anchor, parentComponent, parentSuspense, isSVG, optimized)；此时 n1 为 null，n2 为上图(subtree)，container 还是#demo，anchor 为 null，parentComponent 为上面的 instance 实例，parentSuspense 为 null，isSVG 为 false，optimized 为 false。
2. 代码依次执行，通过上图可以看到，component 获取到的实例的 subtree 的 type 为 Fragment，则会走到 processFragment 函数。

processFragment 的代码如下：

```javascript
  const processFragment = (
      n1: VNode | null,
      n2: VNode,
      container: RendererElement,
      anchor: RendererNode | null,
      parentComponent: ComponentInternalInstance | null,
      parentSuspense: SuspenseBoundary | null,
      isSVG: boolean,
      optimized: boolean
  ) => {
      const fragmentStartAnchor = (n2.el = n1 ? n1.el : hostCreateText(''))!
      const fragmentEndAnchor = (n2.anchor = n1 ? n1.anchor : hostCreateText(''))!
      let {patchFlag, dynamicChildren} = n2
      if (patchFlag > 0) {
          optimized = true
      }
      if (n1 == null) {
          hostInsert(fragmentStartAnchor, container, anchor)
          hostInsert(fragmentEndAnchor, container, anchor)
          mountChildren(
              n2.children as VNodeArrayChildren,
              container,
              fragmentEndAnchor,
              parentComponent,
              parentSuspense,
              isSVG,
              optimized
          )
      } else {
      	 // 其他逻辑
      }
  }
```

根据参数可以知道会走到当前 if 逻辑，会先插入骨架；然后执行 mountChildren，n2.children 通过上面的 subtree 可以知道，值为一个数组，数组里面有 1 个元素，就是咱们要渲染的 ul。

```javascript
const mountChildren: MountChildrenFn = (
    children,
    container,
    anchor,
    parentComponent,
    parentSuspense,
    isSVG,
    optimized,
    start = 0
) => {
    for (let i = start; i < children.length; i++) {
        const child = (children[i] = optimized
            ? cloneIfMounted(children[i] as VNode)
            : normalizeVNode(children[i]))
        patch(
            null,
            child,
            container,
            anchor,
            parentComponent,
            parentSuspense,
            isSVG,
            optimized
        )
    }
}
```

可以看到将会对 n2.children 进行遍历，n2.children 只有一个元素，是 ul。

4. 使用上面的运行时的参数，调用 patch(n1, n2, container, anchor, parentComponent, parentSuspense, isSVG = false, optimized)；参数：n1 为 null；child 为上面提到的 ul；container 为#demo，anchor 为上面 processFragment 函数里面的 fragmentEndAnchor；parentComponent 为 instance 实例；parentSuspense 为 null；isSVG 为 false；optimized 为 true，因为在上面 processFragment 里面进行了改变。

5. 由上面参数可知，ul 的类型为 ul，此时会走到 processElement 函数，processElement 函数的参数和 patch 函数的参数是一样的，进行了透传。

```javascript
const processElement = (
    n1: VNode | null,
    n2: VNode,
    container: RendererElement,
    anchor: RendererNode | null,
    parentComponent: ComponentInternalInstance | null,
    parentSuspense: SuspenseBoundary | null,
    isSVG: boolean,
    optimized: boolean
) => {
    isSVG = isSVG || (n2.type as string) === 'svg'
    if (n1 == null) {
        mountElement(
            n2,
            container,
            anchor,
            parentComponent,
            parentSuspense,
            isSVG,
            optimized
        )
    } else {
        patchElement(n1, n2, parentComponent, parentSuspense, isSVG, optimized)
    }
}
```

根据参数 n1 为 null 可以知晓，会走到 mountElement 的逻辑，参数不会发生改变。执行 mountElement 的过程中会检测 ul 的 children，发现 ul 的 children 下面有值，则会调用 mountChildren 函数。

```javascript
mountChildren(
    vnode.children as VNodeArrayChildren,
    el,
    null,
    parentComponent,
    parentSuspense,
    isSVG && type !== 'foreignObject',
    optimized || !!vnode.dynamicChildren
)
```

此时 vnode.children 为由 4 个 li 组成的数组；el 为 ul，anchor 为 null，parentComponent 为 instance 实例；parentSuspense 为 null；isSVG 为 false；optimized 为 true；重复上面 mountChildren 函数；

mountChildren 函数里面进行 for 循环的时候，li 的 type 为 li，则会继续走到 processElement,重复上面步骤，依次执行完成。

上面所有的步骤执行完成，现在数据已经呈现到页面上，此时基本所有的事情都干完了，也就是相当于主队列空闲了，调用 flushPostFlushCbs()开始执行队列里面的函数，最后把 container 的\_vnode 属性指向当前 vnode；方便下次做 dom diff 使用， 第一次渲染运行完成。
