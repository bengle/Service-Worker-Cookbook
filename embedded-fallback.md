# Embedded fallback

本案例主要是使用service worker在资源缺失的情况下回调响应嵌入的内容。

# 难度等级

中级

# 应用场景

当业务需要确认用户即使在网络不可用的情况也总的得到某些内容。

# 解决方案

当请求资源失败时回调返回嵌入好的内容。

# 方案类型

缓存策略

# 示例代码

controlled.js

```js
/*
  为了能够从一开始就提供回调，我们只能在service worker准备就绪后加载图像。
  另外，service worker应该开始就做拦截而无需等待当前客户端关闭。
*/

// 检验controller是一种有效判断当前页面是否在service worker控制的方法
if (navigator.serviceWorker.controller) {
  loadImage();
} else {
// 如果没有worker控制，那么就等它出现
  navigator.serviceWorker.oncontrollerchange = function() {
  // 监控controller直到它可以拦截请求
  this.controller.onstatechange = function() {
      if (this.state === 'activated') {
        loadImage();
      }
    };
  };
}

function loadImage() {
  document.querySelector('img').src = './missing';
}
```

index.js

```js
/*
注册service worker并限制其作用域在controlled开头的URL，需要注意的是这个作用域只是个前缀并非具体路径。
首先请求地址被转化为绝对路径URL，然后通过测试请求URL的前缀是否符合作用域设置来确定当前资源是否在service worker控制范围。
*/
navigator.serviceWorker.register('service-worker.js', {
  scope: './controlled'
});

var referenceIframe = document.getElementById('reference');
var sampleIframe = document.getElementById('sample');
// iframe刷新时重新设置其高度
referenceIframe.onload = fixHeight;
sampleIframe.onload = fixHeight;
// 计算iframe内容高度匹配调整iframe高度
function fixHeight(evt) {
  var iframe = evt.target;
  var document = iframe.contentWindow.document.documentElement;
  iframe.style.height = document.getClientRects()[0].height + 'px';

  if (window.parent !== window) {
    window.parent.document.body.dispatchEvent(new CustomEvent('iframeresize'));
  }
}
```

server.js

```js
var MAX_IMAGES = 50;
var imageNumber = 0;

module.exports = function(app, route) {
  app.get(route + 'asset', function(req, res) {
    serveImage(res, 10000);
  });
};

var lastUpdate = -Infinity;

function serveImage(res, timeout) {
  var now = Date.now();
  if (now - lastUpdate > timeout) {
    imageNumber = (imageNumber + 1) % MAX_IMAGES;
    lastUpdate = Date.now();
  }
  var imageName = 'picture-' + (imageNumber + 1) + '.png';
  res.sendFile(imageName, { root: './imgs/random/' });
}
```

service-worker.js

```js
var CACHE = 'offline-fallback';
// 在oninstall事件中缓存不可用的资源
self.addEventListener('install', function(evt) {
  console.log('The service worker is being installed.');
  // 调用cache的addAll方法将所有资源缓存，让service worker保持在installing状态只到得到返回promise对象的resolve执行
  evt.waitUntil(precache().then(function () {
  // 因为我们想要service worker处于激活状态并控制客户端，因此这里要skipWaiting，取消等待它重新加载
    return self.skipWaiting();
  }));
});

self.addEventListener('activate', function (evt) {
  /* 
    self.clients.claim()会让service worker马上拦截请求
    除了self.skipWaiting()之外，我们还需要从一开始就允许调用回调
  */
  evt.waitUntil(self.clients.claim());
});

self.addEventListener('fetch', function(evt) {
  console.log('The service worker is serving the asset.');
  // 这里可以使用你想用的任何策略，如果失败则启用回调
  evt.respondWith(networkOrCache(evt.request).catch(function () {
    return useFallback();
  }));
});
/*
开启一个缓存，将一组资源传递给addAll方法把它们全部添加到缓存中，当所有资源缓存完毕返回promise的resolve
*/
function precache() {
  return caches.open(CACHE).then(function (cache) {
    return cache.addAll([
      './controlled.html'
    ]);
  });
}
/*
  这个个简化版的网络请求，在错误控制中立即使用缓存
*/
function networkOrCache(request) {
  return fetch(request).then(function (response) {
    return response.ok ? response : fromCache(request);
  })
  .catch(function () {
    return fromCache(request);
  });
}
// 回调返回是一个svg图片
var FALLBACK =
    '<svg xmlns="http://www.w3.org/2000/svg" width="200" height="180" stroke-linejoin="round">' +
    '  <path stroke="#DDD" stroke-width="25" d="M99,18 15,162H183z"/>' +
    '  <path stroke-width="17" fill="#FFF" d="M99,18 15,162H183z" stroke="#eee"/>' +
    '  <path d="M91,70a9,9 0 0,1 18,0l-5,50a4,4 0 0,1-8,0z" fill="#aaa"/>' +
    '  <circle cy="138" r="9" cx="100" fill="#aaa"/>' +
    '</svg>';
// 因为使用嵌入回调，请求就不会失败
function useFallback() {
  return Promise.resolve(new Response(FALLBACK, { headers: {
    'Content-Type': 'image/svg+xml'
  }}));
}
/*
打开缓存，从中查找请求的资源。
需要注意：如果没有找到对应资源仍然会返回promise的resovle，只不过得到的value是undefine
*/
function fromCache(request) {
  return caches.open(CACHE).then(function (cache) {
    return cache.match(request).then(function (matching) {
      return matching || Promise.reject('request-not-in-cache');
    });
  });
}
```

index.html

```html
<!doctype html>
<html>
<head>
  <meta charset="utf-8">
  <title>Cache only - ServiceWorker Cookbook</title>
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <style>
   iframe {
     display: block;
     margin: 1rem;
     box-shadow: 2px 2px 10px 0px #eee inset;
     width: 50%;
     min-height: 500px;
   }
   #comparison {
     display: flex;
     direction: row;
     margin-bottom: 2rem;
   }
   button {
     width: 100%;
     border: none;
     background-color: #279CD7;
     color: white;
     font-size: large;
     padding: 1em;
     cursor: pointer;
   }
  </style>
</head>
<body>
  <h1>Embedded fallback</h1>
  <div id="comparison">
    <iframe src="./non-controlled.html" id="reference"></iframe>
    <iframe src="./controlled.html" id="sample"></iframe>
  </div>
  <p>The images in these iframe points to the same asset in the server. But the first is controlled by the service worker and the second is not.</p>

<script src="./index.js"></script>
</body>
</html>
```

non-controlled.html

```html
<!doctype html>
<html>
<head>
  <meta charset="utf-8">
  <title>Network or cache: non controlled page - ServiceWorker Cookbook</title>
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <style>
   body {
     text-align: center;
   }
  </style>
</head>
<body>
  <h1>Always synchronized</h1>
  <img src="./asset" alt="sample asset">
  <p>This image originates from a non controlled page so, if you reload, it will be always synced with the version in the server.</p>
</body>
</html>
```

controlled.html

```html
<!doctype html>
<html>
<head>
  <meta charset="utf-8">
  <title>Network or cache: controlled page - ServiceWorker Cookbook</title>
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <style>
   body {
     text-align: center;
   }
  </style>
</head>
<body>
  <h1>Network or cache</h1>
  <img src="./asset" alt="sample asset" />
  <p>This image request originates from a controlled page so the image will
    be served by the service worker. The service worker will try to retrieve
    the most updated content from network but if the answer does not arrive
    before a timeout, it will fall back to the cached content. Try to
    <a href="https://developers.google.com/web/tools/chrome-devtools/profile/network-performance/network-conditions?hl=en">
    adjust throttling</a> to GPRS to see the effects of network latency.</p>
</body>
</html>
```



