---
title: pwa改造之路
date: 2018-06-10 23:19:28
tags:
---

# 背景

公司移动端的项目使用`pwa`进行了改造，从原来的服务端直接模板渲染换成了前后端分离的纯前端渲染模式，再加上`pwa`进行一些缓存的加持(目前主要使用了`pwa`的缓存，其他的一些推送通知之类的东西还没有添加，以后有时间还是需要继续的)，感觉效果还是挺好的，主要技术栈使用了`vue`的全家桶，也遇到了一些`vue`的坑...

## `service-worker.js`文件目录问题

我的项目是通过`vue-cli`的`pwa`模板创建的，这东西创建一个项目起来还是很简单，但是，这个项目的模板也是有坑的，最大的坑就是他的`service-worker.js`文件中的一段代码

```js
(window.location.protocol === 'https:' || isLocalhost)) {
    navigator.serviceWorker.register('service-worker.js')
        .then(function (registration) {
```

这个`service-worker.js`是一个相对路径，导致只有页面在根目录下的时候才能将项目添加到首屏幕，其他的路径下都加不成功，因为只有根目录下才能下载成功`service-worker.js`这个文件，这个文件在`build`之后就在项目的根目录下，所以需要稍作修改成绝对路径即可

```js
(window.location.protocol === 'https:' || isLocalhost)) {
    navigator.serviceWorker.register('/service-worker.js')
        .then(function (registration) {
```

## 文件缓存问题

`vue-cli`创建的项目使用了一个`sw-precache-webpack-plugin"`插件来做`precache`，但是也就只能做`prechache`，我还想要要做一些`runtime cache`这个插件是做不到的，所以基本上这个插件是废了

换一个插件名字叫做`workbox`,这个东西是对`sw-precache`和`sw-toolbox`的封装，都是`google`的，所以质量还是可以的，省去了对底层的`sw`接口的调用，更加方便...

具体`workbox`用法，就去查文档，列一下基础的东西

关于`precache`配置：

```js
new WorkboxPlugin.InjectManifest({
  swDest: "service-worker.js",
  swSrc: path.resolve(__dirname, "sw-prod.js"),
  exclude: [/\.(?:png|jpg|jpeg|svg)$/, /\.map$/, ".DS_Store"]
})
```

其他配置就写在`sw-prod.js`文件中即可，也就是写一写缓存策略等，其他的都已经封装好了，使用非常简单

关于缓存的策略，直接参考`workbox`的[文档](https://developers.google.com/web/tools/workbox/)就可以，说的都很清楚的，

### 缓存策略主要需要注意跨域请求的缓存

对于跨域请求，默认是需要使用`networkFirst`策略的，这种情况下是为了避免错误导致缓存无法被更新。如果不想用，那需要在其他策略中指定请求状态码，这样避免缓存了失败的请求结果

还有对于跨域的地址的正则表达式需要从头开始，例如

```js
workbox.routing.registerRoute(
  new RegExp('^https://d1n1fwner8zc14.cloudfront.net/static/img/'),
  ...
)
```

表达式开头的 `^` 最好不要省略，否则可能导致无法缓存请求

最后说一下，最好给`cacheFirst`缓存策略设置缓存时间和缓存的大小，也就是缓存的数量，这样可以防止缓存无法过期和占用空间过多的问题

## manifest 的问题

默认生成的`manifest.json`文件很简单， 不可配置，还缺少一些图标等，为了解决这样问题， 也找了些`webpack`插件，但是都有点问题，有些就是配置不全，还不能自己加，所以就自己写了一个，这样就方便多了，自动裁剪不同尺寸的图标，非常方便。[插件地址](https://github.com/lujin123/html-webpack-manifest-plugin)
