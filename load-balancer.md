# Load Balancer

本案例演示了用service worker控制网络逻辑，根据服务器可用性状态动态选择最佳的内容提供给应用程序。

# 难度等级

中级

# 应用场景

作为服务提供者，我希望在所选资源的可用性方面提供最佳来源。

# 解决方案

使用service worker拦截对资源的请求，根据其可用性选择适当的资源内容提供给应用程序。

# 方案类型

离线扩展

# 代码示例

index.html

```html
<!doctype html>
<html>
<head>
  <meta charset="utf-8">
  <title>Load balancer - ServiceWorker Cookbook</title>
  <meta name="viewport" content="width=device-width, initial-scale=1">
</head>
<body>
  <h2>Resources</h2>
  <p>
    Choose from one of the images to be loaded:
  </p>
  <p>
    Notice the white label in the image which is telling you the server it comes from.
  </p>
  <p>
    <select id="image-selector" disabled>
      <option value="">Select and image</option>
      <option value="imgs/a.jpeg">Collaboration</option>
      <option value="imgs/b.jpeg">Loneliness</option>
      <option value="imgs/c.jpeg">Bread</option>
    </select>
  </p>
  <p><img src="" alt="" /></p>
  <h2>Configuration</h2>
  <p>Configure the load of the content providers.</p>
  <p>Servers set to: <span id="loads-label"></span></p>
  <form id="load-configuration">
    <p><label>Server 1: <input type="number" id="load-1" min="0" max="100" value="50" disabled /> %</label></p>
    <p><label>Server 2: <input type="number" id="load-2" min="0" max="100" value="75" disabled /> %</label></p>
    <p><label>Server 3: <input type="number" id="load-3" min="0" max="100" value="25" disabled /> %</label></p>
    <input type="submit" value="Configure" />
  </form>
  <script src="./index.js"></script>
</body>
</html>
```

index.js

```js
var $ = document.querySelector.bind(document); // eslint-disable-line id-length

var serverLoadInputs = [
  $('#load-1'),
  $('#load-2'),
  $('#load-3')
];
// 注册service worker，激活image选项
navigator.serviceWorker.register('service-worker.js');
navigator.serviceWorker.ready.then(enableUI);

function enableUI() {
  getServerLoads().then(function(loads) {
    serverLoadInputs.forEach(function(input, index) {
      input.value = loads[index];
      input.disabled = false;
    });
    $('#image-selector').disabled = false;
  });
}

function getServerLoads() {
  return fetch(addSession('./server-loads/')).then(function(response) {
    return response.json();
  });
}
// 点击load-configuration按钮时发送负载值到后端模拟服务负载
$('#load-configuration').onsubmit = function(event) {
  event.preventDefault();
  // 从input获取fake级别
   var loads = serverLoadInputs.map(function(input) {
    return parseInt(input.value, 10);
  });
  // 发送请求以配置序列化正文的负载级别并正确设置请求内容的header
  fetch(addSession('./server-loads'), {
    method: 'PUT',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(loads)
  }).then(function(response) {
    return response.json();
  }).then(function(result) {
    $('#loads-label').textContent = result;
  });
};

$('#image-selector').onchange = function() {
  var imgUrl = $('select').value;
  if (imgUrl) {
     // 添加时间戳参数_b避免HTTP缓存
     $('img').src = addSession(imgUrl) + '&_b=' + Date.now();
     $('img').onload = function() {
      if (window.parent !== window) {
        window.parent
          .document.body.dispatchEvent(new CustomEvent('iframeresize'));
      }
    };
  }
};
// URL中添加session字段
function addSession(url) {
  return url + '?session=' + getSession();
}
// 用基于localstorage存储的随机字符串简单模拟一个session管理器
function getSession() {
  var session = localStorage.getItem('session');
  if (!session) {
    session = '' + Date.now() + '-' + Math.random();
    localStorage.setItem('session', session);
  }
  return session;
}
```

server.js

```js
var bodyParser = require('body-parser');
// 简单用hash做一个缓存
var sessions = {};
// 模拟资源服务器资源负载api
module.exports = function(app, route) {
  app.use(bodyParser.json());
  // 定义服务器负责
  app.put(route + 'server-loads', function(req, res) {
    var loads = req.body;
    sessions[req.query.session] = loads;
    res.status(201).json(loads);
  });
  // 查询服务器负载api
  app.get(route + 'server-loads', function(req, res) {
    var loads = sessions[req.query.session] || [50, 75, 25];
    res.json(loads);
  });
};
```

service-worker.js

```js
// 通过oninstall和onactive事件强制service worker接管客户端
self.oninstall = function(event) {
  event.waitUntil(self.skipWaiting());
};

self.onactivate = function(event) {
  event.waitUntil(self.clients.claim());
};
// 拦截fetch请求，区分这是否是资源获取
// 如果是，那么调用服务器选择算法；如果不是，则继续走网络。
self.onfetch = function(event) {
  var request = event.request;
  if (isResource(request)) {
    event.respondWith(fetchFromBestServer(request));
  } else {
    event.respondWith(fetch(request));
  }
};
// 如果是针对imgs内容的GET请求，那么则是资源请求
function isResource(request) {
  return request.url.match(/\/imgs\/.*$/) && request.method === 'GET';
}
// 从最佳服务器获取服务器负载，选择负载最低的服务器，从该服务器请求资源
function fetchFromBestServer(request) {
  var session = request.url.match(/\?session=([^&]*)/)[1];
  return getServerLoads(session)
    .then(selectServer)
    .then(function(serverUrl) {
      // 获取资源路径与serverUrl结合，在所选的服务器中获取资源URL
      var resourcePath = request.url.match(/\/imgs\/[^?]*/)[0];
      var serverRequest = new Request(serverUrl + resourcePath);
      return fetch(serverRequest);
    });
}
// 查询服务后端负载
function getServerLoads(session) {
  return fetch('./server-loads?session=' + session).then(function(response) {
    return response.json();
  });
}
// 获取最小负载服务器URL
// 真实场景中，这里可能返回其他域中的服务器，需要记得正确设置CORS头
function selectServer(serverLoads) {
  // 以最小负载查找服务器索引虽然效率不高但非常清晰
  var min = Math.min.apply(Math, serverLoads);
  var serverIndex = serverLoads.indexOf(min);

  return './server-' + (serverIndex + 1);
}
```



