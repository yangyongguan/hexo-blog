---
title: 使用postcss实现vw移动端适配
date: 2018-01-30 12:01:36
tags:
---

有关于移动端的适配布局一直以来都是众说纷纭，对应的解决方案也是有很多种。在《[使用Flexible实现手淘H5页面的终端适配](http://web.jobbole.com/84285/)》提出了Flexible的布局方案，
随着viewport单位越来越受到众多浏览器的支持，因此在《[再聊移动端页面的适配](https://www.cnblogs.com/zhouyangla/p/9273700.html)》一文中提出了vw来做移动端的适配问题。
到目前为止不管是哪一种方案，都还存在一定的缺陷。在接触到大漠先生牵头开发的vw解决方案之前，我使用的是阿里的第一代适配解决方案 lib-flexible 在使用vw解决方案开发一套H5之后，我真正的被vw的威力所折服。
话不多说开工

<hr>

#### 使用vue-cli新建项目

```javascript
    vue init webpack vue-demo
    cd vue-demo
    npm run dev
```

#### 安装依赖
```javascript
   npm i postcss-aspect-ratio-mini postcss-px-to-viewport postcss-write-svg postcss-cssnext postcss-viewport-units cssnano cssnano-preset-advanced --S
```

#### 配置postcssrc.js
```javascript
   module.exports = {
     "plugins": {
       "postcss-import": {},
       "postcss-url": {},
       "postcss-aspect-ratio-mini": {},
       "postcss-write-svg": { utf8: false },
       "postcss-cssnext": {},
       "postcss-px-to-viewport": {
         viewportWidth: 750, // 视窗的宽度，对应的是我们设计稿的宽度，一般是750
         viewportHeight: 1334, // 视窗的高度，根据750设备的宽度来指定，一般指定1334，也可以不配置
         unitPrecision: 3, // 指定`px`转换为视窗单位值的小数位数（很多时候无法整除）
         viewportUnit: 'vw', // 转换成的视窗单位，建议使用vw
         selectorBlackList: ['.ignore', '.hairlines'], // 指定不转换为视窗单位的类，可以自定义，可以无限添加,建议定义一至两个通用的类名
         minPixelValue: 1, // 小于或等于`1px`不转换为视窗单位，你也可以设置为你想要的值
         mediaQuery: false // 是否允许在媒体查询中转换`px`
       },
       "postcss-viewport-units": {},
       "cssnano": {
         preset: "advanced",
         autoprefixer: false,
         "postcss-zindex": false
       }
     }
   }
```

## 说明
  * 容器适配，可以使用vw
  * 文本的适配，可以使用vw
  * 大于1px的边框、圆角、阴影都可以使用vw
  * 内距和外距，可以使用vw

<hr>

## postcss-write-svg 实现Retina屏1像素边框
首先记得在heade头加入
```html
  <meta name="viewport" content="width=device-width,initial-scale=1,maximum-scale=1,minimum-scale=1,user-scalable=no" />
```
#### 如果部分机型有问题，换插件降级处理
使用 [Viewport Units Buggyfill](https://github.com/rodneyrehm/viewport-units-buggyfill) 插件
在vue项目的index.html文件head标签添加引用
```javascript
<script src="//g.alicdn.com/fdilab/lib3rd/viewport-units-buggyfill/0.6.2/??viewport-units-buggyfill.hacks.min.js,viewport-units-buggyfill.min.js"></script>
```
在Index.html文件body标签后添加以下代码

```javascript
<script>
  // vw兼容性处理viewport-units-buggyfill
    window.onload = function () {
      window.viewportUnitsBuggyfill.init({ hacks: window.viewportUnitsBuggyfillHacks });
      //以下代码用户测试
      // var winDPI = window.devicePixelRatio;
      // var uAgent = window.navigator.userAgent;
      // var screenHeight = window.screen.height;
      // var screenWidth = window.screen.width;
      // var winWidth = window.innerWidth;
      // var winHeight = window.innerHeight;
      // console.log("Windows DPI:" + winDPI + ";\ruAgent:" + uAgent + ";\rScreen Width:" +
      //   screenWidth + ";\rScreen Height:" + screenHeight + ";\rWindow Width:" + winWidth +
      //   ";\rWindow Height:" + winHeight)
    }
  </script>
```

最后做个对img兼容处理，在全局添加(在main.js 用 Import '@/common/index.css')

```css
<script>
    img {
      content: normal !important;
    }
```


