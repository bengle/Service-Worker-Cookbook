# Offline Status

本案例演示了如何缓存关键资源，并告知用户他们可以在离线情况下得到和在线一样的用户体验。

# 难度等级

初级

# 应用场景

这是最基本的service worker应用场景：将文件缓存到本地让用户离线情况下仍然可以使用。在这个demo中我们增加了一个提示，告诉用户在离线情况下仍然可以正常使用。

# 功能和用法

* 注册一个service worker
* 监测所需资源的缓存状态
* 当资源缓存完毕后通知用户

用户唯一的操作就是访问页面完成页面初始化加载，页面加载后service worker会完成安装并将所需资源缓存。

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
  <title>Offline Status - ServiceWorker Cookbook</title>
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <link rel="stylesheet" type="text/css" href="style.css">
  <style>
    body {
      font-family: monospace;
      font-size: 1rem;
      white-space: pre-wrap;
    }
  </style>
</head>
<body>

  <p>The goal of this recipe is to create an app which can be used both online and offline with the help of a ServiceWorker.  Once the ServiceWorker has cached assets and becomes activated, the user will receive a notification that they can then go offline and use the app!</p>

  <img src="random-1.png" alt="" id="logoImage">

  <button id="randomButton">Show Another Random Logo</button>

  <button id="clearAndReRegister">View Complete Demo Again</button>

  <div class="hidden notification" id="offlineNotification">You are now safe to go offline!</div>

  <script src="./index.js"></script>
  <script src="./app.js"></script>
</body>
</html>
```

app.js

```js
// 此文件执行需确保app可以离线运行
document.getElementById('randomButton').addEventListener('click', function() {
  var image = document.getElementById('logoImage');
  var currentIndex = Number(image.src.match('random-([0-9])')[1]);
  var newIndex = getRandomNumber();
  // 确保我们接收到的图片与当前图片不一样
  while (newIndex === currentIndex) {
    newIndex = getRandomNumber();
  }

  image.src = 'random-' + newIndex + '.png';

  function getRandomNumber() {
    return Math.floor(Math.random() * 6) + 1;
  }
});

document.getElementById('clearAndReRegister').addEventListener('click',
  function() {
    navigator.serviceWorker.getRegistration().then(function(registration) {
      registration.unregister();
      window.location.reload();
    });
  }
);
```

index.js

```js
// 注册service worker
navigator.serviceWorker.register('service-worker.js', {
  scope: '.'
}).then(function(registration) {
  console.log('The service worker has been registered ', registration);
});
// 监听service worker声明状态
navigator.serviceWorker.addEventListener('controllerchange', function(event) {
  console.log(
    '[controllerchange] A "controllerchange" event has happened ' +
    'within navigator.serviceWorker: ', event
  );
  // 监听service worker状态变化
  navigator.serviceWorker.controller.addEventListener('statechange',
    function() {
      console.log('[controllerchange][statechange] ' +
        'A "statechange" has occured: ', this.state
      );
      // 一旦service worker的状态编程activated通知用户他可以离线操作了
      if (this.state === 'activated') {
        document.getElementById('offlineNotification')
        .classList.remove('hidden');
      }
    }
  );
});
```

service-worker.js

```js
var CACHE_NAME = 'dependencies-cache';
// 这是需要缓存的文件，实际操作时大家可以自行添加和修改
var REQUIRED_FILES = [
  'random-1.png',
  'random-2.png',
  'random-3.png',
  'random-4.png',
  'random-5.png',
  'random-6.png',
  'style.css',
  'index.html',
  'index.js',
  'app.js'
];

self.addEventListener('install', function(event) {
  event.waitUntil(
    // install阶段将所需资源加载到缓存
    caches.open(CACHE_NAME)
      .then(function(cache) {
        // 缓存所有资源文件
        console.log('[install] Caches opened, adding all core components' +
          'to cache');
        return cache.addAll(REQUIRED_FILES);
      })
      .then(function() {
        console.log('[install] All required resources have been cached, ' +
          'we\'re good!');
        return self.skipWaiting();
      })
  );
});

self.addEventListener('fetch', function(event) {
  event.respondWith(
    caches.match(event.request)
      .then(function(response) {
        // 资源已经缓存则直接从缓存返回
        if (response) {
          console.log(
            '[fetch] Returning from ServiceWorker cache: ',
            event.request.url
          );
          return response;
        }
        // 资源不在缓存，从线上服务器fetch当前资源
        console.log('[fetch] Returning from server: ', event.request.url);
        return fetch(event.request);
      }
    )
  );
});

self.addEventListener('activate', function(event) {
  console.log('[activate] Activating ServiceWorker!');
  console.log('[activate] Claiming this ServiceWorker!');
  // 通过claim()方法强制触发service worker的“controllerchange”事件
  event.waitUntil(self.clients.claim());
});
```



