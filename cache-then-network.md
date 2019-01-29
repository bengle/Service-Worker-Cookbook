# Cache then Network

本章案例演示了从网络或者缓存响应网络请求的方法。

# 难度等级

初级

# 应用场景

用service worker的一个优势就是可以通过编程控制从缓存返回响应内容以及希望从服务器加载的内容。本示例为您提供如何选择的前端方案。

# 功能和用法

* 配置表单以允许或禁止缓存（和/或）网络
* 点击“Get new data”触发请求

# 应用类型

性能优化

# 代码示例

index.html

```html
<!doctype html>
<html>
<head>
  <meta charset='utf-8'>
  <title>Cache then Network - ServiceWorker Cookbook</title>
  <meta name='viewport' content='width=device-width, initial-scale=1'>
  <style>
    body {
      font-family: monospace;
      font-size: 1rem;
      white-space: pre-wrap;
    }
  </style>
</head>
<body>

  <label>Data: </label><output id='data'></output>
  <label>Cache: </label><output id='cache-status'></output>
  <label>Network: </label><output id='network-status'></output>

  <form>Cache:
    <input id='cache-fail' type='checkbox'>Fail</input>
    Delay: <input id='cache-delay' type='number' value='500'></input>ms
  </form>

  <form>Network:
    <input id='network-fail' type='checkbox'>Fail</input>
    Delay: <input id='network-delay' type='number' value='1250'></input>ms
  </form>

  <button id='getDataButton'>Get new data</button>

  <script src='./index.js'></script>
</body>
</html>

```

index.js

```js
var dataUrl = 'https://api.github.com/events';
var cacheDelayInput = document.getElementById('cache-delay');
var cacheFailInput = document.getElementById('cache-fail');
var networkDelayInput = document.getElementById('network-delay');
var networkFailInput = document.getElementById('network-fail');
var cacheStatus = document.getElementById('cache-status');
var networkStatus = document.getElementById('network-status');
var getDataButton = document.getElementById('getDataButton');
var dataElement = document.getElementById('data');

var cacheName = 'cache-then-network';
var gotNetworkData = false;

var networkFetchStartTime;
var cacheFetchStartTime;

function reset() {
  dataElement.textContent = '';
  cacheStatus.textContent = '';
  networkStatus.textContent = '';
  gotNetworkData = false;
}

function disableUI() {
  getDataButton.disabled = true;
  cacheDelayInput.disabled = true;
  cacheFailInput.disabled = true;
  networkDelayInput.disabled = true;
  networkFailInput.disabled = true;
  reset();
}

function enableUI() {
  getDataButton.disabled = false;
  cacheDelayInput.disabled = false;
  cacheFailInput.disabled = false;
  networkDelayInput.disabled = false;
  networkFailInput.disabled = false;
}

function updatePage(data) {
  dataElement.textContent = 'User ' + data[0].actor.login +
                            ' modified repo ' + data[0].repo.name;
}

function handleFetchCompletion(res) {
  var shouldNetworkError = networkFailInput.checked;
  if (shouldNetworkError) {
    throw new Error('Network error');
  }
  
  var resClone = res.clone();
  caches.open(cacheName).then(function(cache) {
    cache.put(dataUrl, resClone);
  });
  
  res.json().then(function(data) {
    updatePage(data);
    gotNetworkData = true;
  });
}

function handleCacheFetchCompletion(res) {
  var shouldCacheError = cacheFailInput.checked;
  if (shouldCacheError || !res) {
    throw Error('Cache miss');
  }
  res.json().then(function(data) {
    if (!gotNetworkData) {
      updatePage(data);
    }
  });
}

getDataButton.addEventListener('click', function handleClick() {
  disableUI();
  networkStatus.textContent = 'Fetching...';
  networkFetchStartTime = Date.now();
  
  var cacheBuster = Date.now();
  var networkFetch = fetch(dataUrl + '?cacheBuster=' + cacheBuster, {
    mode: 'cors',
    cache: 'no-cache',
  }).then(function(res) {
    var networkDelay = networkDelayInput.value || 0;
    return new Promise(function(resolve, reject) {
      setTimeout(function() {
        try {
          handleFetchCompletion(res);
          resolve();
        } catch (err) {
          reject(err);
        }
      }, networkDelay);
    });
  }).then(function() {
    var now = Date.now();
    var elapsed = now - networkFetchStartTime;
    networkStatus.textContent = 'Success after ' + elapsed + 'ms';
  }).catch(function(err) {
    var now = Date.now();
    var elapsed = now - networkFetchStartTime;
    networkStatus.textContent = err + ' after ' + elapsed + 'ms';
  });
  cacheStatus.textContent = 'Fetching...';
  cacheFetchStartTime = Date.now();
  var cacheFetch = caches.open(cacheName).then(function(cache) {
    return cache.match(dataUrl).then(function(res) {
      var cacheDelay = cacheDelayInput.value || 0;
      return new Promise(function(resolve, reject) {
        setTimeout(function() {
          try {
            handleCacheFetchCompletion(res);
            resolve();
          } catch (err) {
            reject(err);
          }
        }, cacheDelay);
      });
    }).then(function() {
      var now = Date.now();
      var elapsed = now - cacheFetchStartTime;
      cacheStatus.textContent = 'Success after ' + elapsed + 'ms';
    }).catch(function(err) {
      var now = Date.now();
      var elapsed = now - cacheFetchStartTime;
      cacheStatus.textContent = err + ' after ' + elapsed + 'ms';
    });
  });

  Promise.all([networkFetch, cacheFetch]).then(enableUI);
});
```



