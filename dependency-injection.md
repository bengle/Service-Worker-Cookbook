# Dependency Injection

本案例中演示了如何用service worker做依赖注入的事情，从而避免在高级组件中对依赖的硬编码。

# 难度等级

高级

# 应用场景

作为一个框架开发者，我想在生产和测试环境配置两个不同的注入器为组件提供正确的模型，而无需更改客户端代码。

# 解决方案

service  worker可以作为注入器，提供两个不同的注入器，一个用于生产，另一个用于测试。由框架代码决定注册使用哪一个，然后根据注入器内的映射为统一资源提供不同的内容。

首先在bootstrap.js了解框架如何检测应该使用哪个注入器。default-mapping.js文件包含有关抽象资源如何映射到具体资源的规范。production-sw.js和testing-sw.js则是作为注入器的service worker。生产环境使用default-mapping.js规范而不用进行修改，而测试环境则覆盖utils/dialogs来替改服务模型。

比较actual-dialogs.js和mock-dialogs.js中对话界面的实际和模拟实现。

# 方案类型

离线扩展

# 代码示例

index.html

```html
<!doctype html>
<html>
  <head>
    <meta charset="utf-8" />
    <title>Dependency injection - ServiceWorker Cookbook</title>
    <meta name="viewport" content="width=device-width, initial-scale=1">
    <script src="bootstrap.js"></script>
    <script defer src="utils/dialogs"></script>
    <script defer src="controller"></script>
  </head>
  <body>
    <p>The demo shows Service Workers as dependency injectors. Depending on your selection,
      a production or testing Service Worker will control this page and will serve proper
      versions of the scripts needed to run the interaction section below.</p>
    <p>Select an environment:</p>
    <p>
      <a href="#production">Switch to production</a> (dialogs will appear using the browser UI)</br>
      <a href="#testing">Switch to testing</a> (dialogs will leave a trace in the <strong>console log</strong>)
    </p>
    <p>What is cool about this demo is that you can see different content for the same resources
      if you explore the source files (Chrome only, Nightly fails to load the resource because it
      actually does not exists) with the developer tools.</p>
    <h2>Interaction</h2>
    <p>
      <button id="show-alert">Show alert</button>
      <button id="show-confirm">Show confirm</button>
      <button id="show-prompt">Show prompt</button>
    </p>
  </body>
</html>
```

bootstrap.js

```js
window.onhashchange = function() {
  var injector = window.location.hash.substr(1) || 'production';
  var currentInjector = getCurrentInjector();

  if (injector !== currentInjector) {
    // 当新的注入器激活就刷新页面
    navigator.serviceWorker.oncontrollerchange = function() {
      this.controller.onstatechange = function() {
        // 现实中的框架可能会使用异步加载更新文件，而不是刷新页面
        if (this.state === 'activated') {
          window.location.reload();
        }
      };
    };
    registerInjector(injector);
  }
};

// 调用初始化
window.onhashchange();

// 根据环境类型注册新的service worker
function registerInjector(injector) {
  var injectorUrl = injector + '-sw.js';
  return navigator.serviceWorker.register(injectorUrl);
}

// 用已经注册的service worker监控当前注入器
function getCurrentInjector() {
  var injector;
  var controller = navigator.serviceWorker.controller;
  if (controller) {
    injector = controller.scriptURL.endsWith('production-sw.js')
                 ? 'production' : 'testing';
  }
  return injector;
}
```

actual-controller.js

```js
document.getElementById('show-alert').onclick =
  dialogs.alert.bind(dialogs, 'Hello!');

document.getElementById('show-confirm').onclick =
  dialogs.confirm.bind(dialogs, 'Do you like these demos?');

document.getElementById('show-prompt').onclick =
  dialogs.prompt.bind(dialogs, 'Enter your rate about these demos:');
```

actual-dialogs.js

```js
(function(window) {
  // 实际的dialog可以用window自带的功能，但是通常会重新定义HTML5的UI
  window.dialogs = {
    alert: function(msg) {
      window.alert(msg);
    },

    confirm: function(msg) {
      return window.confirm(msg);
    },

    prompt: function(msg) {
      return window.prompt(msg);
    }
  };
})(window);
```

default-mapping.js

```js
var mapping = {
  'controller': 'actual-controller.js',
  'utils/dialogs': 'actual-dialogs.js'
};
```

injector.js

```js
function onInstall(event) {
  event.waitUntil(self.skipWaiting());
}

function onActivate(event) {
  event.waitUntil(self.clients.claim());
}

// 简单实现一个从抽象请求中找到实际资源URL的方法，如果没有找到就直接请求这个抽象资源，在本例子中没有模拟失败的
function onFetch(event) {
  var abstractResource = event.request.url;
  var actualResource = findActualResource(abstractResource);
  event.respondWith(fetch(actualResource || abstractResource));
}

function findActualResource(abstractResource) {
  var actualResource;
  var patterns = Object.keys(mapping);
  for (var index = 0; index < patterns.length; index++) {
    var pattern = patterns[index];
    // 一个超简单的匹配器仅为学习目的
    if (abstractResource.endsWith(pattern)) {
      actualResource = mapping[pattern];
      break;
    }
  }
  // 如果没有资源则返回undefined
  return actualResource;
}
```

mock-dialogs.js

```js
(function(window) {
  window.dialogs = {
    alert: function(msg) {
      console.log('alert:', msg);
    },

    confirm: function(msg) {
      console.log('confirm:', msg);
      return true;
    },

    prompt: function(msg) {
      console.log('prompt:', msg);
      return 'test';
    }
  };
})(window);
```

production-sw.js

```js
importScripts('injector.js');

// 默认映射包含抽象资源到其具体对应的映射。
importScripts('default-mapping.js');

// 所有事件都定义在`injector.js`，这里我们只是调用对应方法
self.onfetch = onFetch;
self.oninstall = onInstall;
self.onactivate = onActivate;
```

testing-sw.js

```js
importScripts('injector.js');

// 加载抽象和具体资源的映射文件
importScripts('default-mapping.js');

// 重写``utils/dialogs``来替代fuwu模型
mapping['utils/dialogs'] = 'mock-dialogs.js';

self.onfetch = onFetch;
self.oninstall = onInstall;
self.onactivate = onActivate;
```



