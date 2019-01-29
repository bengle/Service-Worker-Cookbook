# Request Deferrer

本案例演示如何在类似发件箱的缓冲区中允许用户离线时可以将请求排入队列，当重新获得连接之后可以继续执行操作。

# 难度等级

高级

# 应用场景

作为一名现代框架开发人员，我想提供一种在离线时处理请求的特殊方式。

# 解决方案

使用service worker拦截请求，当离线时将请求记录在队列中保存其顺序并用假的响应内容response；当在线时，将队列总的请求重新发送与服务器同步。

此高级技术旨在与REST API集成，客户端可以处理异步\(POST\)操作\(HTTP响应状态码202\)。

这个解决方案只是证明方案可行性，没有包含错误处理，而这才是该方案在现实应用中真正的挑战，部分内联文档说明了部分问题。

# 方案类型

离线扩展

# 代码示例

index.html

```html
<!doctype html>
<html>
<head>
  <meta charset="utf-8" />
  <title>Request sync - ServiceWorker Cookbook</title>
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <style>
    .queue {
      background-color: #68CCED;
      padding: 10px;
      border-radius: 10px;
      font-weight: bold;
    }
  </style>
</head>
<body>
  <h1>How to</h2>
  <p>Try to add and delete some quotations.</p>
  <p>Then go offline by disconnecting from Internet</p>
  <p>And try to continue adding some quotes.</p>
  <p>You can see they don't have a delete button because they are in queue</p>
  <p>Now reconnect to the Internet and you'll see how they automatically synchronize.</p>
  <p><strong>Show console logs for more information.</strong></a>
  <p><form id="add-form">
    <input type="text" id="new-quote" placeholder="Add a quote here"/>
    <input type="text" id="quote-author" placeholder="And its author"/><input type="submit" value="Add" />
  </form></p>
  <h2>Quotes</h2>
  <table id="quotations">
  </table>
<script src="./index.js"></script>
</body>
</html>
```

index.js

```js
var ENDPOINT = 'api/quotations';
// 注册service worker，显示quotation列表
if (navigator.serviceWorker.controller) {
  loadQuotations();
} else {
  navigator.serviceWorker.oncontrollerchange = function() {
    this.controller.onstatechange = function() {
      if (this.state === 'activated') {
        loadQuotations();
      }
    };
  };
  navigator.serviceWorker.register('service-worker.js');
}
// 强制清空队列请求
window.addEventListener('online', function() {
  loadQuotations();
});
// 点击add-form按钮，添加条新的quote、author和post
document.getElementById('add-form').onsubmit = function(event) {
  event.preventDefault();

  var newQuote = document.getElementById('new-quote').value.trim();
  if (!newQuote) { return; }
  // 如果quote-author为空则用Anonymous代替
  var quoteAuthor =
    document.getElementById('quote-author').value.trim() || 'Anonymous';
  var quote = { text: newQuote, author: quoteAuthor };
  var headers = { 'content-type': 'application/json' };
  var body = JSON.stringify(quote);
  fetch(addSession(ENDPOINT), { method: 'POST', headers: headers, body: body })
    .then(function(response) {
      // 已接受但未添加，这点区别很重要
      // 我们使用它来支持离线情况下对quotations的延迟加载
      if (response.status === 202) {
        return quote;
      }
      return response.json();
    })
    .then(function(addedQuote) {
      document.getElementById('quotations').appendChild(getRowFor(addedQuote));
      if (window.parent !== window) {
        window.parent.document.body
          .dispatchEvent(new CustomEvent('iframeresize'));
      }
    });
};
// 简单的GET请求(通常是fetch)就足够完成对quotes的查询了
function loadQuotations() {
  fetch(addSession(ENDPOINT))
    .then(function(response) {
      return response.json();
    })
    .then(showQuotations);
}
// 用quotation填充table
function showQuotations(collection) {
  var table = document.getElementById('quotations');
  table.innerHTML = '';
  collection.forEach(function(quote) {
    table.appendChild(getRowFor(quote));
  });
  if (window.parent !== window) {
    window.parent.document.body.dispatchEvent(new CustomEvent('iframeresize'));
  }
}
// 用quote数据生成表格一行内容
function getRowFor(quote) {
  var tr = document.createElement('TR');
  var id = quote.id;
  if (id) {
    tr.id = id;
  }

  tr.appendChild(getCell(quote.text));
  tr.appendChild(getCell('by ' + quote.author));

  if (!id) {
    tr.appendChild(getInQueueLabel());
  } else {
    tr.appendChild(quote.isSticky ? getCell('') : getDeleteButton(id));
  }
  return tr;
}
function getCell(text) {
  var td = document.createElement('TD');
  td.textContent = text;
  return td;
}
function getInQueueLabel() {
  var td = getCell('(in queue...)');
  td.classList.add('queue');
  return td;
}
function getDeleteButton(id) {
  var td = document.createElement('TD');
  var button = document.createElement('BUTTON');
  button.textContent = 'Delete';
  button.onclick = function() {
    deleteQuote(id).then(function() {
      var tr = document.getElementById(id);
      tr.parentNode.removeChild(tr);
    });
  };

  td.appendChild(button);
  return td;
}
function deleteQuote(id) {
  return fetch(addSession(ENDPOINT + '/' + id), { method: 'DELETE' });
}
function addSession(url) {
  return url + '?session=' + getSession();
}
// 在localstorage中用session字段做一个简单的session管理器
// session使用日期和随机数的字符串表示
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
var sessions = {};
var defaultQuotations = makeDefaults([
  {
    text: 'Humanity is smart. Sometime in the technology world we think we ' +
    'are smarter, but we are not smarter than you.',
    author: 'Mitchell Baker'
  },
  {
    text: 'A computer would deserve to be called intelligent if it could ' +
    'deceive a human into believing that it was human.',
    author: 'Alan Turing'
  },
  {
    text: 'If you optimize everything, you will always be unhappy.',
    author: 'Donald Knuth'
  },
  {
    text: 'If you don\'t fail at least 90 percent of the time, you\'re not ' +
    'aiming high enough',
    author: 'Alan Kay'
  },
  {
    text: 'Colorless green ideas sleep furiously.',
    author: 'Noam Chomsky'
  }
]);
module.exports = function(app, route) {
  app.use(bodyParser.json());
  app.get(route + 'api/quotations', function(req, res) {
    var quotations = getQuotationsForSession(req.query.session);
    res.json(quotations.filter(function(item) {
      return item !== null;
    }));
  });
  app.delete(route + 'api/quotations/:id', function(req, res) {
    var quotations = getQuotationsForSession(req.query.session);
    var id = parseInt(req.params.id, 10) - 1;
    if (!quotations[id].isSticky) {
      quotations[id] = null;
    }
    res.sendStatus(204);
  });
  app.post(route + 'api/quotations', function(req, res) {
    var quotations = getQuotationsForSession(req.query.session);
    var quote = req.body;
    quote.id = quotations.length + 1;
    quotations.push(quote);
    res.status(201).json(quote);
  });
};
function makeDefaults(quotationList) {
  for (var index = 0, max = quotationList.length, quote; index < max; index++) {
    quote = quotationList[index];
    quote.id = index + 1;
    quote.isSticky = true;
  }
  return quotationList;
}
function getQuotationsForSession(session) {
  if (!(session in sessions)) {
    sessions[session] = JSON.parse(JSON.stringify(defaultQuotations));
  }
  return sessions[session];
}
```

service-worker.js

```js
/* eslint-env es6 */
/* eslint no-unused-vars: 0 */
/* global importScripts, ServiceWorkerWare, localforage */
importScripts('./lib/ServiceWorkerWare.js');
importScripts('./lib/localforage.js');
// 定义路由的根路径
// 如果service worker的URL是http://example.com/path/to/sw.js，那么root路径就是http://example.com/path/to/
var root = (function() {
  var tokens = (self.location + '').split('/');
  tokens[tokens.length - 1] = '';
  return tokens.join('/');
})();
// 使用Mozilla的ServiceWorkerWare，我们可以快速为虚拟服务器设置路由
// 你可以从下面的案例代码中方便的查看虚拟服务器的路由配置
// 我们的做法是，首先检查是否在线，
// 如果不在线将请求排入队列并提供模拟的响应
// 如果在线则将队列中的请求通过网络拿到真实响应
var worker = new ServiceWorkerWare();
function tryOrFallback(fakeResponse) {
  return function(req, res) {
    if (!navigator.onLine) {
      console.log('No network availability, enqueuing');
      return enqueue(req).then(function() {
        return fakeResponse.clone();
      });
    }
    console.log('Network available! Flushing queue.');
    return flushQueue().then(function() {
      return fetch(req);
    });
  };
}
worker.get(root + 'api/quotations?*', tryOrFallback(new Response(
  JSON.stringify([{
    text: 'You are offline and I know it well.',
    author: 'The Service Worker Cookbook',
    id: 1,
    isSticky: true
  }]),
  { headers: { 'Content-Type': 'application/json' } }
)));
worker.delete(root + 'api/quotations/:id?*', tryOrFallback(new Response({
  status: 204
})));
worker.post(root + 'api/quotations?*', tryOrFallback(new Response(null, {
  status: 202
})));
worker.init();
// 使用Mozilla的localforage db包装器，我们可以快速设置多功能键值数据库
// 我们用它来存储延迟请求的队列
// enqueue将请求添加到队列，由于IndexedDB限制，我们无法保存请求和响应的对象
function enqueue(request) {
  return serialize(request).then(function(serialized) {
    localforage.getItem('queue').then(function(queue) {
      /* eslint no-param-reassign: 0 */
      queue = queue || [];
      queue.push(serialized);
      return localforage.setItem('queue', queue).then(function() {
        console.log(serialized.method, serialized.url, 'enqueued!');
      });
    });
  });
}
function flushQueue() {
  return localforage.getItem('queue').then(function(queue) {
    /* eslint no-param-reassign: 0 */
    queue = queue || [];
    if (!queue.length) {
      return Promise.resolve();
    }
    console.log('Sending ', queue.length, ' requests...');
    return sendInOrder(queue).then(function() {
      return localforage.setItem('queue', []);
    });
  });
}
function sendInOrder(requests) {
  var sending = requests.reduce(function(prevPromise, serialized) {
    console.log('Sending', serialized.method, serialized.url);
    return prevPromise.then(function() {
      return deserialize(serialized).then(function(request) {
        return fetch(request);
      });
    });
  }, Promise.resolve());
  return sending;
}
function serialize(request) {
  var headers = {};
  for (var entry of request.headers.entries()) {
    headers[entry[0]] = entry[1];
  }
  var serialized = {
    url: request.url,
    headers: headers,
    method: request.method,
    mode: request.mode,
    credentials: request.credentials,
    cache: request.cache,
    redirect: request.redirect,
    referrer: request.referrer
  };
  if (request.method !== 'GET' && request.method !== 'HEAD') {
    return request.clone().text().then(function(body) {
      serialized.body = body;
      return Promise.resolve(serialized);
    });
  }
  return Promise.resolve(serialized);
}
function deserialize(data) {
  return Promise.resolve(new Request(data.url, data));
}
```



