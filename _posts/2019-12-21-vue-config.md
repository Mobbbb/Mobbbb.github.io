---
layout: post
title: Vue config 配置
subtitle: 
# gh-repo: daattali/beautiful-jekyll
# gh-badge: [star, fork, follow]
tags: [vue, share]
comments: true
---

![bg](../assets/img/posts/vue-config/1.jpg)

# 1. babel.config.js

### 1.1 浏览器兼容

- presets: array
  
  默认情况下，它会根据源代码中出现的语言特性自动检测需要的 polyfill。这确保了最终包里 polyfill 数量的最小化。然而，这也意味着如果其中一个依赖需要特殊的 polyfill，默认情况下 Babel 无法将其检测出来

  - 如果该依赖基于一个目标环境不支持的 ES 版本撰写: 将其添加到 vue.config.js 中的 ```transpileDependencies``` 
  选项(一般用来显式的转译modules依赖)。这会为该依赖同时开启语法转换和根据使用情况检测 polyfill。

  - 如果该依赖交付了 ES5 代码并显式地列出了需要的 polyfill: 你可以使用 @vue/babel-preset-app 的 polyfills 选项预包含所需要的 polyfill。注意 es6.promise 将被默认包含，因为现在的库依赖 Promise 是非常普遍的。

  - 如果该依赖交付 ES5 代码，但使用了 ES6+ 特性且没有显式地列出需要的 polyfill (例如 Vuetify)：请使用 useBuiltIns: 'entry' 然后在入口文件添加 import '@babel/polyfill'。这会根据 browserslist 目标导入所有 polyfill，这样你就不用再担心依赖的 polyfill 问题了，但是因为包含了一些没有用到的 polyfill 所以最终的包大小可能会增加。

  - useBuiltIns: 'usage'，cdn引入browsers-polyfill（官方不推荐）

   (我们推荐以上面的方式添加 polyfill 而不是在源代码中直接导入它们，因为如果这里列出的 polyfill 在 browserslist 的目标中不需要，则它会被自动排除)

  ```javascript
  module.exports = {
      presets: [
        ['@vue/app', {
          useBuiltIns: 'usage', // false: 不使用polyfill
          polyfills: ['es6.array.iterator', 'es6.promise', 'es6.object.assign', 'es7.promise.finally']
        }]
      ]
  }
  ```

# 2. vue.config.js

### 2.1 简单配置

* 旧baseUrl: '/' ——> 新publicPath: '/' 

  tips：请始终使用 ```publicPath ``` 而不要直接修改 webpack 的 ```output.publicPath```。

* outputDir: 'dist'

  tips：请始终使用 ```outputDir ``` 而不要直接修改 webpack 的 ```output.path```。

* assetsDir: ''

  tips：从生成的资源覆写 filename 或 chunkFilename 时，```assetsDir``` 会被忽略。

* indexPath: 'index.html'

  tips：指定生成的 ```index.html``` 的输出路径 (相对于 ```outputDir``` )。也可以是一个绝对路径。

* productionSourceMap: true

  生产环境使用 source map，可以将其设置为 false 以加速生产环境构建。

  开启：f12 -> settings -> enable js source maps

* pluginOptions: Object

  它可以用来传递任何第三方插件选项


### 2.2 常用配置

* pages

  单页应用无需配置，在 multi-page 模式下构建应用。每个“page”应该有一个对应的 JavaScript 入口文件。其值应该是一个对象。

  - entry: 'src/index/main.js', // page 的入口
      
  - template: 'public/index.html', // 模板来源
      
  - filename: 'index.html', // 在 dist/index.html 的输出
      
  - title: 'Index Page', // 当使用 title 选项时，

  - chunks: ['chunk-vendors', 'chunk-common'] // 在这个页面中包含的块，默认情况下会包含vendors和common

* configureWebpack: Object \| Function

  如果这个值是一个对象，则会通过 webpack-merge 合并到最终的配置中。

  如果这个值是一个函数，则会接收被解析的配置作为参数。该函数及可以修改配置并不返回任何东西，也可以返回一个被克隆或合并过的配置版本。

* chainWebpack: Function

  是一个函数，会接收一个基于 webpack-chain 的 ChainableConfig 实例。允许对内部的 webpack 配置进行更细粒度的修改。

  - 修改 Loader 选项

  ```javascript
  chainWebpack: config => {
    config.module
      .rule('vue')
      .use('vue-loader')
        .loader('vue-loader')
        .tap(options => {
          // 修改它的选项...
          return options
        })
  }
  ```

  - 修改插件选项

* devServer: Object

  所有 webpack-dev-server 的选项都支持。

  - port: 9000
  - compress: true
  - ```javascript
    proxy: {
      '/api': {
        target: '//10.0.10.11:8080',
        ws: true,
        changeOrigin: true,
        pathRewrite: {
          '^/api': '/'
        }
      },
    }
    ```

### 2.3 HTML与静态资源

* 插值
  
   ```<link rel="icon" href="<%= BASE_URL %>favicon.ico">```

* Preload

  用来指定页面加载后很快会被用到的资源，在页面加载的过程中，我们希望在浏览器开始主体渲染之前尽早 preload
  默认情况下，一个 Vue CLI 应用会为所有初始化渲染需要的文件自动生成 preload 提示

* Prefetch

  用来告诉浏览器在页面加载完成后，利用空闲时间提前获取用户未来可能会访问的内容。

* 从相对路径导入

  - 当你在 JavaScript、CSS 或 *.vue 文件中使用相对路径 (必须以 . 开头) 引用一个静态资源时，该资源将会被包含进入 webpack 的依赖图中。在其编译过程中，所有诸如 ```<img src="...">```、background: url(...) 和 CSS @import 的资源 URL 都会被解析为一个模块依赖。
  - 在其内部，通过 file-loader 用版本哈希值和正确的公共基础路径来决定最终的文件路径，再用 url-loader 将小于 4kb 的资源内联，以减少 HTTP 请求的数量

* public

  任何放置在 public 文件夹的静态资源都会被简单的复制，而不经过 webpack。你需要通过绝对路径来引用它们

  - 你需要在构建输出中指定一个文件的名字。
  - 你有上千个图片，需要动态引用它们的路径。
  - 有些库可能和 webpack 不兼容，这时你除了将其用一个独立的 ```<script>``` 标签引入没有别的选择


### 2.4 CSS相关

* PostCSS
  
  在生产环境构建中，Vue CLI 会优化 CSS 并基于目标浏览器抛弃不必要的浏览器前缀规则。因为默认开启了 autoprefixer，你只使用无前缀的 CSS 规则即可

* CSS Modules

  你可以通过 ```<style module>``` 以开箱即用的方式在 *.vue 文件中使用 CSS Modules。

  如果想在 JavaScript 中作为 CSS Modules 导入 CSS 或其它预处理文件，该文件应该以 .module.(css\|less\|sass\|scss\|styl) 结尾：

  ```javascript
  module.exports = {
    css: {
      requireModuleExtension: false, // 前css.modules，v4弃用
      extract: true,
      sourceMap: false,

      loaderOptions: {
        css: {
          localIdentName: '[name]-[hash]', // 自定义生成的 CSS Modules 模块的类名
          camelCase: 'only',
        },
        // 给 sass-loader 传递选项
        sass: {
          // 假设你有 `src/variables.sass` 这个文件
          // 注意：在 sass-loader v7 中，这个选项名是 "data"
          prependData: `@import "~@/variables.sass"`
        },
        // 默认情况下 `sass` 选项会同时对 `sass` 和 `scss` 语法同时生效
        // 因为 `scss` 语法在内部也是由 sass-loader 处理的
        // 但是在配置 `data` 选项的时候
        // `scss` 语法会要求语句结尾必须有分号，`sass` 则要求必须没有分号
        // 在这种情况下，我们可以使用 `scss` 选项，对 `scss` 语法进行单独配置
        scss: {
          prependData: `@import "~@/variables.scss";`
        },
        // 给 less-loader 传递 Less.js 相关选项
        less:{
          // http://lesscss.org/usage/#less-options-strict-units `Global Variables`
          // `primary` is global variables fields name
          globalVars: {
            primary: '#fff'
          }
        }
      }
    }
  }
  ```
