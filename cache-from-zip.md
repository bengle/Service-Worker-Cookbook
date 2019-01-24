# Cache from ZIP

本案例演示如何从一个zipfile添加内容到缓存。

# 难度等级

中级

# 应用场景

作为一名web开发人员，我希望将应用程序作为数个单独的压缩包分发，这样既可以减少HTTP请求数量又可以提供一种隐式的方法来列出所有资源供离线使用。

# 解决方案

在安装service worker的时候，下载zipfile并解压，将资源加入缓存。

# 方案类型

离线扩展

# 代码示例

index.html

```html
<!doctype html>
<html>
<head>
  <meta charset="utf-8">
  <title>Cache from ZIP - ServiceWorker Cookbook</title>
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <style>
    body {
      font-family: monospace;
      font-size: 1rem;
      white-space: pre-wrap;
    }
    p {
      text-align: center;
    }
  </style>
</head>
<body>
  <section id="install-notice">
    We are going to simulate installing and uninstalling a web application
    in the browser. To do that, enable logs from service workers to be shown
    and click on install button.
    <button id="install">Install</button>
  </section>

  <section id="images" hidden>
    Choose from one the images to be loaded: <select>
      <option value="imgs/a.jpeg">Collaboration</option>
      <option value="imgs/b.jpeg">Loneliness</option>
      <option value="imgs/c.jpeg">Bread</option>
    </select> <button id="load-image">Load</button>
    <p><img src="" alt="" /></p>

    Now you can go offline, restart the browser and check how the page
    continues working. Once you convince yourself, click on uninstall. 
    <button id="uninstall">Uninstall</button>
  </section>

  <pre id="results"></pre>

<script src="./index.js"></script>
</body>
</html>
```

index.js

```js
var $ = document.querySelector.bind(document);
// 通过监测controller验证应用是否已经完成装载
// 我们假设一旦service worker控制页面，那么则认为应用已经加载完成
navigator.serviceWorker.getRegistration().then(function(registration) {
  if (registration && registration.active) {
    showImagesSection();
  }
});
// 在安装阶段，一旦service worker被激活就将image动态加载选项显示
navigator.serviceWorker.oncontrollerchange = function() {
  if (navigator.serviceWorker.controller) {
    logInstall('The application has been installed');
    showImagesSection();
  }
};
// 这里装载service worker，它负责下载资源压缩包，解压和将资源添加到缓存
$('#install').onclick = function() {
  navigator.serviceWorker.register('worker.js').then(function() {
    logInstall('Installing...');
  }).catch(function(error) {
    logInstall('An error happened during installing the service worker:');
    logInstall(error.message);
  });
};
// 这里的卸载只是卸载service worker并没有删除缓存的资源，这些资源实际上还是加载进来了，只不过没有service worker控制它们
$('#uninstall').onclick = function() {
  navigator.serviceWorker.getRegistration().then(function(registration) {
    if (!registration) { return; }
    registration.unregister()
      .then(function() {
        logUninstall('The application has been uninstalled');
        setTimeout(function() { location.reload(); }, 500);
      })
      .catch(function(error) {
        logUninstall('Error while uninstalling the service worker:');
        logUninstall(error.message);
      });
  });
};
// 加载图片只不过是在显示器上显示正确的URL
$('#load-image').onclick = function() {
  $('img').src = $('select').value;
};
// 一系列控制UI的工具方法
function showImagesSection() {
  $('#images').hidden = false;
  $('#install-notice').hidden = true;
}

function logInstall(what) {
  log(what, 'Install');
}

function logUninstall(what) {
  log(what, 'Uninstall');
}

function log(what, tag) {
  var label = '[' + tag + ']';
  console.log(label, what);
  $('#results').textContent += label + ' ' + what + '\n';
}
```

worker.js

```js
importScripts('./lib/zip.js');
importScripts('./lib/ArrayBufferReader.js');
importScripts('./lib/deflate.js');
importScripts('./lib/inflate.js');

var ZIP_URL = './package.zip';
zip.useWebWorkers = false;

self.oninstall = function(event) {
  event.waitUntil(
    fetch(ZIP_URL)
      .then(function(response) {
        return response.arrayBuffer();
      })
      .then(getZipReader)
      .then(cacheContents)
      .then(self.skipWaiting.bind(self)) // control clients ASAP
  );
};

self.onactivate = function(event) {
  event.waitUntil(self.clients.claim());
};

self.onfetch = function(event) {
  event.respondWith(openCache().then(function(cache) {
    return cache.match(event.request).then(function(response) {
      return response || fetch(event.request);
    });
  }));
};

function getZipReader(data) {
  return new Promise(function(fulfill, reject) {
    zip.createReader(new zip.ArrayBufferReader(data), fulfill, reject);
  });
}

function cacheContents(reader) {
  return new Promise(function(fulfill, reject) {
    reader.getEntries(function(entries) {
      console.log('Installing', entries.length, 'files from zip');
      Promise.all(entries.map(cacheEntry)).then(fulfill, reject);
    });
  });
}

function cacheEntry(entry) {
  if (entry.directory) { return Promise.resolve(); }

  return new Promise(function(fulfill, reject) {
    entry.getData(new zip.BlobWriter(), function(data) {
      return openCache().then(function(cache) {
        var location = getLocation(entry.filename);
        var response = new Response(data, { headers: {
          // 由于zip文件没有说明文件的性质，
          'Content-Type': getContentType(entry.filename)
        } });

        console.log('-> Caching', location,'(size:', entry.uncompressedSize, 'bytes)');
        // 如果entry文件是index，那么则将其加入根域缓存
        if (entry.filename === 'index.html') {
          // 由于响应是一次性对象，因此.put()方法克隆响应主体中的数据以便再次使用
          cache.put(getLocation(), response.clone());
        }

        return cache.put(location, response);
      }).then(fulfill, reject);
    });
  });
}
// 返回入口location
function getLocation(filename) {
  return location.href.replace(/worker\.js$/, filename || '');
}

var contentTypesByExtension = {
  'css': 'text/css',
  'js': 'application/javascript',
  'png': 'image/png',
  'jpg': 'image/jpeg',
  'jpeg': 'image/jpeg',
  'html': 'text/html',
  'htm': 'text/html'
};
// 根据文件扩展名返回响应类型
function getContentType(filename) {
  var tokens = filename.split('.');
  var extension = tokens[tokens.length - 1];
  return contentTypesByExtension[extension] || 'text/plain';
}
// 由于开启缓存是开销昂贵的操作，
// 因此我们只打开缓存一次，将返回的promise对象缓存起来
var cachePromise;
function openCache() {
  if (!cachePromise) { cachePromise = caches.open('cache-from-zip'); }
  return cachePromise;
}
```



