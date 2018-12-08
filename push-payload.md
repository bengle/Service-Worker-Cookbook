# Push Payload

在这个例子中，我们可以学会发送和接收包含各种内容的推送消息，消息可以包含除字符串之外的多种数据格式（字符串、ArrayBuffer、Blob、JSON等）。

# 难度等级

入门级

# 应用场景

推送消息内容除了文本还可以axing应用程序提供各种有效数据格式内容，当我们需要向应用推送富负载内容的时候可以参考本章是如何做的。

# 方案类型

网络推送

# 示例代码

index.js

```js
// 注册service worker
navigator.serviceWorker.register('service-worker.js');

navigator.serviceWorker.ready
.then(function(registration) {
    // 使用pushManager获取用户对推送服务的订阅
    return registration.pushManager.getSubscription()
      .then(async function(subscription) {
        // 找到订阅信息并返回
        if (subscription) {
          return subscription;
        }
        // 获取服务的公钥
        const response = await fetch('./vapidPublicKey');
        const vapidPublicKey = await response.text();
        // chrome不支持base64编码的vapidPublicKey，这里使用tools.js中定义的urlBase64ToUint8Array方法处理
        const convertedVapidKey = urlBase64ToUint8Array(vapidPublicKey);
        // 没有订阅信息则订阅用户(userVisibleOnly允许我们指定哪些消息不发送给用户)
        return registration.pushManager.subscribe({
          userVisibleOnly: true,
          applicationServerKey: convertedVapidKey
        });
    });
}).then(function(subscription) {
  // 使用fetch向服务端发送订阅详情
  fetch('./register', {
    method: 'post',
    headers: {
      'Content-type': 'application/json'
    },
    body: JSON.stringify({
      subscription: subscription
    }),
  });

  document.getElementById('doIt').onclick = function() {
    const payload = document.getElementById('notification-payload').value;
    const delay = document.getElementById('notification-delay').value;
    const ttl = document.getElementById('notification-ttl').value;
    // 让服务端向客户端发送消息(这是测试用法，实际应用中，推送消息可能会由服务端的某个事件生成)
    fetch('./sendNotification', {
      method: 'post',
      headers: {
        'Content-type': 'application/json'
      },
      body: JSON.stringify({
        subscription: subscription,
        payload: payload,
        delay: delay,
        ttl: ttl,
      }),
    });
  };

});
```

server.js

```js
/*
  我们使用web-push库实现消息推送，
  这个库封装了应用服务与消息推送服务之间通信的实施细节，
  可以参考： 
    https://tools.ietf.org/html/draft-ietf-webpush-protocol
    https://tools.ietf.org/html/draft-ietf-webpush-encryption
*/
const webPush = require('web-push');
// 设置推送消息的加密key
if (!process.env.VAPID_PUBLIC_KEY || !process.env.VAPID_PRIVATE_KEY) {
  console.log("You must set the VAPID_PUBLIC_KEY and VAPID_PRIVATE_KEY "+
    "environment variables. You can use the following ones:");
  console.log(webPush.generateVAPIDKeys());
  return;
}

webPush.setVapidDetails(
  'https://serviceworke.rs/',
  process.env.VAPID_PUBLIC_KEY,
  process.env.VAPID_PRIVATE_KEY
);

module.exports = function(app, route) {
  app.get(route + 'vapidPublicKey', function(req, res) {
    res.send(process.env.VAPID_PUBLIC_KEY);
  });

  app.post(route + 'register', function(req, res) {
    // 现实应用中我们会存储订阅的消息
    res.sendStatus(201);
  });

  app.post(route + 'sendNotification', function(req, res) {
    const subscription = req.body.subscription;
    const payload = req.body.payload;
    const options = {
      TTL: req.body.ttl
    };

    setTimeout(function() {
      webPush.sendNotification(subscription, payload, options)
      .then(function() {
        res.sendStatus(201);
      })
      .catch(function(error) {
        console.log(error);
        res.sendStatus(500);
      });
    }, req.body.delay * 1000);
  });
};
```

service-worker.js

```js
// 注册push事件监听
self.addEventListener('push', function(event) {
  /*
    从event.data（一个PushMessageData对象）中解析出有效的文本数据，ArrayBuffer、Blob、JSON等格式也是支持的
    具体使用方法可以查看文档：https://developer.mozilla.org/en-US/docs/Web/API/PushMessageData.
  */
  const payload = event.data ? event.data.text() : 'no payload';
  // 在notification创建成功前保持service worker激活状态
  event.waitUntil(
    // 将‘ServiceWorker Cookbook’作为推送消息title，接口数据作为消息体，显示推送消息
    self.registration.showNotification('ServiceWorker Cookbook', {
      body: payload,
    })
  );
});
```

tools.js

```js
function urlBase64ToUint8Array(base64String) {
  var padding = '='.repeat((4 - base64String.length % 4) % 4);
  var base64 = (base64String + padding)
    .replace(/\-/g, '+')
    .replace(/_/g, '/');

  var rawData = window.atob(base64);
  var outputArray = new Uint8Array(rawData.length);

  for (var i = 0; i < rawData.length; ++i) {
    outputArray[i] = rawData.charCodeAt(i);
  }
  return outputArray;
}
```

index.html

```html
<!doctype html>
<html>
<head>
  <meta charset="utf-8">
  <title>Push Notification with payload - ServiceWorker Cookbook</title>
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
<p>This demo shows how to send push notifications with a payload.</p>

<form>
Notification payload: <input id='notification-payload' type='text' value='Insert here a payload'></input>
Notification delay: <input id='notification-delay' type='number' value='5'></input> seconds
Notification Time-To-Live: <input id='notification-ttl' type='number' value='0'></input> seconds
</form>

<button id="doIt">Request sending a notification!</button>

<script src="/tools.js"></script>
<script src="./index.js"></script>
</body>
</html>
```



