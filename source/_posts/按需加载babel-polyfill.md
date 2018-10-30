---
title: 按需加载babel-polyfill
date: 2018-01-10 16:01:36
tags:
---

  一直再使用vue项目，浏览器对js支持不一，我们在使用新的API在进行开发时不得不注意是否支持的兼容问题。babel-polyfill可以帮助我们解决很多问题。

## 1、认识babel-polyfill
Babel默认只转换JS语法，而不转换新的API，比如Iterator、Generator、Set、Maps、Proxy、Reflect、Symbol、Promise等全局对象，以及一些定义在全局对象上的方法(比如<span class="color-blue">Object.assign</span>)都不会转码。

举例来说，ES2015在Array对象上新增了Array.from方法。Babel 就不会转码这个方法。如果想让这个方法运行，必须使用babel-polyfill。(内部集成了core-js和regenerator)

```javascript
    npm install babel-polyfill --save
```
使用时，在所有代码运行之前增加<span class="color-blue">require('babel-polyfill')</span>或者更常规的操作是在webpack.config.js中将babel-polyfill作为第一个entry。因此必须把babel-polyfill作为dependencies而不是devDependencies

> 注意: 在你的整个应用里只使用一次<span class="color-blue">require('babel-polyfill')</span>。多次import或require('babel-polyfill')会引起报错，因为它可能导致全局冲突和其他难以追踪的问题。 我们建议创建一个只包含require语句的单个入口文件。

## 2、使用babel-polyfill缺点
给予一下两个缺点，这就是我们进行优化的地方：
1、使用后打包后的体积很大，因为babel-polyfill是一个整体，把所有方法都加到原型链上。比如我们只使用了<span class="color-blue">Array.from</span>，但它把<span class="color-blue">Object.defineProperty</span>也给加上了，这就是一种浪费了。
2、babel-polyfill会污染全局变量，给很多类的原型链上都作了修改，如果我们开发的也是一个类库供其他开发者使用，这种情况就会变得非常不可控。

## 3、解决方法
分级上面两个问题，我们提供以下几种办法进行参考
#### 解决方法1：单独引入
可以通过单独使用core-js的某个类库来解决，比如通过引入babel-runtime/core-js/promise来获取Promise
但是这样需要我们认为判断并且手动引入类库，太麻烦了。
#### 解决方法2：使用babel-runtime和babel-plugin-transform-runtime
安装
```javascript
    npm install --save-dev babel-plugin-transform-runtime
    npm install --save babel-runtime
```
然后在.babelrc中：
```javascript
    {
      "plugins": ["transform-runtime"]
    }
```
启用插件babel-plugin-transform-runtime后，Babel就会使用babel-runtime下的工具函数，将<span class="color-blue">Promise</span>重写成<span class="color-blue">_Promise</span>，然后引入<span class="color-blue">_Promise helper</span>函数。这样就避免了重复打包代码和手动引入模块的痛苦。

由于采用了沙盒（Sandbox）机制，不会污染全局变量，同时也不会去修改内建类的原型，带来的坏处是它不会polyfill原型上的扩展（例如 <span class="color-blue">Array.prototype.includes()</span>不会被polyfill，<span class="color-blue">Array.from()</span>则会被polyfill）
#### 解决方法3：使用babel-preset-env
在Babel7中引入了babel-preset-env，根据你支持的环境自动决定适合你的Babel插件
```javascript
    npm install babel-preset-env --save-dev
```
在没有任何配置选项的情况下，babel-preset-env与babel-preset-latest（或者babel-preset-es2015，babel-preset-es2016和babel-preset-es2017一起）的行为完全相同。

下面例子包含了支持每个浏览器最后两个版本和safari大于等于7版本所需的polyfill和代码转换：
```javascript
    {
      "presets": [
        ["env", {
          "targets": {
            "browsers": ["last 2 versions", "safari >= 7"]
          }
        }]
      ]
    }
```
如果你目标开发Node.js而不是浏览器应用的话，你可以配置babel-preset-env仅包含特定版本所需的polyfill和transform:
```javascript
    {
      "presets": [
        ["env", {
          "targets": {
            "node": "6.10"
          }
        }]
      ]
    }
```
按需加载babel-polyfill的关键是useBuiltIns选项，默认值为false，它的值有三种：

1. false: 不对polyfills做任何操作
2. entry: 根据target中浏览器版本的支持，将polyfills拆分引入，仅引入有浏览器不支持的polyfill
3. usage(新)：检测代码中ES6/7/8等的使用情况，仅仅加载代码中用到的polyfills

这个选项可以启用一个新的插件来替换语句import "babel-polyfill"或者require("babel-polyfill")以及基于浏览器环境的babel-polyfill个性化需求。

我们需要将选项值设为usage，然后它会在每个JS文件运行，分析根据每个文件用到的语言特性导入相关的polyfill，例如
```javascript
    import "core-js/modules/es6.promise";
    var a = new Promise();
```
当然分析可能会有错误，例如：
```javascript
    import "core-js/modules/es7.array.includes";
    a.includes // assume a is an []
```
babel-preset-env会假设a是数组，所以会导入相关的es7的includes方法

这样我们就真正实现了按需加载，会让我们打包后的代码大大减小。
