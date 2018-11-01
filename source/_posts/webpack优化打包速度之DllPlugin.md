---
title: webpack优化打包速度之webpack.DllPlugin与webpack.DllReferencePlugin
date: 2018-01-20 12:01:36
tags:
---

  webpack是现在使用最广泛的打包工具，公司项目也一直使用。最近随着公司项目资源及文件慢慢庞大,打包速度越来越慢。因此上网搜索找到解决办法：
使用webpack插件webpack.DllPlugin与webpack.DllReferencePlugin将不需要改动的第三方插件与自己的业务代码进行分开打包，
首先：
在项目根目录新建一个文件webpack.dll.config.js

```javascript
    const path    = require('path');
    const webpack = require('webpack');
    module.exports = {
      entry: {
          vendor: ['vue-router','vuex','vue/dist/vue.common.js','vue/dist/vue.js','vue-loader/lib/component-normalizer.js','vue']
      },
      output: {
        path: path.resolve(base.path),
        filename: '[name].dll.js',
        library: '[name]_library'
      },
      plugins: [
        new webpack.DllPlugin({
          path: path.resolve('./dist', '[name]-manifest.json'),
          name: '[name]_library'
        })
      ]
    }
```
将是用到的第三方插件添加到vendor中
然后：在webpack.config.js中添加代码
```javascript
   plugins: [
       new webpack.DllReferencePlugin({
         manifest: require('./dist/vendor-manifest.json')
       })
   ]
```
还需要在入口html文件中引入vendor.dll.js
```javascript
    <script type="text/javascript" src="./../vendor.dll.js"></script>
```
然后在package.json文件中添加快捷命令(划线句子），指定执行文件：
```javascript
    "scripts": {
        "build:dll": "webpack --config webpack.dll.config.js"
    },
```
最后打包的时候首先执行npm run build:dll命令会在打包目录下生成
vendor-manifest.json文件与vendor.dll.js文件。
Dll这个概念应该是借鉴了Windows系统的dll。一个dll包，就是一个纯纯的依赖库，它本身不能运行，是用来给你的app引用的。
打包dll的时候，Webpack会将所有包含的库做一个索引，写在一个manifest文件中，而引用dll的代码（dll user）在打包的时候，只需要读取这个manifest文件，就可以了。
然后在执行npm run build
发现现在的webpack打包速度就快了很多。
