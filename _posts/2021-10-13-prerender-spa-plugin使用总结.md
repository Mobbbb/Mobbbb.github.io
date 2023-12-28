---
layout: post
title: prerender-spa-plugin使用总结
subtitle: 
# gh-repo: daattali/beautiful-jekyll
# gh-badge: [star, fork, follow]
tags: ["vue", "prerender-spa-plugin", 总结]
comments: true
---

![bg](../assets/img/posts/vue-config/1.jpg)

### 1. 介绍
 > prerender-spa-plugin 利用了 Puppeteer 的爬取页面的功能。 Puppeteer 是一个 Chrome官方出品的 headlessChromenode 库。它提供了一系列的 API, 可以在无 UI 的情况下调用 Chrome 的功能, 适用于爬虫、自动化处理等各种场景。它很强大，所以很简单就能将运行时的 HTML 打包到文件中。原理是在 Webpack 构建阶段的最后，在本地启动一个 Puppeteer 的服务，访问配置了预渲染的路由，然后将 Puppeteer 中渲染的页面输出到 HTML 文件中，并建立路由对应的目录

### 2. Unable to prerender all routes!
- 1、由于`webpack5: mkdirp is no longer expected to be a function on the output file system`，如果当前使用的是webpack5版本，需更新插件代码。
- 2、作者提交的修复版本的分支 [https://github.com/jsbugwang/prerender-spa-plugin/tree/webpack5](https://github.com/jsbugwang/prerender-spa-plugin/tree/webpack5)
- 3、ps：该分支漏了 `const fs = require('fs')` 的声明，需自行加上
- 4、ps：可在 `perrender-spa-plugin/es6/index.js` 中，代码 `console.error(msg)` 处添加 `console.log(err)` 输出具体报错原因，以定位问题

### 3. pyppeteer.errors.TimeoutError: Navigation Timeout Exceeded: 30000 ms.
- 1、预渲染的路由过多，超过pyppeteer默认的超时时间，减少打包的路由
- 2、设置 `maxConcurrentRoutes: 10`

### 4. 设置publicPath二级目录后，预渲染白屏
- 1、`publicPath: process.env.NODE_ENV === 'production' ? '/vue' : '/'`
- 2、`outputDir: path.join(__dirname, 'dist', '/vue')`
- 3、参考链接 [https://github.com/chrisvfritz/prerender-spa-plugin/issues/344](https://github.com/chrisvfritz/prerender-spa-plugin/issues/344) #takatama, #cnvoa 的评论
具体配置如下:
```javascript
new PrerenderSPAPlugin({
    indexPath: path.join(__dirname, 'dist/vue/index.html'),
    // Required - The path to the webpack-outputted app to prerender.
    staticDir: path.join(__dirname, 'dist'),
    // Optional - The path your rendered app should be output to.
    // (Defaults to staticDir.)
    outputDir: path.join(__dirname, 'dist/vue'),
    // Required - Routes to render.
    routes: ['/'],
    renderer: new Renderer({
        headless: true, // 是否隐藏无头浏览器(调试时可开启)
        renderAfterTime: 5000,
    }),
    server: {
        proxy: {
            '/api': {
                target: 'http://www.a.c',
                changOrigin: true,
                secure: false,
            },
        },
    },
})
```

### 5. 预渲染未请求接口数据(一)
- 1、在配置插件中增加 `headless: false` ，用以开启无头浏览器进行调试，查看console面板中的错误
- 2、定位到问题出在发起的接口请求中，打印返回内容为html文本
- 3、打印查看接口请求地址，为 `http://localhost:8080/api/xxx` ，原因是代码中接口请求路径为缺省域名的写法，请求时会自动补全当前网页所在的域名
- 4、更改代码中的请求，补全请求地址，如：`http://www.a.c/api/xxx` (由于此做法会有跨域问题，并未采用)
- 5、查看PrerenderSPAPlugin源码发现其使用的服务为express框架，并用到了http-proxy-middleware反向代理，所以只要在插件中增加配置 server 即可解决，具体配置方法与http-proxy-middleware一致

### 6. 预渲染未请求接口数据(二)
- 1、在配置插件中增加 `headless: false` ，用以开启无头浏览器进行调试，查看console面板中的错误
- 2、接口请求返回500错误，打包界面提示 `ERR_TLS_CERT_ALTNAME_INVALID` 错误代码
- 3、定位问题为https证书验证的问题
- 4、首先确认nginx中的https配置ssl协议是否如下 `ssl_protocols: TLSv1 TLSv1.1 TLSv1.2`
- 5、在proxy中增加配置信息 `secure: false` 以关闭ssl证书验证
具体配置如下:
```javascript
new PrerenderSPAPlugin({
    // Required - The path to the webpack-outputted app to prerender.
    staticDir: path.join(__dirname, 'dist'),
    // Optional - The path your rendered app should be output to.
    // (Defaults to staticDir.)
    outputDir: path.join(__dirname, 'dist'),
    // Required - Routes to render.
    routes: ['/'],
    renderer: new Renderer({
        headless: true, // 是否隐藏无头浏览器(调试时可开启)
        renderAfterTime: 5000,
    }),
    server: {
        proxy: {
            '/api': {
                target: 'http://www.a.c',
                changOrigin: true,
                secure: false,
            },
        },
    },
})
```

### 7. 使用限制
 > prerender-spa-plugin 仅适用于 history 模式，若为 hash 模式，那么 routes 配置只有根路由可被成功预渲染。

## References
[1] [https://github.com/chrisvfritz/prerender-spa-plugin](https://github.com/chrisvfritz/prerender-spa-plugin)

[2] [https://github.com/jsbugwang/prerender-spa-plugin/tree/webpack5](https://github.com/jsbugwang/prerender-spa-plugin/tree/webpack5)