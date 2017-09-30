# 最全的vue-cli项目下的配置简析

tips: 本文主要介绍了利用原生Javascript实现了异步加载，及跨域请求JSONP的实现方式.以下方法也在日常生产中使用，由于博客服务有时候波动访问不了，建议先收藏到Github,仓库地址欢迎给个`★`(star)：

<a style="color:#16bc5a" href="https://github.com/WeideMo/vuecli-webpack-configs">『https://github.com/WeideMo/vuecli-webpack-configs』</a>

另外，如果在使用vue开发的过程中，出现了一些莫名其妙的问题，也可以参考一下我前一篇文章[`Vuejs渡劫系列一：日常开发中必须掌握的细节（keng）`](https://moweide.com/2017/09/02/vue_started/)，相信你也会有很多收获。

## 准备
### 安装命令行工具（CLI）

Vue.js提供了很人性化的命令行工具，在全局安装了依赖以后，便能轻松的在文件系统任意角落利用脚手架创建并启动一个配备了带热重载、保存时静态检查以及可用于生产环境的构建配置的项目，具体命令如下：

```bash
  # 全局安装 vue-cli
  $ npm install --global vue-cli
  # 创建一个基于 webpack 模板的新项目
  $ vue init webpack my-project
  # 安装依赖
  $ cd my-project
  $ npm install
  $ npm run dev
```
### 查看安装结果

```bash
  vue -V
  2.8.1
```

## 项目组成
### 目录结构


    ├── build
    │   ├── build.js
    │   ├── check-versions.js
    │   ├── dev-client.js
    │   ├── dev-server.js
    │   ├── utils.js
    │   ├── vue-loader.conf.js
    │   ├── webpack.base.conf.js
    │   ├── webpack.dev.conf.js
    │   ├── webpack.test.conf.js
    │   └── webpack.prod.conf.js
    ├── config
    │   ├── dev.env.js
    │   ├── index.js
    │   ├── test.env.js
    │   └── prod.env.js
    ├── node_modules    
    ├── src
    │   ├── App.vue
    │   ├── assets
    │   │   └── img.png
    │   ├── components
    │   │   └── Hello.vue
    │   └── main.js
    ├── static
    ├── test
    │   └── unit
    │   │   ├─── coverage
    │   │   ├─── specs
    │   │   ├─── .eslintrc
    │   │   ├─── index.js
    │   │   └── karma.conf.js
    │   └── e2e
    ├── .babelrc
    ├── .editorconfig
    ├── .gitignore
    ├── .postcssrc.js
    ├── index.html
    ├── package.json
    └── README.md

## /build
### build.js

在执行 `npm run build` 的时候，执行的js脚本，主要是加载了生产环境下的webpack配置，并根据webpack配置重新构建出生产环境下使用的压缩项目，最终资源文件会生成到根目录下 `dist` 文件中。

### check-version.js
在build之前的一个check步骤，作为构建生产项目包之前的一个校验处理。

### dev-client.js

```js
  require('eventsource-polyfill')
  var hotClient = require('webpack-hot-middleware/client?noInfo=true&reload=true')

  hotClient.subscribe(function (event) {
    if (event.action === 'reload') {
      window.location.reload()
    }
  })
```
加载了热重载插件，并在监听到事件的 `action` 变化为 `reload` 的时候，实现网页reload。

### dev-server.js

```js
  var app = express();
  ...
  var hotMiddleware = require('webpack-hot-middleware')(compiler, {
    log: false,
    heartbeat: 2000
  })
```

利用了express实现了一个本地开发的服务，同时配置了热重载功能以配合本地开发浏览器端自动reload刷新。

### utils.js

`utils.js`主要是对外给响应的webpack配置使用的工具方法，包含了对外提供静态资源asset路径：

```js
  exports.assetsPath = function (_path) {
    var assetsSubDirectory = process.env.NODE_ENV === 'production'
      ? config.build.assetsSubDirectory
      : config.dev.assetsSubDirectory
    return path.posix.join(assetsSubDirectory, _path)
  }
```

对外提供各种loaders的配置，如cssLoaders, css/postcss/less/sass/scss/stylus/styl等预处理或后处理器配置等：

```js
    return {
    css: generateLoaders(),
    postcss: generateLoaders(),
    less: generateLoaders('less'),
    sass: generateLoaders('sass', { indentedSyntax: true }),
    scss: generateLoaders('sass'),
    stylus: generateLoaders('stylus'),
    styl: generateLoaders('stylus')
  }
```

### vue-loader.conf.js

顾名思义，该文件就是为vue单文件所用的配置，主要是一些单文件内使用css预处理的配置，还有一些文件资源的转化配置项。

### webpack.base.conf.js

webpack的通用配置文件，即无论环境是 `test`,`develop`还是 `production`，都需要加载的webpack配置，该文件很重要，也时长需要修改配置：

> 入口文件 entry

```js
  entry: {
    app: './src/main.js',
    utils:['./src/assets/libs/jquery.js']
  }
```

默认只有app为入口的`main.js`，utils为自定义入口，通过配置多个文件入口，可以减少build后的文件大小，同时也可以配合 `CommonsChunkPlugin`插件进行项目分包。

> 输出文件output

config的配置在config/index.js文件中

```js
output: {
  path: config.build.assetsRoot, //导出目录的绝对路径
  filename: '[name].js', //导出文件的文件名
  publicPath: process.env.NODE_ENV === 'production'? config.build.assetsPublicPath : config.dev.assetsPublicPath //生产模式或开发模式下html、js等文件内部引用的公共路径
}
```

> 文件解析resolve

主要设置模块如何被解析。

```js
resolve: {
  extensions: ['.js', '.vue', '.json'], //自动解析确定的拓展名,使导入模块时不带拓展名
  alias: {   // 创建import或require引入时，文件使用的别名
    'vue$': 'vue/dist/vue.esm.js', 
    '@': resolve('src')
  }
}
```
其中默认定义了`vue$`为 `vue.esm.js`文件的别名， `@`为 `/src`目录的别名，通过 `@/xxx.js` 或者 `@/xxx.vue`可以快速引入文件，注意vue文件内只有.js和.vue文件可以使用@去寻址，scss或者less等文件是不被允许的使用`@`去找寻文件的。

> 模块解析机制 module

该模块主要用于定义不同文件后缀的文件，应该使用何种loaders（加载器）进行加载解析，与webpack 1.x不同的是，原来的 `loaders` 被替换为 `rules`;这里需要注意，默认的vue-cli已经为你配备了`.vue`,`.js`,`img类`,`媒体类`,`字体类`等的文件的解析，同时，`.vue`组件中如需用到的预处理语言，则需要在 `<style>`标签上增加 `lang`属性即可，如

```html
  <style lang="scss" scoped>
          body{
            background-color:#FFF;
          }
  </style>
```
### webpack.dev.conf.js

> 通过merge方法合并webpack.base.conf.js基础配置

```js
var merge = require('webpack-merge')
var baseWebpackConfig = require('./webpack.base.conf')
module.exports = merge(baseWebpackConfig, {})
```

> 模块配置

```js
module: {
  //通过传入一些配置来获取rules配置，此处传入了sourceMap: false,表示不生成sourceMap
  rules: utils.styleLoaders({ sourceMap: config.dev.cssSourceMap }) 
}
```

这里引用了刚才utils.js中styleLoaders方法，自动生成的配置如下

```js
exports.styleLoaders = function (options) {
   //定义输出的数组
  var output = []
  // 调用cssLoaders方法返回各类型的样式对象(css: loader)
  var loaders = exports.cssLoaders(options) 
  //循环遍历loaders
  for (var extension in loaders) {  
    //根据遍历获得的key(extension)来得到value(loader)
    var loader = loaders[extension] 
    output.push({     
      // 通过正则匹配判断各类型的样式文件，并生成test和对应loaders
      test: new RegExp('\\.' + extension + '$'), 
      use: loader
    })
  }
  return output
}
```

上面的代码中调用了exports.cssLoaders(options),主要用于对各类css预处理的loaders实现,具体实现如下

```js
exports.cssLoaders = function (options) {
  options = options || {}

  var cssLoader = { 
    loader: 'css-loader',
    options: {  
      //生成环境下压缩文件
      minimize: process.env.NODE_ENV === 'production', 
      //根据参数是否生成sourceMap文件
      sourceMap: options.sourceMap  
    }
  }
  function generateLoaders (loader, loaderOptions) {  
    // 预置css-loader
    var loaders = [cssLoader] 
    // 如果参数loader存在
    if (loader) { 
      //如果配置的预处理需要，则push一个进入loaders数组
      loaders.push({
        loader: loader + '-loader',
        options: Object.assign({}, loaderOptions, { 
          sourceMap: options.sourceMap
        })
      })
    }
    // 如果传入的options存在extract且为true
    if (options.extract) { 
      //通过使用ExtractTextPlugin插件分离js中引入的css文件
      return ExtractTextPlugin.extract({  
        use: loaders,  
        //没有被提取分离的使用vue-style-loader加载
        fallback: 'vue-style-loader' 
      })
    } else {
        //如果没有传入的options存在extract或为false时，统一使用vue-style-loader处理
      return ['vue-style-loader'].concat(loaders)
    }
  }
  return {  //返回css类型对应的loader组成的对象 generateLoaders()来生成loader
    css: generateLoaders(),
    postcss: generateLoaders(),
    less: generateLoaders('less'),
    sass: generateLoaders('sass', { indentedSyntax: true }),
    scss: generateLoaders('sass'),
    stylus: generateLoaders('stylus'),
    styl: generateLoaders('stylus')
  }
}
```

> 插件配置

```js
plugins: [
  // 编译时配置的全局变量
  new webpack.DefinePlugin({ 
    //当前环境为开发环境
    'process.env': config.dev.env 
  }),
  //热重载插件
  new webpack.HotModuleReplacementPlugin(), 
  //不触发错误,即编译后运行的包正常运行
  new webpack.NoEmitOnErrorPlugin(), 
  //自动生成html文件
  new HtmlWebpackPlugin({  
    filename: 'index.html', //生成的文件名
    template: 'index.html', //模板
    inject: true
  }),
  //友好的错误提示
  new FriendlyErrorsPlugin() 
]
```
### webpack.prod.conf.js

该文件是生产环境打包用到的配置文件，同样使用merge的方式合并了基础配置，形成了生产要用的配置：

> 判断环境

```js
  //判断环境，如果是testing的话，则加载test.env
var env = process.env.NODE_ENV === 'testing'
  ? require('../config/test.env')
  : config.build.env
```

> 加载样式处理器

```js
module: {
  //同样使用了utils.styleLoaders的方法处理，这里不赘述
    rules: utils.styleLoaders({
      sourceMap: config.build.productionSourceMap,
      extract: true
    })
  },
  //通过判断配置值，判断是否生产source-map
  devtool: config.build.productionSourceMap ? '#source-map' : false
  }
```

> 输出文件

```js
  output: {
    //配置的生产资源路径
    path: config.build.assetsRoot,
    //生成的文件名，一般会生成入口名称.[hash].js，如app.7d0bcfcc47ab773ebe20834b27a0927a.js
    filename: utils.assetsPath('js/[name].[chunkhash].js'),
    //生成的异步文件块，一般是分配id.[hash].js，如0.app.7d0bcfcc47ab773ebe20834b27a0927a.js
    chunkFilename: utils.assetsPath('js/[id].[chunkhash].js')
  }
```
注意，这里的异步文件块，系统会根据自己在 `router` 下 `index.js` 当中引入的异步组件自动生成，所以我们无需重命名这部分的文件，否则容易导致文件索引出错，出现404错误。

> 插件

```js
  //这里定义全局环境为生产
  new webpack.DefinePlugin({
      'process.env': env
    }),
  //js压缩插件
  new webpack.optimize.UglifyJsPlugin({
    compress: {
      //不显示警告
      warnings: false
    },
    //生产source-map
    sourceMap: true
  }), 
  //在css文件单独分离出来
  new ExtractTextPlugin({
      //生成的文件名，一般会生成入口名称.[hash].css，如app.7d0bcfcc47ab773ebe20834b27a0927a.css
      filename: utils.assetsPath('css/[name].[contenthash].css')
    }),
    //css配置插件，可以提取并压缩css文件
  new OptimizeCSSPlugin({
      cssProcessorOptions: {
        safe: true
      }
    }),
  //CommonsChunkPlugin 公共块提取插件
   new webpack.optimize.CommonsChunkPlugin({
     //配置生成的文件名称：vendor
      name: 'vendor',
      minChunks: function (module, count) {
       //这里是默认的使用方法，将node_module引用到打包在一起
        return (
          //正则匹配
          module.resource &&
          /\.js$/.test(module.resource) &&
          module.resource.indexOf(
            path.join(__dirname, '../node_modules')
          ) === 0
        )
      }
    }),
```
注意，这里通过 `CommonsChunkPlugin` 插件将 vue, vuex等包文件统一打在了名为:vendor的js中，最后会生成一个类似叫 `vendor.7d0bcfcc47ab773ebe20834b27a0927a.js`,当然为了防止app打包的hash的每次变化，导致资源无法缓存，我们需要增加一个 `manifest`，去将运行时的代码单独编译到manifest文件，以防每次编译都导致vendor.js的hash改变：

```js
  new webpack.optimize.CommonsChunkPlugin({
      name: 'manifest',
      chunks: ['vendor']
    }),
```

有时候，我们需要将我们引入的第三发库js，单独打包另外一个文件，假如叫utils.js，我们同样可以使用`CommonsChunkPlugin` 进行处理：

```js
   new webpack.optimize.CommonsChunkPlugin({
      names: ["utils"]
    })
```
别忘了我们需要为它加入manifest处理，只需在已有的基础上简单修改：

```js
  new webpack.optimize.CommonsChunkPlugin({
      name: 'manifest',
      chunks: ['vendor','utils']
    }),
```

当然，我们需要为其增加一个入口文件，

```js
  entry: {
    app: './src/main.js',
    utils:['./src/assets/libs/jquery.js'] //这里可以根据你自己需求增加其他三方库文件
  }
```
> 其他插件

此外，还有其他的一些插件，如 `compression-webpack-plugin`,用于根据正则匹配进行压缩处理的； `webpack-bundle-analyzer`可以为你的包的尺寸优化等的，自己可以摸索一下。

## /config

### 环境变量定义

该部分定义了生产，开发和测试的全局环境变量，以调整在不同环境下使用不同的webpack配置，通过定义 `NODE_ENV`变量：

```js
var merge = require('webpack-merge')
var prodEnv = require('./prod.env')

module.exports = merge(prodEnv, {
  NODE_ENV: '"development"'
})
```
## /node_modules
### npm安装的第三包

这里估计使用过npm的同学都不会陌生，这里放置的都是我们通过`npm install`方式安装的第三方工具类，可供我们通过require关键字快速引入。

## /test
### unit

该文件主要是对vue-cli建立的项目配备了unit-test的配套，可以让用户快速编写组件级别的单元测试，并通过在根目录下使用 `npm run unit`命令去进行代码的单元测试：

> index.js

```js
const testsContext = require.context('./specs', true, /\.spec$/)
testsContext.keys().forEach(testsContext)
```
通过require 当前目录下 `specs` 中的用例文件，去进行.vue单元测试

```js
const srcContext = require.context('../../src', true, /^\.\/(?!main(\.js)?$)/)
srcContext.keys().forEach(srcContext)
```
这里定义了读取的源文件目录，第一行通过正则的方式去读取`../../src`下，除了`main.js`以外的文件进行遍历，我么如果只需对局部的文件进行单元测试，我们可以把路径范围缩小 如只对src下的components进行单元测试：

```js
const srcContext = require.context('../../src/components', true, /^\.\/(?!main(\.js)?$)/)
```

> karma.conf.js

由于vue-cli配备的单元测试，使用了karma.js，这里我们只需要使用默认的配置即可，一般不用更改。

> specs

这里是测试用例当中的 `断言` 部分，通过编写 `*.spec.js`去对组件中的逻辑部分进行断言判断，达到单元测试的效果,下面是一个对组件实例化，并对组件中的视图文字进行断言的例子：

```js
import Vue from 'vue'
import Hello from '@/components/Hello'

describe('Hello.vue', () => {
  it('should render correct contents', () => {
    const Constructor = Vue.extend(Hello)
    const vm = new Constructor().$mount()
    expect(vm.$el.querySelector('.hello h1').textContent)
      .to.equal('Welcome to Your Vue.js App')
  })
})
```

> coverage

karma的单元测试，同时提供了 `coverage Rate`即单元测试的覆盖率，会根据不同文件，从行数到函数的数量，描述你的单元测试是否完整，通过 `npm run unit`完成后，查看 `coverage\lcov-report\index.html`可以看到覆盖率结果：

![简析vue-cli快速搭建项目下的配置](http://7ktvc3.com1.z0.glb.clouddn.com/vue_webpack_coverage.png)

## 根目录
### .babelrc

```js
{
  "presets": [
    ["env", {
      "modules": false,
      "targets": {
        "browsers": ["> 1%", "last 2 versions", "not ie <= 8"]
      }
    }],
    "stage-2"
  ],
  "plugins": ["transform-runtime"],
  "env": {
    "test": {
      "presets": ["env", "stage-2"],
      "plugins": ["istanbul"]
    }
  }
}
```
制定了babel的配置，定义了加载的插件和测试运行时所需的插件 `istanbul`

### .editorconfig

```js
root = true
[*]
charset = utf-8
indent_style = space
indent_size = 2
end_of_line = lf
insert_final_newline = true
trim_trailing_whitespace = true
```
定义了编辑格式：利用`utf-8`编码， 空格缩进方式 `space` , 行缩进 `两个字符` 还有 结尾插入新行，处理字符首尾空白字符。

### .gitignore

很熟悉的配置，即git仓库同步时需要忽略提交的文件，可以通过准确和正则的方式进行所需要忽略，不赘述。

### .postcssrc.js

新版的webpack只要配备了postcss的loaders，都会检查根目录是否有该配置文件，通过导出 `plugins`，进行配置你所需要的，下面是我使用了 `autoprefixer` 和 `postcss-sprites`的配置

```js
module.exports = {
  "plugins": {
    // to edit target browsers: use "browserslist" field in package.json
    "autoprefixer": {},
    "postcss-sprites":{
      // stylesheetPath: './css',
      spritePath: './css/images/',
      filterBy: function(image) {
        // Allow only png files
        if (!/\.png$/.test(image.url)) {
          return Promise.reject();
        }

        return Promise.resolve();
      }
    }
  }
}
```

### index.html

```html
<!DOCTYPE html>
<html>
  <head>
    <meta charset="utf-8">
    <title>uvoice</title>
  </head>
  <body>
    <section class="container" id="J_container">
    </section>
    <!-- built files will be auto injected -->
  </body>
</html>
```

html模板，可以供htmlPlugins插件使用，后面编译好的脚本会自动injected到注释行内。

### package.json

估计用过npm的同学，也应该了解了，这个文件是用于记录和管理npm包的依赖和版本号，不赘述。

### README.md

文件如其名，就是阅读说明，你们可以在里面对项目的一些使用或者命令行进行描述，以作记录。
