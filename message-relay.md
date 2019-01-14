# Message Relay

本案例演示了如何在serivce worker与页面之间进行通信，以及如何使用service worker在页面之间中继消息。

# 难度等级

入门级

# 应用场景

postMessage API非常适合在床空合iframe之间传递消息，我们可以使用service worker的消息事件实现一个消息管理器。

# 特点和用法

* postMessage API
* Service worker注册

# 方案类别

通用方法

# 代码示例

index.html

```html
<!doctype html>
<html>
<head>
  <meta charset="utf-8">
  <title>Message Relay - ServiceWorker Cookbook</title>
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <style>
    body {
      font-family: monospace;
      font-size: 1rem;
      white-space: pre-wrap;
    }
  </style>
</head>
<body>

<div>
  Service Worker Status: <span id="status">not supported</span>
</div>

Open <a href="./index.html" target="_blank">another window with this page</a> and type some text in below to postMessage it to the ServiceWorker which will forward the message along.

<div id="received"></div>

<textarea id="message" style="width: 100%" rows="10"></textarea>

<script src="./index.js"></script>
</body>
</html>

```

index.js

```js
(function() {
    // 只有当前客户端支持service worker的时候执行下面的逻辑
    if (navigator.serviceWorker) {
        // 拿到当前界面的DOM节点
        var message = document.getElementById('message');
        var received = document.getElementById('received');
        var status = document.getElementById('status');
        var inbox = {};
        // 设置一个支持service worker的索引
        status.textContent = 'supported';
        navigator.serviceWorker.register('service-worker.js');
        // 监听service worker的message事件
        navigator.serviceWorker.addEventListener('message', function(event) {
            // 当一条message接收后在页面上显示这条message
            var clientId = event.data.client;
              var node;
              // 如果当前客户端之前没有接收过message，我们需要设置个地方显示message              
              if (!inbox[clientId]) {
                node = document.createElement('div');
                received.appendChild(node);
                inbox[clientId] = node;
              }
              // 显示message
               node = inbox[clientId];
               node.textContent = 'Client ' + clientId + ' says: ' + event.data.message;
        });
        
        message.addEventListener('input', function() {
            // 判断service worker是否存在，并非总是有service worker可以发送消息，比如：当页面强制刷新的时候
            if (!navigator.serviceWorker.controller) {
                status.textContent = 'error: no controller';
                return;
            }
            //发送消息给service worker
            navigator.serviceWorker.controller.postMessage(message.value);
        });
    }
})();
```

service-worker.js

```js
// 监听客户端的message事件
self.addEventListener('message', function(event) {
  // 获取所有已连接的客户端并转发message
  var promise = self.clients.matchAll()
  .then(function(clientList) {
    // event.source.id字段表示消息发送者的ID
    var senderID = event.source.id;

    clientList.forEach(function(client) {
      // 跳过消息发送者
      if (client.id === senderID) {
        return;
      }
      client.postMessage({
        client: senderID,
        message: event.data
      });
    });
  });
  // 如果定义了event.waitUntil，则用它来延长service worker的生命周期
  if (event.waitUntil) {
    event.waitUntil(promise);
  }
});
/*
  立即生命所有新客户端，这不是发送消息必须的，但是不用刷新即可使用可以获得更好的演示体验
  在Immediate Claim章节有更完整的例子
*/
self.addEventListener('activate', function(event) {
  event.waitUntil(self.clients.claim());
});
```



