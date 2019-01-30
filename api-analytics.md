# API Analytics

通过添加service worker来手机api使用请求并实时上传收集到的数据，而从采集api使用日志而不干扰UI层逻辑。

# 难度等级

中级

# 应用场景

作为一个web应用程序开发者，我希望在不修改客户端和服务端代码的情况下可以收集站点API调用追踪记录。

# 解决方案

添加service worker拦截客户端的所有请求，再将日志信息发送到日志API。

# 方案类型

离线扩展

# 代码示例

index.html

```html
<!doctype html>
<html>
<head>
  <meta charset="utf-8">
  <title>API analytics - ServiceWorker Cookbook</title>
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <style>
    body {
      font-family: monospace;
      font-size: 1rem;
      white-space: pre-wrap;
    }
  </style>
</head>
<body>
  Try to add and delete some quotations.
  <form id="add-form">
    <input type="text" id="new-quote" placeholder="Add a quote here"/>
    <input type="text" id="quote-author" placeholder="And its author"/><input type="submit" value="Add" />
  </form>
  <table id="quotations">
  </table>
  Go to <a href="./report/" target="_blank">report</a>.
<script src="./index.js"></script>
</body>
</html>
```

index.js

```js
var ENDPOINT = 'api/quotations';
// 注册service worker并显示quotation列表
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
// 当点击add按钮时，在末尾添加一条新的quote、author和post数据
document.getElementById('add-form').onsubmit = function(event) {
  event.preventDefault();
  var newQuote = document.getElementById('new-quote').value.trim();

  if (!newQuote) { return; }
  // 如果quote-author为空则用Anonymous代替
  var quoteAuthor = document.getElementById('quote-author').value.trim() || 'Anonymous';
  var quote = { text: newQuote, author: quoteAuthor };
  var headers = { 'content-type': 'application/json' };
  // 发送请求获取quotations列表数据
  fetch(ENDPOINT, {
    method: 'POST',
    body: JSON.stringify(quote),
    headers: headers
  })
  .then(function(response) {
    return response.json();
  })
  .then(function(addedQuote) {
    document.getElementById('quotations').appendChild(getRowFor(addedQuote));
    resizeIframe();
  });
};
// 用一个简单的get请求（默认为fetch）查询quote列表数据
function loadQuotations() {
  fetch(ENDPOINT)
    .then(function(response) {
      return response.json();
    })
    .then(showQuotations);
}
// 根据quotations填充列表
function showQuotations(collection) {
  var table = document.getElementById('quotations');
  table.innerHTML = '';
  for (var index = 0, max = collection.length, quote; index < max; index++) {
    quote = collection[index];
    table.appendChild(getRowFor(quote));
  }
  resizeIframe();
}
// 根据quote创建一行
function getRowFor(quote) {
  var tr = document.createElement('TR');
  var id = quote.id;
  // 为了删除方便将tr的id设置为quote的唯一id
  tr.id = id;
  tr.appendChild(getCell(quote.text));
  tr.appendChild(getCell('by ' + quote.author));
  tr.appendChild(quote.isSticky ? getCell('') : getDeleteButton(id));
  return tr;
}
// 根据text创建数据单元
function getCell(text) {
  var td = document.createElement('TD');
  td.textContent = text;
  return td;
}
// 创建删除按钮
function getDeleteButton(id) {
  var td = document.createElement('TD');
  var button = document.createElement('BUTTON');
  button.textContent = 'Delete';
  // 点击删除按钮则请求删除api，从列表中删除对应quotation
  button.onclick = function() {
    deleteQuote(id).then(function() {
      var tr = document.getElementById(id);
      tr.parentNode.removeChild(tr);
    });
  };

  td.appendChild(button);
  return td;
}
// 删除请求
function deleteQuote(id) {
  return fetch(ENDPOINT + '/' + id, { method: 'DELETE' });
}

function resizeIframe() {
  if (window.parent !== window) {
    window.parent.document.body.dispatchEvent(new CustomEvent('iframeresize'));
  }
}
```

server.js

```js
// 创建一个简单的server提供两个API，一个操作quotation另一个获取日志
var path = require('path');
var bodyParser = require('body-parser');
var swig = require('swig');
// 存放日志的数组
var requestsLog = [];
// 默认的quotations
var quotations = [
  {
    text: 'Humanity is smart. Sometime in the technology world we think' +
    'we are smarter, but we are not smarter than you.',
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
    text: 'If you don\'t fail at least 90 percent of the time' +
    'you\'re not aiming high enough',
    author: 'Alan Kay'
  },
  {
    text: 'Colorless green ideas sleep furiously.',
    author: 'Noam Chomsky'
  }
].map(function(quotation, index) {
  // 给默认quotations添加id和sticky标志，使其不可删除
  quotation.id = index + 1;
  quotation.isSticky = true;

  return quotation;
});
// 针对quotation和log的REST API
// service worker允许我们记录每个请求而无需关心API实现
module.exports = function(app, route) {
  var report = swig.compileFile(path.join(__dirname, './report.html'));
  // express body parser
  app.use(bodyParser.json());
  // 显示report
  app.get(route + 'report', function(req, res) {
    var statistics = summarizeLogs();
    var buffer = report({ statistics: statistics });
    res.send(buffer);
  });
  // 根据url和operation添加一条新的日志
  app.post(route + 'report/logs', function(req, res) {
    var logEntry = logRequest(req.body);
    res.status(201).json(logEntry);
  });
  // 返回所有quotations
  app.get(route + 'api/quotations', function(req, res) {
    res.json(quotations.filter(function(item) {
      return item !== null;
    }));
  });
  // 根据id删除一条quote
  app.delete(route + 'api/quotations/:id', function(req, res) {
    var id = parseInt(req.params.id, 10) - 1;
    if (!quotations[id].isSticky) {
      quotations[id] = null;
    }
    res.sendStatus(204);
  });
  // 给数组添加一条quote
  app.post(route + 'api/quotations', function(req, res) {
    var quote = req.body;
    quote.id = quotations.length + 1;
    quotations.push(quote);
    res.status(201).json(quote);
  });
};

function logRequest(request) {
  request.id = requestsLog.length + 1;
  requestsLog.push(request);
  return request;
}
// 按照URL聚合日志，统计GET、DELETE和POST操作
function summarizeLogs() {
  // 聚合操作
  var aggregations = requestsLog.reduce(function(partialSummary, entry) {
    if (!(entry.url in partialSummary)) {
      partialSummary[entry.url] = {
        url: entry.url,
        GET: 0,
        POST: 0,
        DELETE: 0,
      };
    }
    partialSummary[entry.url][entry.method]++;
    return partialSummary;
  }, {});
  // 返回一个结果数组
  return Object.keys(aggregations).map(function(url) {
    return aggregations[url];
  });
}
```

service-worker.js

```js
// 实际应用中记录日志的接口可能是其他域的
var LOG_ENDPOINT = 'report/logs';
// 使用oninstall和onactive强制service worker控制客户端
self.oninstall = function(event) {
  event.waitUntil(self.skipWaiting());
};

self.onactivate = function(event) {
  event.waitUntil(self.clients.claim());
};

self.onfetch = function(event) {
  event.respondWith(
    // 发送日志记录请求，然后请求真正的API
    log(event.request)
    .then(fetch)
  );
};
// 将每一次请求的基本信息发送到服务器用于记录操作历史
function log(request) {
  var returnRequest = function() {
    return request;
  };

  var data = {
    method: request.method,
    url: request.url
  };

  return fetch(LOG_ENDPOINT, {
    method: 'POST',
    body: JSON.stringify(data),
    headers: { 'content-type': 'application/json' }
  })
  .then(returnRequest, returnRequest);
}
```



