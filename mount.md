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
