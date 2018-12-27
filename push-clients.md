# Push Clients

本章案例演示了消息推送客户端的操作处理，当用户点击推送过来的消息时，控制service worker将客户端聚焦到应用程序的选项卡上，如果该应用选项卡已经关闭则重新打开它。

# 难度等级

高级

# 应用场景

演示了推送消息在根据应用程序状态不同产生的3种用户操作例子，它可以识别你是否在页面上，是否需要切换到打开的选项卡，或者重新打开选项卡。

# 方案类型

网络推送

# 代码示例

index.html

```html
<!doctype html>
<html>
<head>
  <meta charset="utf-8">
  <title>Push Notification with clients management - ServiceWorker Cookbook</title>
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
<p>This demo shows how to control the clients of a service worker when the user clicks on a push notification.</p>

<p>Press on 'Send notification' and try one of:</p>
<ul>
  <li>- Switching to another page: clicking on the notification will re-focus this page;</li>
  <li>- Closing this page: clicking on the notification will reopen it;</li>
  <li>- Remaining on this page: a different notification will be shown, nothing will happen if you click on it.</li>
</ul>

<button id="doIt">Send notification</button>

<script src="/tools.js"></script>
<script src="./index.js"></script>
</body>
</html>
```

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
  // 通知服务端向客户端发送一个推送消息（这里出于测试的目的简单使用，真实场景中我们通常在服务端的某个事件中生成推送消息）
  function askForNotifications(visible) {
    var notificationNum = document.getElementById('notification-num').value;
    fetch('./sendNotification', {
      method: 'post',
      headers: {
        'Content-type': 'application/json'
      },
      body: JSON.stringify({
        subscription: subscription,
        visible: visible,
        num: notificationNum,
      }),
    });
  }
  // 让服务端向客户端发送消息(这是测试用法，实际应用中，推送消息可能会由服务端的某个事件生成)
  document.getElementById('doIt').onclick = function() {
    fetch('./sendNotification', {
      method: 'post',
      headers: {
        'Content-type': 'application/json'
      },
      body: JSON.stringify({
        subscription: subscription
      }),
    });
  };

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
    setTimeout(function() {
      webPush.sendNotification(req.body.subscription)
      .then(function() {
        res.sendStatus(201);
      })
      .catch(function(error) {
        res.sendStatus(500);
        console.log(error);
      });
    }, 10000);
  });
};
```

service-worker.js

```js
// 立即控制页面
// 你可以参考“Immediate Claim”获得以下两个事件的详细说明
self.addEventListener('install', function(event) {
  event.waitUntil(self.skipWaiting());
});

self.addEventListener('activate', function(event) {
  event.waitUntil(self.clients.claim());
});

// 注册push事件
self.addEventListener('push', function(event) {
  event.waitUntil(
    // 查看当前service worker的客户端列表
    self.clients.matchAll().then(function(clientList) {
      // 检测是否至少有一个聚焦的客户端（是否有打开的客户端）
      var focused = clientList.some(function(client) {
        return client.focused;
      });

      var notificationMessage;
      if (focused) {
        notificationMessage = 'You\'re still here, thanks!';
      } else if (clientList.length > 0) {
        notificationMessage = 'You haven\'t closed the page, ' +
                              'click here to focus it!';
      } else {
        notificationMessage = 'You have closed the page, ' +
                              'click here to re-open it!';
      }

      /*
        显示标题是“ServiceWorker Cookbook”，内容是当前service worker的client状态的推送消息
        client状态包含：1页面是focus状态；2页面打开但是不是focus状态；3页面已经关闭
      */
      return self.registration.showNotification('ServiceWorker Cookbook', {
        body: notificationMessage,
      });
    })
  );
});

// 注册“notificationclick”事件
self.addEventListener('notificationclick', function(event) {
  event.waitUntil(
    // 查看当前service worker的客户端列表
    self.clients.matchAll().then(function(clientList) {
      // 如果有打开的页面则聚焦上去
      if (clientList.length > 0) {
        return clientList[0].focus();
      }

      // 否则就打开一个新的页面
      return self.clients.openWindow('../push-clients_demo.html');
    })
  );
});
```



