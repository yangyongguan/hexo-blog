---
title: 跨页面通信的各种姿势
date: 2017-09-19 10:02:23
tags:
---


## 前言

当看到这个题目的时候惊喜不惊喜，【第1060期】跨浏览器tab页的通信解决方案尝试不是刚刚推送了吗？今天还来，对因为在留言中发现了不一样的东西。今日早读文章由蚂蚁金服数据前端@mekron授权分享。

正文从这开始～

将跨页面通讯类比计算机进程间的通讯，其实方法无外乎那么几种，而web领域可以实现的技术方案主要是类似于以下两种原理：

获取句柄，定向通讯

共享内存，结合轮询或者事件通知来完成业务逻辑

由于第二种原理更利于解耦业务逻辑，具体的实现方案比较多样。以下是具体的实现方案，简单介绍下，权当科普：


<h4 style="color:#3DA742">一、获取句柄</h4>

<span style="color:#007EAF">具体方案</span>

```javascript

// parent.html
const childPage = window.open('child.html', 'child')

childPage.onload = () => {
   childPage.postMessage('hello', location.origin)
}

// child.html
window.onmessage = evt => {
// evt.data

```

父页面通过`window.open(url, name)`方式打开的子页面可以获取句柄，然后通过postMessage完成通讯需求。

<span style="color:#007EAF">提示</span>

1. 当指定`window.open`的第二个name参数时，再次调用`window.open('****', 'child')`会使之前已经打开的同name子页面刷新

2. 由于安全策略，异步请求之后再调用`window.open`会被浏览器阻止，不过可以通过句柄设置子页面的url即可实现类似效果

```javascript

// 首先先开一个空白页
const tab = window.open('about:blank')

// 请求完成之后设置空白页的url
fetch(/* ajax */).then(() => {
   tab.location.href = '****'
})

```

<span style="color:#007EAF">优劣</span>
缺点是只能与自己打开的页面完成通讯，应用面相对较窄；但优点是在跨域场景中依然可以使用该方案。

<h4 style="color:#3DA742">二、localStorage</h4>

<span style="color:#007EAF">具体方案</span>

设置共享区域的storage，storage会触发storage 事件

```javascript

// A.html
localStorage.setItem('message', 'hello')

// B.html
window.onstorage = evt => {
// evt.key, evt.oldValue, evt.newValue
}

```

<span style="color:#007EAF">提示</span>
1. 触发写入操作的页面下的**storage listener**不会被触发
2. storage事件只有在发生改变的时候才会触发，即重复设置相同值不会触发listener
3. safari隐身模式下无法设置localStorage值

<span style="color:#007EAF">优劣</span>
API简单直观，兼容性好，除了跨域场景下需要配合其他方案，无其他缺点

<h4 style="color:#3DA742">三、BroadcastChannel</h4>

<span style="color:#007EAF">具体方案</span>
和`localStorage`方案基本一致，额外需要初始化

```javascript

// A.html
const channel = new BroadcastChannel('tabs')
channel.onmessage = evt => {
// evt.data
}

// B.html
const channel = new BroadcastChannel('tabs')
channel.postMessage('hello')

```

<span style="color:#007EAF">优劣</span>

和`localStorage`方案没特别区别，都是同域、API简单，`BroadcastChannel`方案兼容性差些（chrome > 58），但比`localStorage`方案生命周期短（不会持久化），相对干净些。

<h4 style="color:#3DA742">四、SharedWorker</h4>

具体方案

SharedWorker本身并不是为了解决通讯需求的，它的设计初衷应该是类似总控，将一些通用逻辑放在SharedWorker中处理。不过因为也能实现通讯，所以一并写下：

```javascript

// A.html
var sharedworker = new SharedWorker('worker.js')
sharedworker.port.start()
sharedworker.port.onmessage = evt => {
// evt.data
}

// B.html
var sharedworker = new SharedWorker('worker.js')
sharedworker.port.start()
sharedworker.port.postMessage('hello')

// worker.js
const ports = []
onconnect = e => {
const port = e.ports[0]
   ports.push(port)
   port.onmessage = evt => {
       ports.filter(v => v!== port) // 此处为了贴近其他方案的实现，剔除自己
       .forEach(p => p.postMessage(evt.data))
   }
}

```

<span style="color:#007EAF">优劣</span>

相较于其他方案没有优势，此外，API复杂而且调试不方便。

<h4 style="color:#3DA742">五、Cookie</h4>

<span style="color:#007EAF">具体方案</span>

一个古老的方案，有点`localStorage`的降级兼容版，我也是整理本文的时候才发现的，思路就是往`document.cookie`写入值，由于cookie的改变没有事件通知，所以只能采取轮询脏检查来实现业务逻辑。

<span style="color:#007EAF">优劣</span>

相较于其他方案没有存在优势的地方，只能同域使用，而且污染cookie以后还额外增加AJAX的请求头内容。

<h4 style="color:#3DA742">六、Server</h4>

之前的方案都是前端自行实现，势必受到浏览器限制，比如无法做到跨浏览器的消息通讯，比如大部分方案都无法实现跨域通讯（需要增加额外的postMessage逻辑才能实现）。通过借助服务端，还有很多增强方案，也一并说下。

乞丐版

后端无开发量，前端定期保存，在tab被激活时重新获取保存的数据，可以通过校验hash之类的标记位来提升检查性能。

```javascript

window.onvisibilitychange = () => {
if (document.visibilityState === 'visible') {
// AJAX
   }
}

```

<h4 style="color:#3DA742">Server-sent Events / Websocket</h4>

项目规模小型的时候可以采取这类方案，后端自行维护连接，以及后续的推送行为。

<span style="color:#007EAF">SSE</span>

```javascript

// 前端
const es = new EventSource('/notification')

es.onmessage = evt => {
// evt.data
}
es.addEventListener('close', () => {
   es.close()
}, false)
// 后端，express为例
const clients = []

app.get('/notification', (req, res) => {
   res.setHeader('Content-Type', 'text/event-stream')
   clients.push(res)
   req.on('aborted', () => {
// 清理clients
   })
})
app.get('/update', (req, res) => {
// 广播客户端新的数据
   clients.forEach(client => {
       client.write('data:hello\n\n')
       setTimeout(() => {
           client.write('event:close\ndata:close\n\n')
       }, 500)
   })
   res.status(200).end()
})

```

<span style="color:#007EAF">Websocket</span>

http://socket.io、sockjs例子比较多，略

<h4 style="color:#3DA742">消息队列</h4>

项目规模大型时，需要消息队列集群长时间维护长链接，在需要的时候进行广播。

提供该类服务的云服务商很多，或者寻找一些开源方案自建。

例如MQTT协议方案（阿里云就有提供），web客户端本质上也是websocket，需要集群同时支持ws和mqtt协议，示例如下：


```javascript

// 前端
// 客户端使用开源的Paho
// port会和mqtt协议通道不同
const client = new Paho.MQTT.Client(host, port, 'clientId')

client.onMessageArrived = message => {
// message. payloadString
}
client.connect({
   onSuccess: () => {
       client.subscribe('notification')
   }
})
// 抑或，借助flash（虽然快要被淘汰了）进行mqtt协议连接并订阅相应的频道，flash再通过回调抛出消息

// 后端
// 根据服务商提供的Api接口调用频道广播接

```


关于本文
作者：@mekron
原文：https://zhuanlan.zhihu.com/p/29368435









