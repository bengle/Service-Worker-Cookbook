# 缓存及其更新

本案例演示如何用service worker从缓存获取内容以达到快速响应的目的，同时从网络更新缓存条目。

# 难度等级

初级

# 使用场景

需要快速响应而并不介意内容与服务端实时一致。

# 解决方案

返回缓存内容，同时发送网络请求获取新数据更新缓存条目，确保用户下次访问该页面时将看到最新内容。

# 方案类型

缓存策略

# 示例代码

index.js

```js
/*
注册service worker并限制其作用域在controlled开头的URL，需要注意的是这个作用域只是个前缀并非具体路径。
首先请求地址被转化为绝对路径URL，然后通过测试请求URL的前缀是否符合作用域设置来确定当前资源是否在service worker控制范围。
*/

navigator.serviceWorker.register('service-worker.js', {
  scope: './controlled'
});
// 当service worker激活后加载受控和非受控两个iframe
navigator.serviceWorker.ready.then(reload);

var referenceIframe = document.getElementById('reference');
var sampleIframe = document.getElementById('sample');
// iframe刷新时重新设置其高度
referenceIframe.onload = fixHeight;
sampleIframe.onload = fixHeight;
// 点击按钮刷新iframe
var reloadButton = document.querySelector('#reload');
reloadButton.onclick = reload;
// 加载受控和非受控两个iframe页面
function loadIframes() {
  referenceIframe.src = './non-controlled.html';
  sampleIframe.src = './controlled.html';
}
// 计算iframe内容高度匹配调整iframe高度
function fixHeight(evt) {
  var iframe = evt.target;
  var document = iframe.contentWindow.document.documentElement;
  iframe.style.height = document.getClientRects()[0].height + 'px';
  if (window.parent !== window) {
    window.parent.document.body.dispatchEvent(new CustomEvent('iframeresize'));
  }
}
// 刷新两个iframe
function reload() {
  referenceIframe.contentWindow.location.reload();
  sampleIframe.contentWindow.location.reload();
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

var lastUpdate = 0;

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
var CACHE = 'cache-and-update';
// 在on install事件中缓存资源
self.addEventListener('install', function(evt) {
  console.log('The service worker is being installed.');
  // 让service worker保持在installing状态只到得到返回promise对象的resolve执行
  evt.waitUntil(precache());
});

self.addEventListener('fetch', function(evt) {
  console.log('The service worker is serving the asset.');

  evt.respondWith(fromCache(evt.request));

  evt.waitUntil(update(evt.request));
});
/*
开启一个缓存，将一组资源传递给addAll方法把它们全部添加到缓存中，当所有资源缓存完毕返回promise的resolve
*/
function precache() {
  return caches.open(CACHE).then(function (cache) {
    return cache.addAll([
      './controlled.html',
      './asset'
    ]);
  });
}
/*
打开缓存，从中查找请求的资源。
需要注意：如果没有找到对应资源仍然会返回promise的resovle，只不过得到的value是undefine
*/
function fromCache(request) {
  return caches.open(CACHE).then(function (cache) {
    return cache.match(request).then(function (matching) {
      return matching || Promise.reject('no-match');
    });
  });
}
/*
打开缓存时执行网络请求和存储响应的数据
*/
function update(request) {
  return caches.open(CACHE).then(function (cache) {
    return fetch(request).then(function (response) {
      return cache.put(request, response);
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
  <title>Network or cache - ServiceWorker Cookbook</title>
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <style>
   iframe {
     display: block;
     margin: 1rem;
     box-shadow: 2px 2px 10px 0px #eee inset;
     width: 50%;
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
  <h1>Network or cache</h1>
  <p>Try to adjust <a href="https://developers.google.com/web/tools/chrome-devtools/profile/network-performance/network-conditions" source="_blank">network throttling</a> to GPRS to see the image falling back to the cached content.</p>
  <div id="comparison">
    <iframe src="./non-controlled.html" id="reference"></iframe>
    <iframe src="./controlled.html" id="sample"></iframe>
  </div>
  <p>The images in these iframes point to the same asset in the server. But the first is controlled by the service worker and the second is not.</p>
  <p>In the server, the image is updated every 10 seconds, try to click on reload to cause new requests from the controlled and uncontrolled pages.</p>
  <p><button id="reload">Reload</button></p>

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



