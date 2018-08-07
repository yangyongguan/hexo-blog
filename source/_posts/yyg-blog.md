---
title: 认识HTML5的WebSocket（一）
date: 2017-07-19 15:01:36
tags:
---

## 认识HTML5的WebSocket
> 在HTML5规范中，我最喜欢的Web技术就是正迅速变得流行的WebSocket API。WebSocket提供了一个受欢迎的技术，以替代我们过去几年一直在用的Ajax技术。这个新的API提供了一个方法，从客户端使用简单的语法有效地推动消息到服务器。让我们看一看HTML5的WebSocket API：它可用于客户端、服务器端。而且有一个优秀的第三方API，名为Socket.IO。

## 一、什么是WebSocket API?

WebSocket API是下一代客户端-服务器的异步通信方法。该通信取代了单个的TCP套接字，使用ws或wss协议，可用于任意的客户端和服务器程序。WebSocket目前由W3C进行标准化。WebSocket已经受到Firefox 4、Chrome 4、Opera 10.70以及Safari 5等浏览器的支持。

WebSocket API最伟大之处在于服务器和客户端可以在给定的时间范围内的任意时刻，相互推送信息。WebSocket并不限于以Ajax(或XHR)方式通信，因为Ajax技术需要客户端发起请求，而WebSocket服务器和客户端可以彼此相互推送信息；XHR受到域的限制，而WebSocket允许跨域通信。

Ajax技术很聪明的一点是没有设计要使用的方式。WebSocket为指定目标创建，用于双向推送消息。

## 二、WebSocket API的用法

只专注于客户端的API，因为每个服务器端语言有自己的API。下面的代码片段是打开一个连接，为连接创建事件监听器，断开连接，消息时间，发送消息返回到服务器，关闭连接。

// 创建一个Socket实例
var socket = new WebSocket('ws://localhost:8080');

// 打开Socket
socket.onopen = function(event) {

  // 发送一个初始化消息
  socket.send('I am the client and I\'m listening!');

  // 监听消息
  socket.onmessage = function(event) {
    console.log('Client received a message',event);
  };

  // 监听Socket的关闭
  socket.onclose = function(event) {
    console.log('Client notified socket has closed',event);
  };

  // 关闭Socket....
  //socket.close()
};

让我们来看看上面的初始化片段。参数为URL，ws表示WebSocket协议。onopen、onclose和onmessage方法把事件连接到Socket实例上。每个方法都提供了一个事件，以表示Socket的状态。

onmessage事件提供了一个data属性，它可以包含消息的Body部分。消息的Body部分必须是一个字符串，可以进行序列化/反序列化操作，以便传递更多的数据。

WebSocket的语法非常简单，使用WebSockets是难以置信的容易……除非客户端不支持WebSocket。IE浏览器目前不支持WebSocket通信。如果你的客户端不支持WebSocket通信，下面有几个后备方案供你使用：

## 三、WebSocket 后备方案

<strong>Flash技术</strong> —— Flash可以提供一个简单的替换。 使用Flash最明显的缺点是并非所有客户端都安装了Flash，而且某些客户端，如iPhone/iPad，不支持Flash。

<strong>AJAX Long-Polling技术</strong> —— 用AJAX的long-polling来模拟WebSocket在业界已经有一段时间了。它是一个可行的技术，但它不能优化发送的信息。也就是说，它是一个解决方案，但不是最佳的技术方案。

由于目前的IE等浏览器不支持WebSocket，要提供WebSocket的事件处理、返回传输、在服务器端使用一个统一的API，那么该怎么办呢？幸运的是，Guillermo Rauch创建了一个Socket.IO技术。
