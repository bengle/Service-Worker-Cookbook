# Render Store

本章演示来自NGA的一项提案，对插值模版的缓存，避免再次请求时耗费获取模型和渲染的时间。

# 难度等级

中级

# 应用场景

作为一个web应用程序开发者，我希望以最小的消耗时间加载已经访问过的资源。

# 解决方案

使用离线缓存在页面模版完全呈现后存储模版，再下次请求时使用此副本。

# 方案类型

性能优化

# 代码示例

index.html

```html
<!doctype html>
<html>
  <head>
    <meta charset="utf-8" />
    <title>Render Store - ServiceWorker Cookbook</title>
    <meta name="viewport" content="width=device-width, initial-scale=1">
    <script defer src="index.js"></script>
  </head>
  <body>
    <h1>Pokemon Character Sheets</h1>
    <p>Choose a character once, see the times and reload. Check the difference in times.</p>
    <p><ul id="pokemon-list"></ul></p>
  </body>
</html>
```

index.js

```js
// 现代浏览器会阻止混合内容，即，如果页面是从https源提供的，那么它将阻止来自其他非https源的内容。
// 我们使用此服务通过安全的来源跟pokemon API打通
var PROXY = 'https://crossorigin.me/';

// 定义pokemon查询接口
var POKEDEX = PROXY + 'http://pokeapi.co/api/v1/pokedex/1/';

// 一旦service worker激活就加载pokemon列表
if ('serviceWorker' in navigator) {
  if (navigator.serviceWorker.controller) {
    loadPokemonList();
  } else {
    navigator.serviceWorker.register('service-worker.js');
    navigator.serviceWorker.ready.then(function() {
      loadPokemonList();
    });
  }
}

// 加载pokemon列表
function loadPokemonList() {
  fetch(POKEDEX)
    .then(function(response) {
      return response.json();
    })
    .then(function(info) {
      fillPokemonList(info.pokemon);

      // Specifically for the cookbook site :(
      if (window.parent !== window) {
        window.parent.document.body
          .dispatchEvent(new CustomEvent('iframeresize'));
      }
    });
}

// 创建pokemon链接列表，这些链接将被service worker拦截
function fillPokemonList(pokemonList) {
  var listElement = document.getElementById('pokemon-list');
  var buffer = pokemonList.map(function(pokemon) {
    var uriTokens = pokemon.resource_uri.split('/');
    var id = uriTokens[uriTokens.length - 2];
    return '<li><a href="pokemon.html?id=' + id + '">' + pokemon.name +
           '</a></li>';
  });
  listElement.innerHTML = buffer.join('\n');
}
```

pokemon.js

```js
var PROXY = 'https://crossorigin.me/';

var startTime = performance.now();
var interpolationTime = 0;
var fetchingModelTime = 0;

var isCached = document.documentElement.dataset.cached;
/*
  思路是从URL中query参数解析pokemon标志用于查询该条数据，根据pokemon模版填充数据
  填充后将文档标记为缓存，再发送给service worker存储到缓存
  缓存标志我们简单使用data-*的属性表示
*/
if (isCached) {
  // 如果已经缓存记录时间
  logTime();
} else {
  // 如果没有缓存，请求数据填充模版，记录时间并缓存
  var pokemonId = getPokemonId();
  getPokemon(pokemonId).then(fillCharSheet).then(logTime).then(cache);
}

function getPokemonId() {
  return window.location.search.split('=')[1];
}

function getPokemon(id) {
  var fetchingModelStart = performance.now();
  var url = PROXY + 'http://pokeapi.co/api/v1/pokemon/' + id + '/?_b=' + Date.now();
  return fetch(url).then(function(response) {
    fetchingModelTime = performance.now() - fetchingModelStart;
    return response.json();
  });
}
// 用pokemon数据填充模版生成内容
function fillCharSheet(pokemon) {
  var element = document.querySelector('body');
  element.innerHTML = interpolateTemplate(element.innerHTML, pokemon);
  document.querySelector('img').onload = function() {
    if (window.parent === window) {
      return;
    }
    window.parent.document.body.dispatchEvent(new CustomEvent('iframeresize'));
  };
}
// 记录渲染模版，请求数据以及全部资源加载的时间
function logTime() {
  var loadingTimeLabel = document.querySelector('#loading-time-label');
  var interpolationTimeLabel =
    document.querySelector('#interpolation-time-label');
  var fetchingModelTimeLabel = document.querySelector('#fetching-time-label');
  loadingTimeLabel.textContent = (performance.now() - startTime) + ' ms';
  interpolationTimeLabel.textContent = interpolationTime + ' ms';
  fetchingModelTimeLabel.textContent = fetchingModelTime + ' ms';
}
// 标记要缓存的文档，然后获取HTML内容发送PUT请求到./render-store/的URL，service worker会拦截请求添加内容到缓存
// 如果想知道发送缓存内容的URL，可以从请求的referrer属性查看
function cache() {
  document.documentElement.dataset.cached = true;
  var data = document.documentElement.outerHTML;
  fetch('./render-store/', { method: 'PUT', body: data }).then(function() {
    console.log('Page cached');
  });
}
// 填充模版，在模版中查找{{key}}用pokemon[key]的值替换它们
function interpolateTemplate(template, pokemon) {
  var interpolationStart = performance.now();
  var result = template.replace(/{{(\w+)}}/g, function(match, field) {
    return pokemon[field];
  });
  interpolationTime = performance.now() - interpolationStart;
  return result;
}
```

service-worker.js

```js
self.oninstall = function(event) {
  event.waitUntil(self.skipWaiting());
};

self.onactivate = function(event) {
  event.waitUntil(self.clients.claim());
};

self.onfetch = function(event) {
  if (event.request.method === 'GET') {
    event.respondWith(getFromRenderStoreOrNetwork(event.request));
  } else {
    event.respondWith(cacheInRenderStore(event.request).then(function() {
      return new Response({ status: 202 });
    }));
  }
};

function getFromRenderStoreOrNetwork(request) {
  return self.caches.open('render-store').then(function(cache) {
    return cache.match(request).then(function(match) {
      return match || fetch(request);
    });
  });
}

function cacheInRenderStore(request) {
  return request.text().then(function(contents) {
    var headers = { 'Content-Type': 'text/html' };
    var response = new Response(contents, { headers: headers });
    return self.caches.open('render-store').then(function(cache) {
      return cache.put(request.referrer, response);
    });
  });
}
```



