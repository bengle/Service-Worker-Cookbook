# Push Clients

本章案例演示如何对推送消息进行订阅管理。

# 难度等级

高级

# 应用场景

允许用户订阅你应用的功能以保持您与访客的联系并提升转化率。

# 初始状态

在service worker注册后，客户端检查它是否已经订阅了推送消息服务，按钮的内容也根据此设置。

# 订阅

在成功注册订阅后\(index.js::pushManager.subscribe\)，客户端向应用服务端发送注册订阅的post请求。

# 推送消息

服务器定期使用消息推送库向已经注册的终端发送消息，如果某个终端不再注册（过期或者取消）则后从订阅列表中删除该终端。

# 取消订阅

在成功取消订阅后\(index.js::pushSubscription.unsubscribe\)，客户端会想应用服务端发送一个取消注册订阅的post请求，服务器就不在发送消息了。

# 订阅过期

service  worker通过监控_pushsubscriptionchange_事件来查看订阅是否过期和向推送服务器重新订阅。

# 不包含的内容

用户可能通过页面之外的方式取消订阅\(例如：浏览器设置或者消息UI等\)，在我们的例子中服务器会停止向客户端推送，但是客户端是不知道的。针对这个问题一种解决方案是客户端定期检查订阅注册是否在激活状态。

# 方案类型

网络推送

# 代码示例

index.js

```html
<!doctype html>
<html>
<head>
  <meta charset='utf-8'>
  <title>Push notifications with subscription management - ServiceWorker Cookbook</title>
  <meta name='viewport' content='width=device-width, initial-scale=1'>
  <style>
    body {
      font-family: monospace;
      font-size: 1rem;
      white-space: pre-wrap;
    }
  </style>
</head>
<body>
  <p>This demo shows how to subscribe/unsubscribe to the push notifications.</p>
  <button id='subscriptionButton' disabled=true></button>
  <p>To simulate the subscription expiration under Firefox please set the <code>dom.push.userAgentId</code> in about:config to an empty string. (Note: <A href="https://bugzilla.mozilla.org/show_bug.cgi?id=1222428">bug 1222428</a>)</p>
  <script src="/tools.js"></script>
  <script src="./index.js"></script>
</body>
</html>
```

index.js

```js
var subscriptionButton = document.getElementById('subscriptionButton');
// 注册service worker
navigator.serviceWorker.register('service-worker.js');
// 当service worker启动之后将button设置enable，再检查我们是否已经设置了订阅
navigator.serviceWorker.ready
.then(function(registration) {
  console.log('service worker registered');
  subscriptionButton.removeAttribute('disabled');

  return registration.pushManager.getSubscription();
}).then(function(subscription) {
  if (subscription) {
    console.log('Already subscribed', subscription.endpoint);
    setUnsubscribeButton();
  } else {
    setSubscribeButton();
  }
});
// 从service worker获取registration，并使用registration.pushManager.subscribe创建新的订阅
// 通过向服务端发送带有订阅信息的post请求来接收新的订阅
function subscribe() {
  navigator.serviceWorker.ready
  .then(async function(registration) {
    // 获取服务端共钥
    const response = await fetch('./vapidPublicKey');
    const vapidPublicKey = await response.text();
    // chrome不支持base64编码的vapidPublicKey，这里使用tools.js中定义的urlBase64ToUint8Array方法处理
    const convertedVapidKey = urlBase64ToUint8Array(vapidPublicKey);
    // 订阅用户
    return registration.pushManager.subscribe({
      userVisibleOnly: true,
      applicationServerKey: convertedVapidKey
    });
  }).then(function(subscription) {
    console.log('Subscribed', subscription.endpoint);
    return fetch('register', {
      method: 'post',
      headers: {
        'Content-type': 'application/json'
      },
      body: JSON.stringify({
        subscription: subscription
      })
    });
  }).then(setUnsubscribeButton);
}
// 通过service worker获取已经存在的订阅，
// 用subscription.unsubscribe()方法取消和注销订阅，再向服务器发送post请求，通知其不再向不存在的终端推送消息
function unsubscribe() {
  navigator.serviceWorker.ready
  .then(function(registration) {
    return registration.pushManager.getSubscription();
  }).then(function(subscription) {
    return subscription.unsubscribe()
      .then(function() {
        console.log('Unsubscribed', subscription.endpoint);
        return fetch('unregister', {
          method: 'post',
          headers: {
            'Content-type': 'application/json'
          },
          body: JSON.stringify({
            subscription: subscription
          })
        });
      });
  }).then(setSubscribeButton);
}
// 根据不同的动作改变button内的文字
function setSubscribeButton() {
  subscriptionButton.onclick = subscribe;
  subscriptionButton.textContent = 'Subscribe!';
}
function setUnsubscribeButton() {
  subscriptionButton.onclick = unsubscribe;
  subscriptionButton.textContent = 'Unsubscribe!';
}
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
// 定义一个全局数组存储激活的终端，实际项目中我们一般用数据库存储
const subscriptions = {};
// 服务器想用户推送消息的频率（秒）
const pushInterval = 10;
// 向推送服务器发送消息，报告订阅信息
// 如果推送服务器返回结果是错误那么就从subscriptions数组中删除订阅终端，就可以将其从推送列表中删除
function sendNotification(subscription) {
  webPush.sendNotification(subscription)
  .then(function() {
    console.log('Push Application Server - Notification sent to ' + subscription.endpoint);
  }).catch(function() {
    console.log('ERROR in sending Notification, endpoint removed ' + subscription.endpoint);
    delete subscriptions[subscription.endpoint];
  });
}
// 实际项目中我们通常在某个事件触发后发送消息
// 这里用定时器模拟pushInterval秒后向订阅的终端发送推送消息
setInterval(function() {
  Object.values(subscriptions).forEach(sendNotification);
}, pushInterval * 1000);

module.exports = function(app, route) {
  app.get(route + 'vapidPublicKey', function(req, res) {
    res.send(process.env.VAPID_PUBLIC_KEY);
  });
  // 通过添加到subscriptions数组将一个订阅完成注册
  app.post(route + 'register', function(req, res) {
    var subscription = req.body.subscription;
    if (!subscriptions[subscription.endpoint]) {
      console.log('Subscription registered ' + subscription.endpoint);
      subscriptions[subscription.endpoint] = subscription;
    }
    res.sendStatus(201);
  });
  // 通过将其从subscriptions数组中删除将一个订阅注销
  app.post(route + 'unregister', function(req, res) {
    var subscription = req.body.subscription;
    if (subscriptions[subscription.endpoint]) {
      console.log('Subscription unregistered ' + subscription.endpoint);
      delete subscriptions[subscription.endpoint];
    }
    res.sendStatus(201);
  });
};
```

service-worker.js

```js
// 监听push事件，定义消息内容，显示推送消息
self.addEventListener('push', function(event) {
  event.waitUntil(self.registration.showNotification('ServiceWorker Cookbook', {
    body: 'Push Notification Subscription Management'
  }));
});
// 监听pushsubscriptionchange事件，订阅到期时会触发该事件
// 向服务器发送带终端信息的post请求可以再次订阅和注册
// 现实项目中有可能会增加用户识别逻辑
self.addEventListener('pushsubscriptionchange', function(event) {
  console.log('Subscription expired');
  event.waitUntil(
    self.registration.pushManager.subscribe({ userVisibleOnly: true })
    .then(function(subscription) {
      console.log('Subscribed after expiration', subscription.endpoint);
      return fetch('register', {
        method: 'post',
        headers: {
          'Content-type': 'application/json'
        },
        body: JSON.stringify({
          endpoint: subscription.endpoint
        })
      });
    })
  );
});
```



