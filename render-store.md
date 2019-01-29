# Render Store

本章演示来来自NGA的一项提案，对插值模版的缓存，避免再次请求时耗费获取模型和渲染的时间。

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
// Modern browsers prevent mixed content. I.e., if the page is served from
// a safe (https) origin, they will block the content from other (non https)
// origins. We use this service to tunnel Pokemon API responses through a
// secure origin.
var PROXY = 'https://crossorigin.me/';

// We need the pokedex entry point to retrieve the complete list
// of Pokemon.
var POKEDEX = PROXY + 'http://pokeapi.co/api/v1/pokedex/1/';

// Once the Service Worker is activated, load the Pokemon list.
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

// Fetch the Pokemon list from pokedex and create the list
// of links.
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

// Creates the links for the pokemon list. These links will be intercepted
// by the service worker.
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

if (isCached) {
  logTime();
} else {
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
function logTime() {
  var loadingTimeLabel = document.querySelector('#loading-time-label');
  var interpolationTimeLabel =
    document.querySelector('#interpolation-time-label');
  var fetchingModelTimeLabel = document.querySelector('#fetching-time-label');
  loadingTimeLabel.textContent = (performance.now() - startTime) + ' ms';
  interpolationTimeLabel.textContent = interpolationTime + ' ms';
  fetchingModelTimeLabel.textContent = fetchingModelTime + ' ms';
}
function cache() {
  document.documentElement.dataset.cached = true;
  var data = document.documentElement.outerHTML;
  fetch('./render-store/', { method: 'PUT', body: data }).then(function() {
    console.log('Page cached');
  });
}
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



