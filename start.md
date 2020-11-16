# vue3 源码分析步骤 及 createApp 方法分析

## 前言

vue 3.0 相较于 vue2.0 主要针对 config api 改为 composition api、typescript 的支持、vue 的内部结构被重写成了一个个可以解耦的模块（允许编译后 tree-shaking），从而减小体积加快运行速度。

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
```
### vue3.0

## createApp 源码分析

### createApp

### createRenderer

### mount
