# Cache, update and refresh

本案例演示如何用service worker从缓存获取内容以达到快速响应的目的，同时从网络更新缓存条目。当网络请求response完成后自动更新界面。

# 难度等级

初级

# 使用场景

当你希望立即显示在后台检索资源内容，而且一旦有新内容可用则选择将其显示。

# 解决方案

响应来自缓存的内容，但同时发送网络请求以更新缓存条目并通知UI更新最新的内容。

# 方案类型

缓存策略

# 示例代码

controlled.js

```js
var CACHE = 'cache-update-and-refresh';

if ('serviceWorker' in navigator) {
  navigator.serviceWorker.onmessage = function (evt) {
    var message = JSON.parse(evt.data);

    var isRefresh = message.type === 'refresh';
    var isAsset = message.url.includes('asset');
    var lastETag = localStorage.currentETag;
    // ETag头字段中通常包含资源的hash值，因此它是检验资源是否最新的一种有效方式
    var isNew =  lastETag !== message.eTag;
    
    if (isRefresh && isAsset && isNew) {
      // 跳过第一次
      if (lastETag) {
        // 通知用户更新
        notice.hidden = false;
      }
      /*
        出于教学目的eTag信息存储在离线存储中，service worker从这里检索eTag的头字段信息，这样可以让实现逻辑简单点
      */
      localStorage.currentETag = message.eTag;
    }
  };
  var notice = document.querySelector('#update-notice');
  var update = document.querySelector('#update');
  update.onclick = function (evt) {
    var img = document.querySelector('img');
    evt.preventDefault();
    
    caches.open(CACHE)
    // 获取更新返回内容
    .then(function (cache) {
      return cache.match(img.src);
    })
    // 将response body抽象为blob对象
    .then(function (response) {
      return response.blob();
    })
    // 更新image内容
    .then(function (bodyBlob) {
      var url = URL.createObjectURL(bodyBlob);
      img.src = url;
      notice.hidden = true;
    });
  };
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
var CACHE = 'cache-update-and-refresh';
// 在on install事件中缓存资源
self.addEventListener('install', function(evt) {
  console.log('The service worker is being installed.');
  // 调用cache的addAll方法将所有资源缓存，让service worker保持在installing状态只到得到返回promise对象的resolve执行
  evt.waitUntil(caches.open(CACHE).then(function (cache) {
    cache.addAll([
      './controlled.html',
      './asset'
    ]);
  }));
});
// 在on fetch事件中使用缓存内容响应，但是同时从服务端更新最新内容
self.addEventListener('fetch', function(evt) {
  console.log('The service worker is serving the asset.');
  // 使用respondWith方法响应ASAP
  evt.respondWith(fromCache(evt.request));
  // 使用waitUntil方式防止worker在缓存更新之前被kill掉
  evt.waitUntil(
    update(evt.request)
    .then(refresh)
  );
});
/*
打开缓存，从中查找请求的资源。
需要注意：如果没有找到对应资源仍然会返回promise的resovle，只不过得到的value是undefine
*/
function fromCache(request) {
  return caches.open(CACHE).then(function (cache) {
    return cache.match(request);
  });
}
/*
打开缓存时执行网络请求和存储响应的数据
*/
function update(request) {
  return caches.open(CACHE).then(function (cache) {
    return fetch(request).then(function (response) {
      return cache.put(request, response.clone()).then(function () {
        return response;
      });
    });
  });
}
// 更新客户端
function refresh(response) {
  // 向客户端发送message
  return self.clients.matchAll().then(function (clients) {
    clients.forEach(function (client) {
      // encode更新到的资源内容，加入eTag使客户端可以识别内容是否改变
      var message = {
          type: 'refresh',
          url: response.url,
          /*
            注意：并不是所有的服务端response都包含eTag信息，
            如果没有那么你就需要使用其他的缓存头字段或者自定义字段来判断内容是否更新
          */
          eTag: response.headers.get('ETag')
      };
      // 通知客户端更新message
      client.postMessage(JSON.stringify(message));
    });
  });
}
```



