# Push Quota

本章案例尝试了不同浏览器对指标管理的策略，例如：发送许多消息（可见或不可见）看看如果保持浏览器选项卡打开和关闭分别会发生什么，或者如果你点击某些消息和不点击这些消息有什么不同。

# 难度等级

高级

# 应用场景

案例代码尝试查看不同浏览器对推送消息指标管理策略表现结果，以便查看用户对不同操作的行为方式。

# 方案类型

网络推送

# 代码示例

index.html

```html
<!doctype html>
<html>
<head>
  <meta charset="utf-8">
  <title>Push Notification with quota management - ServiceWorker Cookbook</title>
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <style>
    body {
      font-family: monospace;
      font-size: 1rem;
      white-space: pre-wrap;
    }

    span {
      color: red;
    }
  </style>
</head>
<body>
<p>This demo allows you to experiment with the quota management rules enforced by browsers.

The browser will likely reduce the quota assigned to you if you send too many notifications not visible to the user, or if the user dismisses your visible notifications.
Try closing this page after selecting one of the two options ("visible" or "invisible") with a high number of notifications.</p>

<ul>
  <li>If you selected visible notifications, you will see them appear one after another on your screen. The quota will not be enforced;</li>
  <li>If you selected invisible notifications, you won't see them on your screen, but you will see the number at the bottom of this page updated once you re-open the page to indicate how many invisible notifications were sent before reaching the quota.</li>
</ul>

<form>
Number of notifications to send: <input id="notification-num" type="number" value="50"></input>
</form>

<button id="visible">Visible notifications</button> <button id="invisible">Invisible notifications</button>

<p>Received <span id="sent-visible">0</span> visible notifications
Received <span id="sent-invisible">0</span> invisible notifications</p>
<button id="clear">Clear</button>

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

  document.getElementById('visible').onclick = function() {
    // 通知服务端发送一个消息，service worker需要会用到可见的消息
    askForNotifications(true);
  };
  document.getElementById('invisible').onclick = function() {
    // 通知服务端发送一个消息，service worker不会用到可见的消息
    askForNotifications(false);
  };
  // 清除缓存‘notifications’的内容（存储了接收到的可见/不可见的消息数）
  document.getElementById('clear').onclick = function() {
    window.caches.open('notifications').then(function(cache) {
      Promise.all([
        cache.put(new Request('invisible'), new Response('0', {
          headers: {
            'content-type': 'application/json'
          }
        })),
        cache.put(new Request('visible'), new Response('0', {
          headers: {
            'content-type': 'application/json'
          }
        })),
      ]).then(function() {
        updateNumbers();
      });
    });
  };

});

function updateNumbers() {
  // 从‘notifications’缓存中读取消息数，更新UI界面
  window.caches.open('notifications').then(function(cache) {
    ['visible', 'invisible'].forEach(function(type) {
      cache.match(type).then(function(response) {
        response.text().then(function(text) {
          document.getElementById('sent-' + type).textContent = text;
        });
      });
    });
  });
}
window.onload = function() {
  // 定时更新接收到的消息数
  updateNumbers();
  setInterval(updateNumbers, 1000);
};
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
  // 发送若干推送消息，用请求参数判断service worker是否需要显示可见的消息
  app.post(route + 'sendNotification', function(req, res) {
    const subscription = req.body.subscription;
    const payload = null;
    const options = {
      TTL: 200
    };

    let num = 1;

    let promises = [];

    let intervalID = setInterval(function() {
      promises.push(webPush.sendNotification(subscription, payload, options));

      if (num++ === Number(req.body.num)) {
        clearInterval(intervalID);

        Promise.all(promises)
        .then(function() {
          res.sendStatus(201);
        })
        .catch(function(error) {
          res.sendStatus(500);
          console.log(error);
        })
      }
    }, 1000);
  });
};
```

service-worker.js

```js
var CACHE_NAME = 'notifications';
// 在on install阶段创建‘notifications’缓存字段用来存储接收到的消息数量
self.addEventListener('install', function(event) {
  event.waitUntil(
    caches.open(CACHE_NAME).then(function(cache) {
      // 我们创建一个假的请求和响应来存储信息
      return Promise.all([
        cache.put(new Request('invisible'), new Response('0', {
          headers: {
            'content-type': 'application/json'
          }
        })),
        cache.put(new Request('visible'), new Response('0', {
          headers: {
            'content-type': 'application/json'
          }
        })),
      ]);
    })
  );
});

self.addEventListener('activate', function(event) {
  event.waitUntil(self.clients.claim);
});

function updateNumber(type) {
  // 根据‘type’字段是visible或者invisible分别更新接收到消息的数量
  return caches.open(CACHE_NAME).then(function(cache) {
    return cache.match(type).then(function(response) {
      return response.json().then(function(notificationNum) {
        var newNotificationNum = notificationNum + 1;

        return cache.put(
          new Request(type),
          new Response(JSON.stringify(newNotificationNum), {
            headers: {
              'content-type': 'application/json',
            },
          })
        ).then(function() {
          return newNotificationNum;
        });
      });
    });
  });
}
// 注册push事件
self.addEventListener('push', function(event) {
  // 将event.data（一个PushMessageData类型的对象）解析为JSON对象
  var visible = event.data ? event.data.json() : false;
  // 在‘notifications’缓存完成更新前保持service worker在工作状态；如果visible字段是true则创建推送消息
  if (visible) {
    event.waitUntil(updateNumber('visible').then(function(num) {
      return self.registration.showNotification('ServiceWorker Cookbook', {
        body: 'Received ' + num + ' visible notifications',
      });
    }));
  } else {
    event.waitUntil(updateNumber('invisible'));
  }
});
```



