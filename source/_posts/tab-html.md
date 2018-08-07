---
title: 跨浏览器tab页的通信解决方案尝试
date: 2017-09-18 11:39:19
tags:
---
本文转载自前端早读课
作者：欲休


## 目录


正文从这开始～

<h4 style="color:#3DA742">目标</h4>

当前页面需要与当前浏览器已打开的的某个tab页通信，完成某些交互。其中，与当前页面待通信的tab页可以是与当前页面同域（相同的协议、域名和端口），也可以是跨域的。

要实现这个特殊的功能，单单使用HTML5的相关特性是无法完成的，需要有更加巧妙的设计。

<h4 style="color:#3DA742">畅想</h4>

现在我们发现下思维，假设多种场景下的解决方案，最终寻找通用解。


<h4 style="color:#007AAA">case 1</h4>

两个需要交互的tab页面具有依赖关系。

如 A页面中通过JavaScript的window.open打开B页面，或者B页面通过iframe嵌入至A页面，此种情形最简单，可以通过HTML5的 window.postMessage API完成通信，由于postMessage函数是绑定在 window 全局对象下，因此通信的页面中必须有一个页面（如A页面）可以获取另一个页面（如B页面）的window对象，这样才可以完成单向通信；B页面无需获取A页面的window对象，如果需要B页面对A页面的通信，只需要在B页面侦听message事件，获取事件中传递的source对象，该对象即为A页面window对象的引用：


```javascript
//B页面

window.addEventListner('message',(e)=>{
    let {data,source,origin} = e;
    source.postMessage('message echo','/');
});

```
postMessage的第一个参数为消息实体，它是一个结构化对象，即可以通过“JSON.stringify和JSON.parse”函数还原的对象；第二个参数为消息发送范围选择器，设置为“/”意味着只发送消息给同源的页面，设置为“*”则发送全部页面。

<h4 style="color:#007AAA">case 2</h4>

两个打开的页面属于同源范畴。

若要实现两个互不相关的通源tab页面通信，可以使用一种比较巧妙的方式：localstorage。localStorage的存储遵循同源策略，因此同源的两个tab页面可以通过这种共享localStorage的方式实现通信，通过约定localStorage的某一个itemName，基于该key值的内容作为“共享硬盘”方式通信。

不过，如果单纯使用localStorage存储做通信方式会遇到一个问题，就是两个页面把握不准通信时机，如果A页面此刻需要发送给B页面一条消息“hello B”，它会设置localStorage.setItem('message','hello B')，并且采用setTimeout轮询等待B的消息；而B此刻也同样使用setTimeout轮训等待localStorage的message项的变化，当获取到'message'字段时，便取出消息'hello B'。B如果要发消息给A，仍然采用同样套路。

这种方式性能极其低下，需要通信两方不停的监听localStorage某项的变化，及其浪费事件队列处理效率。
幸好，HTML5提供了storage事件，通过window对象侦听storage事件，会侦听localStorage对象的变化事件（包括item的添加、修改和删除）。因此，通过事件可以完成高效的通信机制：



```javascript
//A 页面

window.addEventListener("storage", function(ev){
    if (ev.key == 'message') {
        // removeItem同样触发storage事件，此时ev.newValue为空
        if(!ev.newValue)
            return;
        var message = JSON.parse(ev.newValue);
        console.log(message);
    }
});

function sendMessage(message){
    localStorage.setItem('message',JSON.stringify(message));
    localStorage.removeItem('message');
}

// 发送消息给B页面
sendMessage('this is message from A');

```


```javascript
B 页面

window.addEventListener("storage", function(ev){
    if (ev.key == 'message') {
        // removeItem同样触发storage事件，此时ev.newValue为空
        if(!ev.newValue)
            return;
        var message = JSON.parse(ev.newValue);
        // 发送消息给A页面
        sendMessage('message echo from B');
    }
});

function sendMessage(message){
    localStorage.setItem('message',JSON.stringify(message));
    localStorage.removeItem('message');
}

```

发送消息采用sendMessage函数，该函数序列化消息，设置为localStorage的message字段值后，删除该message字段。这样做的目的是不污染localStorage空间，但是会造成一个无伤大雅的反作用，即触发两次storage事件，因此我们在storage事件处理函数中做了if(!ev.newValue) return;判断。

当我们在A页面中执行sendMessage函数，其他同源页面会触发storage事件，而A页面却不会触发storage事件；而且连续发送两次相同的消息也只会触发一次storage事件，如果需要解决这种情况，可以在消息体体内加入时间戳：

```javascript

sendMessage({
    data: 'hello world',
    timestamp: Date.now()
});
sendMessage({
    data: 'hello world',
    timestamp: Date.now()
});

```

通过这种方式，可以实现同源下的两个tab页通信，兼容性

> 通过caniuse网站查询storage事件发现，IE的浏览器支持非常的不友好，caniuse使用了“completely wrong”的形容词来表述这一程度。IE10的storage事件会在页面document文档对象构建完成后触发，这在嵌套iframe的页面中造成诸多问题；IE11的storage Event对象却不区分oldValue和newValue值，它们始终存储更新后的值

case 3

两个互不相关的tab页面通信。

这种情况才是最急需解决的问题，如何实现两个没有任何关系的tab页面通信，这需要一些技巧，而且需要有同时修改这两个tab页面的权限，否则根本不可能实现这两个tab页的能力。

在上述条件满足的情况下，我们就可以使用case1 和 case2的技术完成case 3的需求，这需要我们巧妙的结合HTML5 postMessage API 和 storage事件实现这两个毫无关系的tab页面的连通。为此，我想到了iframe，通过在这两个tab页嵌入同一个iframe页实现“桥接”，最终完成通信：

<h4 style="color:#007AAA">case 3</h4>


```javascript

tab A -----> iframe A[bridge.html]
                     |
                     |
                    \|/
             iframe B[bridge.html] ----->  tab B

```

单方向的通信原理如上图所示，tab A中嵌入iframe A，tab B中嵌入iframe B，这两个iframe引用相同的页面“bridge.html”。如果tab A发消息给tab B，首先tab A通过postMessage消息发送给iframe A（tab A可以获取到iframe A的window对象iframe.contentWindow）；此后iframe A通过storage消息完成与iframe B的通信（由于iframeA 与iframe B同源，因此case 2的通信方式这里可以使用）；最终，iframe B同样采用postMessage方式发送消息给tab B（在iframe中通过window.parent引用tab B的window对象）。至此，tab A的消息走通了所有链路，成功抵达tab B。

反方向发送消息同样的道理，这里就不在详细说明。接下来到了 talk is cheap，show me the code 环节：


```javascript

tab A:

// 向弹出的tab页面发送消息
window.sendMessageToTab = function(data){
    // 由于[#J_bridge]iframe页面的源文件在vstudio服务器中，因此postMessage发向“同源”
    document.querySelector('#J_bridge').contentWindow.postMessage(JSON.stringify(data),'/');
};

// 接收来自 [#J_bridge]iframe的tab消息
window.addEventListener('message',function(e){
    let {data,source,origin}  = e;
    if(!data)
        return;
    try{
        let info = JSON.parse(JSON.parse(data));
        if(info.type == 'BSays'){
           console.log('BSay:',info);
        }
    }catch(e){
    }
});

sendMessageToTab({
    type: 'ASays',
    data: 'hello world, B'
})

```


```javascript

bridge.html

window.addEventListener("storage", function(ev){
    if (ev.key == 'message') {
        window.parent.postMessage(ev.newValue,'*');
    }
});

function message_broadcast(message){
    localStorage.setItem('message',JSON.stringify(message));
    localStorage.removeItem('message');
}

window.addEventListener('message',function(e){
    let {data,source,origin}  = e;
    // 接受到父文档的消息后，广播给其他的同源页面
    message_broadcast(data);
});

```

```javascript

tab B

window.addEventListener('message',function(e){
    let {data,source,origin}  = e;
    if(!data)
        return;
    let info = JSON.parse(JSON.parse(data));
    if(info.type == 'ASays'){
        document.querySelector('#J_bridge').contentWindow.postMessage(JSON.stringify({
            type: 'BSays',
            data: 'hello world echo from B'
        }),'*');
    }
});

// tab B主动发送消息给tab A
document.querySelector('button').addEventListener('click',function(){
    document.querySelector('#J_bridge').contentWindow.postMessage(JSON.stringify({
        type: 'BSays',
        data: 'I am B'
    }),'*');
})

```

至此，通过在tab A和tab B中引入“桥接”功能的iframe[bridge.html]页面，实现了两个无关tab页的双向通信，这种实现的技巧性较强。

<h4 style="color:#3DA742">参考资料</h4>

Communication between tabs or windows

关于本文

作者：@欲休

原文：http://www.cnblogs.com/accordion/p/7535188.html
































