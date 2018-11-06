# 网络与缓存

# 网络与缓存

本案例我们用service worker通过网络更新最新的内容，但是当网络拥堵的时候则使用缓存的内容替代。

# 难度

初级

# 使用场景

当你想显示最新的内容，但是最好加载速度也尽可能的快

# 解决方案

在请求网络资源的时候设置一个定时器，当网络请求没有按时返回时则调用缓存数据响应此请求。

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

// 当service worker激活成功加载受控或非受控页面
navigator.serviceWorker.ready.then(reload);

// 每当iframe重新加载页面时设置其高度
var referenceIframe = document.getElementById('reference');
var sampleIframe = document.getElementById('sample');
referenceIframe.onload = fixHeight;
sampleIframe.onload = fixHeight;

// 按钮点击时刷新两个iframe
var reloadButton = document.querySelector('#reload');
reloadButton.onclick = reload;

// 加载受控和非受控的iframe
function loadIframes() {
    referenceIframe.src = './non-controlled.html';
    sampleIframe.src = './controlled.html';
}
// 计算iframe内容高度并重新设置iframe高度匹配其内容
function fixHeight(evt) {
    var iframe = evt.target;
    var document = iframe.contentWindow.document.documentElement;
    iframe.style.height = document.getClientRects()[0].height + 'px';
    if (window.parent !== window) {
        window.parent.document.body.dispatchEvent(new CustomEvent('iframeresize'));
    }
}
// 刷新iframe
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
var CACHE = 'network-or-cache';
// oninstall之后缓存资源
self.addEventListener('install', function(evt) {
  console.log('The service worker is being installed.');
  // 让service worker保持在installing状态只到得到返回promise对象的resolve执行
  evt.waitUntil(precache());
});

// 设置fetch事件，使用缓存响应但是同时从服务端更新最新资源
self.addEventListener('fetch', function(evt) {
  console.log('The service worker is serving the asset.');
  evt.respondWith(fromNetwork(evt.request, 400).catch(function () {
    return fromCache(evt.request);
  }));
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
设置时间上限的网络请求，如果网络没有响应或者响应超时则返回promise的reject，如果响应成功则执行fullfill
*/
function fromNetwork(request, timeout) {
  return new Promise(function (fulfill, reject) {
    var timeoutId = setTimeout(reject, timeout);
    
    fetch(request).then(function (response) {
      clearTimeout(timeoutId);
      fulfill(response);
    }, reject);
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
```



