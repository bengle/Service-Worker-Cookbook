# Offline Fallback

本章演示在用户离线情况下如何使用缓存响应资源，达到可以离线使用应用的目的。

# 难度等级

初级

# 用例分析

一个比较麻烦的事情是当你离线情况下的提示因浏览器而异：

* 浏览器品牌多样
* 不同浏览器显示效果也不同
* 提示信息不一定经过本地化

更好的解决方法是给用户统一显示一段保存在缓存中的离线代码片段。

# 功能和用法

* 注册一个servece worker
* 缓存offline.html文件
* 如果网络不通，则用offline.html做响应显示

# 方案类型

离线

# 代码示例

index.html

```html

<!doctype html>
<html>
<head>
  <meta charset="utf-8">
  <title>Offline Fallback - ServiceWorker Cookbook</title>
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
<strong>Yay, you are online!</strong>
Now go offline and <a href="#" id="refresh">refresh</a>!

Registered ServiceWorker:	<span id="register">Did not register</span>
Active Controller:	<span id="controller">Did not activate</span>

ServiceWorker logs:
<div id="log">Here be logs: <br></div>

<button id="clearAndReRegister">View Complete Demo Again</button>

<script src="./index.js"></script>
</body>
</html>

```

index.js

```js
if (navigator.serviceWorker.controller) {
  debug(
    navigator.serviceWorker.controller.scriptURL +
    ' (onload)', 'controller'
  );
  debug(
    'An active service worker controller was found, ' +
    'no need to register'
  );
} else {
  navigator.serviceWorker.register('service-worker.js', {
    scope: './'
  }).then(function(reg) {
    debug(reg.scope, 'register');
    debug('Service worker change, registered the service worker');
  });
}
document.querySelector('#refresh').search = Date.now();
function debug(message, element, append) {
  var target = document.querySelector('#' + (element || 'log'));
  target.textContent = message + ((append) ? ('/n' + target.textContent) : '');
}
document.getElementById('clearAndReRegister').addEventListener('click',
  function() {
    navigator.serviceWorker.getRegistration().then(function(registration) {
      registration.unregister();
      window.location.reload();
    });
  }
);
```

service-worker.js

```js
self.addEventListener('install', function(event) {
  var offlineRequest = new Request('offline.html');
  event.waitUntil(
    fetch(offlineRequest).then(function(response) {
      return caches.open('offline').then(function(cache) {
        console.log('[oninstall] Cached offline page', response.url);
        return cache.put(offlineRequest, response);
      });
    })
  );
});

self.addEventListener('fetch', function(event) {
  var request = event.request;
  if (request.method === 'GET') {
    console.error(
          '[onfetch] Failed. Serving cached offline fallback ' +
          error
        );
        return caches.open('offline').then(function(cache) {
          return cache.match('offline.html');
        });
      })
    );
  }
});
```



