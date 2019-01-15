# Live Flowchart

本文通过Mozilla Developer Network上展示的SW workflow流程图给大家演示一种学习如何使用service worker的方法。示例中将一个真实Web App运行service worker的步骤打印在屏幕上方便大家理解。

# 难度等级

高级

# 截屏展示

Register + Unregister![](/assets/register-unregister.png)Active service worker on page load + unregister![](/assets/active-service-worker-unregister.png)Wrong service worker scriptURL![](/assets/wrong-scriptURL.png)Security error![](/assets/security-error.png)

# 功能和用法

功能：

* 注册service worker
* 刷新页面
* 注销service worker

这些功能都集中在页面顶部的按钮上，你可以按任意顺序按下尝试，你也可以指定SW scriptURL和scope来模拟不同的测试用例。

用法：

* 点击按钮
* 查看日志
* 必要的行动（例如：打开about://serviceworkers）
* 修改代码看看会发生什么

# 怎么读取日志

查看两种日志：

* HTML日志
* 浏览器日志

HTML日志中不同颜色代表不同的等级：

* 红色代表错误
* 黄色代表告警
* 绿色代表信息
* 白色代表日志
* 灰色代表debug

浏览器日志：

* 同一条日志与HTML日志等级一致
* service worker日志

# 兼容性

Mac OSX 10.8.5上以下浏览器测试正常：

* Firefox Nightly 44.0a1\(2015-10-12\)
* Chrome Canary 48.0.2533.0
* Opera 32.0

注意：这些浏览器必须支持ES6

# 下一步计划

* 支持移动端
* 用svg构建流程图将service worker状态可视化
* 对现有功能做自动化测试
* 对新功能按BDD/TDD模式添加自动化测试
* 在oninstall阶段做一些事情
* 在onactivate阶段做一些事情
* 在onfetch阶段做一些事情
* 添加一个按钮模拟离线状态
* 添加一个按钮o模拟lie-li网络状态（低速网络但非离线）

# 方案类型

通用

# 代码示例

index.html

```html
<!doctype html>
<html>
	<head>
		<title>Live flowchart</title>
		<link href="style.css" type="text/css" rel="stylesheet" />
	</head>
	<body>
		<div class="container">
			<div class="flowchart">
				<h1><a href="https://developer.mozilla.org/en-US/docs/Web/API/Service_Worker_API/Using_Service_Workers" target="_blank">Using Service Workers</a></h1>

				<img alt="sw-flowchart" src="https://qiyouxiaogui.nos-eastchina1.126.net/security-error.png" />
			</div>

			<div class="playground">
				<div class="inputs">
					<span class="title">Service Worker</span>
				    <input id="swscripturl" placeholder="SW path" value="./service-worker.js" />
				    <span class="title">Options: </span>
				    <input id="swscope" placeholder="scope (optional)" />
				    <span>This works: './service-worker.js' (in scope './')</span>
				</div>
				<div class="actions">
					<ul>
						<li><button id="swinstall" disabled>Register specified SW</button></li>
					    <li><button id="reloadapp">Reload document</button></li>
					    <li><button id="swuninstall" disabled>Unregister active SW</button></li>
					</ul>
				</div>
			    <div id="log" class="log"></div> 
				<div class="credits">
				 	<a href="http://www.francesco.iovine.name/" target="_blank">@franciov</a>
			 	</div>		    
		 	</div>

			<script src="logger.js"></script>
			<script src="service-worker-util.js"></script>
			<script src="app.js"></script>
		</div>
	</body>
</html>
```

app.js

```js
/* global Logger, SWUtil */
function App() {
  this.constructor();
}

App.prototype.constructor = function constructor() {
  var thisapp = this;

  Logger.log('App started');
  this.swUtil = new SWUtil();
  document.querySelector('#reloadapp').addEventListener('click', function() {
    window.location.reload();
  });
  if (this.swUtil.areServiceWorkersSupported()) {
    document.querySelector('#swinstall').addEventListener('click',
      function() {
        thisapp.enableCoolFeatures();
      }
    );

    document.querySelector('#swuninstall').addEventListener('click',
      function() {
        thisapp.disableCoolFeatures();
      }
    );
    
    if (this.swUtil.isServiceWorkerControllingThisApp()) {
      Logger.info('App code run as expected');

      this.disableSWRegistration();
    } else {
      this.enableSWRegistration();
    }
  } else {
    Logger.error('Service workers are not supported by this browser');
  }
};

App.prototype.enableCoolFeatures = function enableCoolFeatures() {
  var scriptURL;
  var scope;

  Logger.newSection();
  Logger.log('Enabling cool features...');
  
  scriptURL = document.querySelector('#swscripturl');
  scope = document.querySelector('#swscope');

  Logger.debug(
    'Configuring the following service worker ' + scriptURL.value +
    ' with scope ' + scope.value
  );

  if (scriptURL.value !== '') {
    Logger.debug('scriptURL: ' + scriptURL.value);
  } else {
    Logger.error('No SW scriptURL specified');
    return;
  }

  if (scope.value !== '') {
    Logger.debug('scope: ' + scope.value);
  } else {
    Logger.warn('scope: not specified ' +
      '(scope defaults to the path the script sits in)'
    );
  }
  
  this.swUtil.registerServiceWorker(scriptURL.value, scope.value).then(
      this.disableSWRegistration, // success
      this.enableSWRegistration // error
  );
};

App.prototype.disableCoolFeatures = function disableCoolFeatures() {
  Logger.newSection();
  Logger.log('Disabling cool features...');

  this.swUtil.unregisterServiceWorker().then(
      this.enableSWRegistration, // success
      this.disableSWRegistration // error
  );
};

App.prototype.enableSWRegistration = function() {
  document.querySelector('#swinstall').disabled = false;
  document.querySelector('#swuninstall').disabled = true;
};

App.prototype.disableSWRegistration = function() {
  document.querySelector('#swinstall').disabled = true;
  document.querySelector('#swuninstall').disabled = false;
};

var app = new App();
console.debug(app);
```

logger.js

```js
var Logger = {
  logDomObj: document.querySelector('#log'),
  debug: function debug(message) {
    this.writeLog(message, { debug: true });
  },
  info: function info(message) {
    this.writeLog(message, { info: true });
  },
  log: function log(message) {
    this.writeLog(message);
  },
  warn: function warn(message) {
    this.writeLog(message, { warn: true });
  },
  error: function error(message) {
    this.writeLog(message, { error: true });
  },
  newSection: function newSection() {
    var separator = '----------';
    var newLine = document.createElement('div');
    newLine.innerHTML = separator;
    this.logDomObj.appendChild(newLine);
    console.log(separator);
  },
  writeLog: function writeLog(message, level) {
    var logMessage = document.createElement('div');

    if (message) {
      logMessage.innerHTML = message;
    } else {
            if (message === null) {
        logMessage.innerHTML = 'null';
      } else if (message === undefined) {
        logMessage.innerHTML = 'undefined';
      }
    }
    if (level) {
      if (level.error) {
        logMessage.setAttribute('class', 'error');
        console.error(message);
      } else if (level.warn) {
        logMessage.setAttribute('class', 'warning');
        console.warn(message);
      } else if (level.info) {
        logMessage.setAttribute('class', 'info');
        console.info(message);
      } else if (level.debug) {
        logMessage.setAttribute('class', 'debug');
        console.debug(message);
      }
    } else {
      console.log(message);
    }
    this.logDomObj.appendChild(logMessage);
    this.logDomObj.scrollTop = this.logDomObj.scrollHeight;
  },

};

Logger.newSection();
```

service-worker-util.js

```js
/* global Logger */
function SWUtil() {
}

SWUtil.prototype.areServiceWorkersSupported = function() {
  Logger.newSection();
  Logger.log('Are Service Workers Supported?');

  Logger.debug('checking navigator.serviceWorker');

  if (navigator.serviceWorker) {
    Logger.info('Service workers supported by this browser');
    return true;
  }

  Logger.warn('No, service workers are NOT supported by this browser');
  Logger.debug(navigator.userAgent);
  return false;
};

SWUtil.prototype.isServiceWorkerControllingThisApp = function() {
  Logger.newSection();
  Logger.log('Are Service Workers in control?');

  Logger.debug('checking navigator.serviceWorker.controller');
  
  if (navigator.serviceWorker.controller) {
  
    Logger.info('A service worker controls this app');
    Logger.debug('The following service worker controls this app: ');
    Logger.debug(navigator.serviceWorker.controller.scriptURL +
                ' <= navigator.serviceWorker.controller.scriptURL');
    Logger.info('Please enable and check the browser logs for the oninstall' +
                ', onactivate, and onfetch events');
    Logger.debug('More on about:serviceworkers (Firefox) ' +
                 'or chrome://serviceworker-internals/');

    return true;
  }
  Logger.warn('No, there is no service worker controlling this app');
  return false;
};

SWUtil.prototype.registerServiceWorker = function(scriptURL, scope) {
  var swRegisterSecondParam = {};

  Logger.newSection();
  Logger.log('Registering service worker...');
  Logger.info('Registering service worker with serviceWorker.register()');

  Logger.debug('navigator.serviceWorker.register(scriptURL, scope)');
  Logger.debug('scriptURL: ' + scriptURL);
  
  if (scope) {
    swRegisterSecondParam.scope = scope;
    Logger.debug('options.scope: ' + swRegisterSecondParam.scope);
  }

  return new Promise(
    function then(resolve, reject) {
      navigator.serviceWorker.register(
        scriptURL,
        swRegisterSecondParam
      ).then(
        function registrationSuccess(swRegistration) {
          Logger.debug('registrationSuccess(swRegistration)');
          Logger.debug(swRegistration);
          
          if (swRegistration.active) {
            Logger.debug('Registering the active service worker again: ');
            Logger.debug(swRegistration.active.scriptURL +
                         ' <= swRegistration.active.scriptURL');
          }
          if (swRegistration.installing) {
            Logger.debug('Registering the following service worker for ' +
                         'the first time: ');
            Logger.debug(swRegistration.installing.scriptURL +
                         ' <= swRegistration.installing.scriptURL');
          }

          if (swRegistration.scope) {
            Logger.debug('with scope: ' + swRegistration.scope +
                         ' <= swRegistration.scope');
          }
          Logger.info('Service worker successfully registered');
          Logger.info('Please enable and check the browser logs for' +
                      ' the oninstall, onactivate, and onfetch events');
          Logger.info('SW is in control, once document is reloaded');
          Logger.debug('More on about:serviceworkers (Firefox) ' +
                       'or chrome://serviceworker-internals/');
          resolve();
        },
        function registrationError(why) {
          Logger.debug('registrationError(why)');
          Logger.info('Service worker NOT registered');
          Logger.error(why);

          reject();
        }
      );
    }
  );
};

SWUtil.prototype.unregisterServiceWorker = function() {
  Logger.newSection();

  Logger.log('Unregistering the active service worker...');
  Logger.debug('if something goes wrong, please unregister ' +
               'the service worker ');
  Logger.debug('about:serviceworkers (Firefox) ' +
               'or chrome://serviceworker-internals/');

  return new Promise(
    function then(resolve, reject) {
      Logger.debug('navigator.serviceWorker.getRegistration()' +
                   '.then(success, error)');
      navigator.serviceWorker.getRegistration()
      .then(function getRegistrationSuccess(swRegistration) {
        Logger.debug('getRegistrationSuccess(swRegistration)');
        Logger.debug(swRegistration);
        Logger.debug('Unregistering the following service worker: ' +
          swRegistration.active.scriptURL +
          ' <= swRegistration.active.scriptURL');
        Logger.debug('with scope ' + swRegistration.scope +
                     ' <= swRegistration.scope');
        Logger.debug('swRegistration.unregister().then()');
        swRegistration.unregister()
          .then(
            function unregisterSuccess() {
              Logger.debug('unregisterSuccess()');
               Logger.info('The service worker has been ' +
                          'successfully unregistered');
               Logger.debug('More on about:serviceworkers (Firefox) ' +
                           'or chrome://serviceworker-internals/');

              resolve();
            },
            function unregisterError(why) {
              Logger.debug('unregisterError(why)');
              Logger.error('The service worker has NOT been unregistered');
              Logger.error(why);

              reject();
            }
          );
      },
      function getRegistrationError(why) {
        Logger.debug('getRegistrationError(why)');
        Logger.error(why);
        Logger.warn('It has been not possibile to unregister' +
                    ' the service worker');
      }
    );
    }
  );
};
```

service-worker.js

```js
console.debug('service-worker.js: this is a service worker');
/*
  这是官方文档描述的service worker基本架构
  对于service worker通常会遵循以下步骤进行基本设置
  1.通过serviceWorker.register()方法获取并注册service worker，此步骤在SWUtil模块中执行
  2.如果执行成功，则service worker会在ServiceWorkerGlobalScope作用域中发挥作用。这是一种特殊的worker上下文，运行主脚本执行线程，没有DOM访问权限
  3.service worker做好监听事件的准备
  4.随后在访问service worker控制页面时，尝试安装worker；
  install事件总是第一个发送到service worker（这里可以添加启动indexDB和缓存站点资源）
  这与安装native应用程序或Firefox OS应用程序的过程非常相似-使所有内容都可以离线使用；
  更多关于InstallEvent的信息可以参考[这里](https://developer.mozilla.org/en-US/docs/Web/API/InstallEvent)
  5.当oninstall事件触发，就表示service worker正式安装完成了；
  6.接下来是activation，当service worker安装完成后就会触发一个activate事件；
  在onactivate事件中最常用的是清除上一个版本service worker使用的资源；
  7.现在在register()成功后打开的页面都将受到service worker控制。即，document已经开始其包含service worker或者不包含service worker的生命周期，这个状态会随生命周期一直保存，因此没有被service worker控制的页面必须重新加载才可以被其控制；
  更多关于FetchEvent的说明可以参考[这里](https://developer.mozilla.org/en-US/docs/Web/API/FetchEvent)
*/
this.addEventListener('install', function oninstall(event) {
  console.info('Service worker installed, oninstall fired');
  console.debug(event);
  console.info('Use oninstall to install app dependencies');
});
this.addEventListener('activate', function onactivate(event) {
  console.info('Service worker activated, onactivate fired');
  console.debug(event);
  console.info('Use onactivate to cleanup old resources');
});
this.addEventListener('fetch', function onfetch(event) {
  console.info('onfecth fired');
  console.debug(event);
  console.info('Modify requests, do whatever you want');
});
```



