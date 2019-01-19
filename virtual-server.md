# Virtual Server

本案例演示如何用service worker模拟一个远程服务器。

# 难度等级

中级

# 应用场景

作为一个应用程序开发者，我希望将UI和业务逻辑完全分离开。

# 解决方案

使用REST API我们可以将客户端与业务逻辑分离，业务逻辑实际上是放在远程服务器上独立运行。使用service worker我们同样可以这么做，只需要将业务逻辑移动到响应fetch事件的service worker上来。

我们不需要自己根据逻辑来区分路由和请求方法，而使用\[ServiceWorkerWare\]\(https://github.com/gaia-components/serviceworkerware\)或者\[sw-toolbox\]\(https://github.com/GoogleChrome/sw-toolbox\#defining-routes\)这两个库的路有功能声明worker工作逻辑。

客户端代码几乎和API Analytics一章相同，然而远程Express服务器已经完全由service worker取代。

# 方案类型

离线扩展

# 代码示例

index.html

```html
<!doctype html>
<html>
<head>
  <meta charset="utf-8" />
  <title>Virtual server - ServiceWorker Cookbook</title>
  <meta name="viewport" content="width=device-width, initial-scale=1">
</head>
<body>
  <h1>How to</h1>
  <p>Try to add and delete some quotations. <strong>The session is not intended to survive refreshing the page</strong> but you could see it pretending to survive. This means the worker holding the state has not been killed yet.</p>
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
  // 匿名quote留空表示
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

service-worker.js

```js
importScripts('./lib/ServiceWorkerWare.js');
// 默认quotation列表
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
  // 添加id和sticky标志使默认quotation不可删除
  quotation.id = index + 1;
  quotation.isSticky = true;

  return quotation;
});
// 设置路由root路径，例如：如果service worker的url是http://example.com/path/to/sw.js
// 那么root路径就是http://example.com/path/to/
var root = (function() {
  var tokens = (self.location + '').split('/');
  tokens[tokens.length - 1] = '';
  return tokens.join('/');
})();
// 使用Mozilla出品的ServiceWorkerWare可以快速设置虚拟服务的路有
var worker = new ServiceWorkerWare();
// 返回所有的quotation
worker.get(root + 'api/quotations', function(req, res) {
  return new Response(JSON.stringify(quotations.filter(function(item) {
    return item !== null;
  })));
});
// 根据id删除一个quote。
// id是quotation数组的位置
worker.delete(root + 'api/quotations/:id', function(req, res) {
  var id = parseInt(req.parameters.id, 10) - 1;
  if (!quotations[id].isSticky) {
    quotations[id] = null;
  }
  return new Response({ status: 204 });
});
// 给数组添加一个新的quote
worker.post(root + 'api/quotations', function(req, res) {
  return req.json().then(function(quote) {
    quote.id = quotations.length + 1;
    quotations.push(quote);
    return new Response(JSON.stringify(quote), { status: 201 });
  });
});
// 初始化service worker
worker.init();
```



