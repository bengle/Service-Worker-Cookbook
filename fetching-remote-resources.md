# Fetching Remote Resources {#fetching-remote-resources}

本章节演示2种标准的加载远程资源的方法，其中一种是用service worker充当代理中间件。

演示的例子中我们使用了3中远程资源：非安全的资源（http），附带allow-origin请求头的安全资源以及不带allow-origin请求头的安全资源。

# DOM元素

DOM元素即将加载的资源

# Fetch

获取每个资源发出的cors或no-cors请求

# 通过service woker代理发送Fetch

响应客户端加载本地资源的Fetch请求./cookbook-proxy/{full URL}，然后将其转换为serivce worker中的真实URL转发给客户端。

# 难度等级

中级

# 方案类别

通用

# 代码示例

index.html

```html

<!doctype html>
<html>
<head>
  <meta charset="utf-8">
  <title>Fetching - ServiceWorker Cookbook</title>
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <style>
    body {
      font-family: monospace;
      font-size: 1rem;
      white-space: pre-wrap;
    }
    section {
      display: block;
    }
    p {
      padding-left: 5em;
      margin-bottom: 0;
      color: green;
    }
    p.error {
      color: red;
    }
    img {
      height: 100px;
    }
  </style>
</head>
<body>
  <!-- ##################### IMG.SRC ####################### -->
  <section id="img-https-acao">
    <strong>Receiving via DOM Elements</strong>

    https, Access-Control-Allow-Origin=*

  </section>
  <section id="img-https">
    https, no Access-Control-Allow-Origin

  </section>
  <section id="img-http">
    http

  </section>
  <!-- ##################### STANDARD FETCH ####################### -->
  <section id="https-acao-cors">
    <strong>Receiving via fetch</strong>

    <strong>request type / source protocol, response header</strong>
    cors / https, Access-Control-Allow-Origin=*
  </section>
  <section id="https-acao-no-cors">
    no-cors / https, Access-Control-Allow-Origin=*
  </section>
  <section id="https-cors">
    cors / https, no Access-Control-Allow-Origin
  </section>
  <section id="https-no-cors">
    no-cors / https, no Access-Control-Allow-Origin
  </section>
  <section id="http-cors">
    cors / http
  </section>
  <section id="http-no-cors">
    no-cors / http
  </section>

  <!-- ##################### PROXY FETCH ####################### -->
  <section id="proxy-https-acao-cors">
    <strong>Receiving via fetch, with a serviceworker proxy</strong>
    This is to check if there is any difference when client is requesting a local resource and service-worker middleware is responding with a remote resource.
    <strong>request type / source protocol, response header</strong>

    cors / https, Access-Control-Allow-Origin=*
  </section>
  <section id="proxy-https-acao-no-cors">
    no-cors / https, Access-Control-Allow-Origin=*
  </section>
  <section id="proxy-https-cors">
    cors / https, no Access-Control-Allow-Origin
  </section>
  <section id="proxy-https-no-cors">
    no-cors / https, no Access-Control-Allow-Origin
  </section>
  <section id="proxy-http-cors">
    cors / http
  </section>
  <section id="proxy-http-no-cors">
    no-cors / http
  </section>
  <script src="./index.js"></script>
</body>
</html>

```

index.js

```js
// 集中不同类型的url地址
// https-acao：设置了header字段Access-Control-Allow-Origin=*的SSL协议请求
// https：没有对header设置的SSL协议请求
// http：没有加密的非安全协议请求
var urls = {
  'https-acao': 'https://mozorg.cdn.mozilla.net/media/img/styleguide/identity/mozilla/wordmark.b9f1818e8d92.png',
  'https': 'https://static.squarespace.com/static/52d66949e4b0a8cec3bcdd46/t/52ebf67fe4b0f4af2a4502d8/1391195777839/1500w/Hello%20Internet.003.png',
  'http': 'http://piotr.zalewa.info/downloads/mozilla.png'
};
// 两种获取远程资源的模式，本案例中都有体现
var fetchModes = ['cors', 'no-cors'];
// 判断service worker是否已经注册，如果没有就将其注册并reload页面保证客户端在service worker的控制下
navigator.serviceWorker.getRegistration().then(function(registration) {
  if (!registration || !navigator.serviceWorker.controller) {
    navigator.serviceWorker.register('./service-worker.js').then(function() {
      console.log('Service worker registered, reloading the page');
      window.location.reload();
    });
  } else {
    console.log('DEBUG: client is under the control of service worker');
    proceed();
  }
});
// 根据urls和fetchModes遍历每一种组合生成一个image元素
function proceed() {
  for (var protocol in urls) {
    if (urls.hasOwnProperty(protocol)) {
      makeImage(protocol, urls[protocol]);
      for (var index = 0; index < fetchModes.length; index++) {
        var fetchMode = fetchModes[index];
        var init = { method: 'GET',
                     mode: fetchMode,
                     cache: 'default' };
        makeRequest(fetchMode, protocol, init)();
      }
    }
  }
}
// 创建一个img元素
function makeImage(protocol, url) {
  var section = 'img-' + protocol;
  var image = document.createElement('img');
  image.src = url;
  document.getElementById(section).appendChild(image);
}
// 创建request
function makeRequest(fetchMode, protocol, init) {
  return function() {
    var section = protocol + '-' + fetchMode;
    var url = urls[protocol];
    // 直接Fetch远程资源
    fetch(url, init).then(function(response) {
      fetchSuccess(response, url, section);
    }).catch(function(error) {
      fetchCatch(error, url, section);
    });
    // 通过service worker中创建的代理Fetch资源，客户端会认为它是一个本地资源
    fetch('./cookbook-proxy/' + url, init).then(function(response) {
      fetchSuccess(response, './cookbook-proxy/' + url, 'proxy-' + section);
    }).catch(function(error) {
      fetchCatch(error, './cookbook-proxy/' + url, 'proxy-' + section);
    });
  };
}
// 打印对应DOM的log信息
function fetchSuccess(response, url, section) {
  if (response.ok) {
    console.log(section, 'SUCCESS: ', url, response);
    log(section, 'SUCCESS');
  } else {
    console.log(section, 'FAIL:', url, response);
    log(section, 'FAIL: response type: ' + response.type +
                 ', response status: ' + response.status, 'error');
  }
}

function fetchCatch(error, url, section) {
  console.log(section, 'ERROR: ', url, error);
  log(section, 'ERROR: ' + error, 'error');
}

function log(section, message, type) {
  var sectionElement = document.getElementById(section);
  var logElement = document.createElement('p');
  if (type) {
    logElement.classList.add(type);
  }
  logElement.textContent = message;
  sectionElement.appendChild(logElement);
}
```

service-worker.js

```js
// 为包含cookbook-proxy字符串的本地url创建代理
self.onfetch = function(event) {
  if (event.request.url.includes('cookbook-proxy')) {
    var init = { method: 'GET',
                 mode: event.request.mode,
                 cache: 'default' };
    var url = event.request.url.split('cookbook-proxy/')[1];
    console.log('DEBUG: proxying', url);
    event.respondWith(fetch(url, init));
  } else {
    event.respondWith(fetch(event.request));
  }
};
```



