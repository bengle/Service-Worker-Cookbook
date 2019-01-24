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
    // When the new injector is activated, reload...
    navigator.serviceWorker.oncontrollerchange = function() {
      this.controller.onstatechange = function() {
        // ...if this were a real framework, instead of reloading we would
        // start to load the view scripts in an asynchronous way but providing
        // such mechanisms is out of the scope of this demo.
        if (this.state === 'activated') {
          window.location.reload();
        }
      };
    };
    registerInjector(injector);
  }
};

// Force the initial check.
window.onhashchange();

// Register one or another Service Worker depending on the type of environment.
function registerInjector(injector) {
  var injectorUrl = injector + '-sw.js';
  return navigator.serviceWorker.register(injectorUrl);
}

// Gets the current injector inspecting the service worker registered, if any.
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
  // The real implementation of dialog rely on the browser dialogs but it could
  // implement their own full HTML5 UI.
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

// Easy, simply try to find an actual resource URL for an abstract request.
// If not found, let's fetch the abstract resource. In this demo, this never
// fails.
function onFetch(event) {
  var abstractResource = event.request.url;
  var actualResource = findActualResource(abstractResource);
  event.respondWith(fetch(actualResource || abstractResource));
}

// Look inside mapping to get the actual resource URL for the abstract resource URL
// passed as parameter. This act like the dependency factory in charge of creating
// the objects to be injected.
function findActualResource(abstractResource) {
  var actualResource;
  var patterns = Object.keys(mapping);
  for (var index = 0; index < patterns.length; index++) {
    var pattern = patterns[index];
    // A really silly matcher just for learning purposes.
    if (abstractResource.endsWith(pattern)) {
      actualResource = mapping[pattern];
      break;
    }
  }
  // Can return undefined if there is no actual resource.
  return actualResource;
}
```

mock-dialogs.js

```js
(function(window) {
  // The module mock dialogs exports a non blocking fake implementation
  // that simply logs the calls.
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

// The default mapping contains the map for abstract resources to their
// concrete counterparts.
importScripts('default-mapping.js');

// All the functionality is in `injector.js` here we only wire the listeners.
self.onfetch = onFetch;
self.oninstall = onInstall;
self.onactivate = onActivate;
```

testing-sw.js

```js
importScripts('injector.js');

// Load the default mapping between abstract and concrete resources.
importScripts('default-mapping.js');

// But override ``utils/dialogs`` to serve the mockup instead.
mapping['utils/dialogs'] = 'mock-dialogs.js';

self.onfetch = onFetch;
self.oninstall = onInstall;
self.onactivate = onActivate;
```



