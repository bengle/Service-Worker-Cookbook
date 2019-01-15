# JSON Cache

本案例主要演示在service worker安装期间获取JSON文件并将所有资源添加到缓存，方案中使用到了立即声明激活service worker的技巧。

# 难度等级

初级

# 应用场景

在Offline Status章节中我们将要缓存的文件作为数组保存在js文件中，大多数时候我们并不希望这么做，更通用的方法是将这些信息保存在其他位置，这样将非常有利于做版本控制。

# 功能和用法

* 注册一个service worker
* service worker获取并解析一个JSON文件，里面包含了需要缓存的关键资源列表
* service worker获取并缓存资源文件并代理响应

你唯一需要操作的是打开页面完成页面初始化，之后service worker会完成install，并将资源缓存。

# 兼容性

经测试可以在以下版本以上浏览器正常运行：

* Firefox Nightly 44.0a1
* Chrome Canary 48.0.2533.0
* Opera 32.0

# 方案类型

离线

# 代码示例

index.html

```html
<!doctype html>
<html>
<head>
  <meta charset="utf-8">
  <title>JSON Cache - ServiceWorker Cookbook</title>
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <style>
    body {
      font-family: monospace;
      font-size: 1rem;
      white-space: pre-wrap;
    }
    img {
      max-width: 200px;
      display: inline-block;
    }
  </style>
</head>
<body>

  <p>This demo uses a service worker which loads a JSON file representing files to be cached by the Service Worker.  Once the JSON file is loaded and parsed, files are placed into the cache via the Service Worker.</p>

  <h2>Assets to Cache</h2>
  <img src="random-1.png" alt="">
  <img src="random-2.png" alt="">
  <img src="random-3.png" alt="">
  <img src="random-4.png" alt="">
  <img src="random-5.png" alt="">
  <img src="random-6.png" alt="">


  <script src="./index.js"></script>
</body>
</html>
```

index.js

```js
// 注册service worker
navigator.serviceWorker.register('service-worker.js', {
  scope: '.'
}).then(function(registration) {
  console.log('The service worker has been registered ', registration);
});
```

file-to-cache.json

```js
[
  "index.html",
  "index.js",
  "random-1.png",
  "random-2.png",
  "random-3.png",
  "random-4.png",
  "random-5.png",
  "random-6.png"
]
```

service-worker.js

```js
var CACHE_NAME = 'dependencies-cache';
/*
  演示install过程步骤：
  1.从服务器加载JSON文件
  2.返回结果parse成JSON对象
  3.将列表里的文件添加到缓存
*/
self.addEventListener('install', function(event) {
  console.log('[install] Kicking off service worker registration!');

  event.waitUntil(
    caches.open(CACHE_NAME)
      .then(function(cache) {
        // 在缓存open后，加载files-to-cahce.json文件到缓存，这个文件包含了需要添加到缓存的关键资源
        return fetch('files-to-cache.json').then(function(response) {
          return response.json();
        }).then(function(files) {
          console.log('[install] Adding files from JSON file: ', files);
          // 用cache.addAll缓存所有资源文件
          return cache.addAll(files);
        });
      })
      .then(function() {
        console.log(
          '[install] All required resources have been cached;',
          'the Service Worker was successfully installed!'
        );
        // 强制激活service worker
        return self.skipWaiting();
      })
  );
});

self.addEventListener('fetch', function(event) {
  event.respondWith(
    caches.match(event.request)
      .then(function(response) {
        // 资源已经在缓存中直接从缓存返回
        if (response) {
          console.log(
            '[fetch] Returning from Service Worker cache: ',
            event.request.url
          );
          return response;
        }
        // 资源不在缓存中则从服务端fetch到加入缓存
        console.log('[fetch] Returning from server: ', event.request.url);
        return fetch(event.request);
      }
    )
  );
});
self.addEventListener('activate', function(event) {
  console.log('[activate] Activating service worker!');
  console.log('[activate] Claiming this service worker!');
  // 通过claim()方法强制service worker触发controllerchange事件
  event.waitUntil(self.clients.claim());
});
```



