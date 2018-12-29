# Immediate Claim {#immediate-claim}

本章案例演示怎样让service worker立即接管页面而不需要等待navigation事件。

# 难度等级

初级

# 应用场景

通常注册service worker都必须等待navigation事件触发之后才能工作，这里教你一个小技巧可以让service worker在install之后可以立即开始工作。

# 特点和用法

* 注册一个service worker
* 如果有老的缓存删除之
* 马上声明service worker

# 方案类型

通用情况

# 代码示例

index.html

```html

<!doctype html>
<html>
<head>
  <meta charset="utf-8">
  <title>Immediate Claim - ServiceWorker Cookbook</title>
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
[Version <em id="version">-</em>]

A random picture by <a href="http://lorempixel.com" target="_blank">lorempixel.com</a>:

  <img src="./default.jpg" id="random" width="200" height="100">

<button id="update">Update ServiceWorker</button>

onLoad:
  <span id="onload">Pending …</span>
Registered:
  <span id="register">Nothing registered</span>
onControllerchange:
  <span id="oncontrollerchange">Not fired</span>

<script src="./index.js"></script>
</body>
</html>

```

index.js

```js
// 该方法模拟简单的界面更新代码
// 反映UI中当前的状态，更新图片和版本号
function fetchUpdate() {
  var img = document.querySelector('#random');
  img.src = img.src = 'random.jpg?' + Date.now();
  fetch('./version').then(function(response) {
    return response.text();
  }).then(function(text) {
    debug(text, 'version');
  });
}
// service worker在onload之后接管网站，并处理离线调用
if (navigator.serviceWorker.controller) {
  var url = navigator.serviceWorker.controller.scriptURL;
  console.log('serviceWorker.controller', url);
  debug(url, 'onload');
  fetchUpdate();
} else {
  // 注册一个service worker
  navigator.serviceWorker.register('service-worker.js', {
    scope: './'
  }).then(function(registration) {
    debug('Refresh to allow ServiceWorker to control this client', 'onload');
    debug(registration.scope, 'register');
  });
}

navigator.serviceWorker.addEventListener('controllerchange', function() {
  var scriptURL = navigator.serviceWorker.controller.scriptURL;
  console.log('serviceWorker.onControllerchange', scriptURL);
  debug(scriptURL, 'oncontrollerchange');
  fetchUpdate();
});

document.querySelector('#update').addEventListener('click', function() {
  navigator.serviceWorker.ready.then(function(registration) {
    registration.update().then(function() {
      console.log('Checked for update');
    }).catch(function(error) {
      console.error('Update failed', error);
    });
  });
});

function debug(message, element) {
  var target = document.querySelector('#' + element || 'log');
  target.textContent = message;
}
```

server.js

```js
var path = require('path');
var swig = require('swig');
var request = require('request');

module.exports = function autoClaim(app, route) {
  var tpl = swig.compileFile(path.join(__dirname, './service-worker.js'));

  app.get(route + 'service-worker.js', function getServiceWorker(req, res) {
    // 以10秒的京都获取当前事件，将生成的service worker每10秒更新一次
    var nowMinute = new Date();
    nowMinute.setSeconds(Math.floor(nowMinute.getSeconds() / 5) * 5);
    // 显示内容替换version值
    var buffer = tpl({
      version: [
        nowMinute.getFullYear(),
        nowMinute.getMonth(),
        nowMinute.getDate(),
        [
          nowMinute.getHours(),
          nowMinute.getMinutes(),
          nowMinute.getSeconds()
        ].join(':')
      ].join('-')
    });
    res.type('js').send(buffer);
  });

  app.get(route + 'random-cached.jpg', function getRandom(req, res) {
    request('http://lorempixel.com/200/100/').pipe(res);
  });
};
```

service-worker.js

```js
var version = '{{ version }}';

self.addEventListener('install', function(event) {
  console.log('[ServiceWorker] Installed version', version);
  event.waitUntil(
    fetch('./random-cached.jpg').then(function(response) {
      return caches.open(version).then(function(cache) {
        console.log('[ServiceWorker] Cached random.jpg for', version);
        // 这里要返回一个Promise对象，以便在缓存更新后触发skipWaiting()
        return cache.put('random.jpg', response);
      });
    }).then(function() {
      // skipWaiting()方法强制将service worker编程激活状态并触发onactivate事件
      // 与clients.claim()方法一起允许service worker在客户端立即生效
      console.log('[ServiceWorker] Skip waiting on install');
      return self.skipWaiting();
    })
  );
});
// onactivate事件通常在install且刷新页面后调用
// 因为我们在install中调用skipWaiting()方法，因此会立即调用activate
self.addEventListener('activate', function(event) {
  // 出于调试目的，这里列举了所有受控的客户端
  self.clients.matchAll({
    includeUncontrolled: true
  }).then(function(clientList) {
    var urls = clientList.map(function(client) {
      return client.url;
    });
    console.log('[ServiceWorker] Matching clients:', urls.join(', '));
  });

  event.waitUntil(
    // 删除所有与当前版本不匹配的缓存
    caches.keys().then(function(cacheNames) {
      return Promise.all(
        cacheNames.map(function(cacheName) {
          if (cacheName !== version) {
            console.log('[ServiceWorker] Deleting old cache:', cacheName);
            return caches.delete(cacheName);
          }
        })
      );
    }).then(function() {
      // claim()方法将scope符合的所有客户端service worker设置为激活状态，同时会触发客户端的oncontrollerchange事件
      console.log('[ServiceWorker] Claiming clients for version', version);
      return self.clients.claim();
    })
  );
});

self.addEventListener('fetch', function(event) {
  if (event.request.url.includes('/random.jpg')) {
    console.log('[ServiceWorker] Serving random.jpg for', event.request.url);
    event.respondWith(
      caches.open(version).then(function(cache) {
        return cache.match('random.jpg').then(function(response) {
          if (!response) {
            console.error('[ServiceWorker] Missing cache!');
          }
          return response;
        });
      })
    );
  }
  if (event.request.url.includes('/version')) {
    event.respondWith(new Response(version, {
      headers: {
        'content-type': 'text/plain'
      }
    }));
  }
});
```



