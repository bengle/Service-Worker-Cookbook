# 介绍

《Service Worker Cookbook》中介绍了一系列在现代网站开发中引入service worker和如何实践的例子。

> 打开浏览器的开发者工具，在console中就可以查看fetch事件以及service worker各种api的详细信息。

# 贡献

这本书是由[Mozilla](https://mozilla.com/)发起，很多象你一样的开发者贡献完成，所有的源码都可以在[github](https://github.com/mozilla/serviceworker-cookbook)上面找到，欢迎大家贡献代码和提交request。

缓存章节中用到的图片都可以在[lorempixel.com](http://lorempixel.com/)上找到。

# 大纲

## 网络和缓存

用service worker从网络上获取即时的数据，但是如果网络拥堵响应不及的时候，它会用缓存的数据替代网络数据。

## 纯缓存

介绍service worker如何在fetch事件中始终返回缓存中的数据。

## 缓存与更新

介绍使用service worker读取缓存数据快速响应请求，同时从网络更新缓存数据。

## 缓存、更新与刷新

介绍使用service worker读取缓存数据快速响应请求，从网络更新缓存数据，而且当网络请求的数据响应之后自动更新网站界面。

## 嵌入式回调

介绍service worker在资源缺失情况下提供嵌入式内容回调的使用方案。

## 推送与信息获取

使用service worker发送一个推送消息请求，以及接受一条消息推送时获取内容。

## 推送信息

使用service worker发送一条附带信息的推送消息，我们的例子会教你如何发送或接收一个字符串，但是实际上我们可以从推送消息中提取各种格式的数据，例如：String, ArrayBuffer, Blob, JSON。

## 推送富文本

使用service worker推送富文本消息，可以定义消息的语言，振动模式以及关联的图片等。在[https://notifications.spec.whatwg.org/\#api](https://notifications.spec.whatwg.org/#api) 上你可以查看所有可以设置的参数，例如：可以从通知激活一组操作。

## 简单推送使用

使用Web Push API最简单的例子，推送一条消息给用户，即使在你的页面没有打开的情况下。

## 消息推送的tag

使用tag标记推送消息，将旧的消息替换为新的。这样可以向用户推送最新的消息，还可以将多个消息折叠为一个推送消息。

## 消息推送相关指标

消息推送在不同浏览器下指标配置实践，包含：发送可见或不可见的消息，浏览器选项卡打开和关闭情况下的不同表现，点击推送消息和不点击分别会发生什么。

## 消息推送在客户端使用

当用户点击一个推送事件出发的消息时，service worker是如何控制客户端的。你可以切换聚焦该应用的选项卡，甚至关闭它再重新打开。

## 推送订阅

介绍如何使用消息推送的订阅管理。

## 即时声明

介绍如何在无需等待navigation事件的情况下，让service worker立即控制页面。

## 消息代理

介绍service worker是如何和页面通信的，并教你如何用service worker代理不同页面间的消息传递。

## 请求远端资源

介绍两种servie worker加载远程资源的标准用法，其中一种方法是将service worker作为代理中间件使用。

## 实时流程图

介绍如何使用service worker实现Mozilla开发者网站上提供的工作流程图的例子，这个例子运行期间会在屏幕上打印出service worker运行的所有步骤日志。

## 离线回调

本章介绍在用户离线环境下如何用service worker从缓存中获取数据替代网络环境下要获取的内容。

## 离线状态

介绍如何用service worker缓存用户的关键资源，并告知用户在离线环境下可以得到同在线环境一样的用户体验。

## JSON数据缓存

本案例介绍了在service worker装载期间获取JSON文件并将所有资源添加到缓存的操作，同时这个例子中还用到了service worker即时声明快速激活。

## 本地下载

本案例介绍如何让用户“下载”在客户端生成的文件。

## 虚拟服务器

本案例介绍如何使用service worker当作远程服务器。

## API分析

本案例用来显示API使用日志，例子中介绍如何使用service worker收集API使用情况并实时异步上传这些内容，从而达到显示这些使用日志而不会干扰到UI层效果。

## 负载均衡

本案例介绍如何使用service worker控制网络逻辑，以根据服务器情况动态选择最佳服务提供者。

## 缓存压缩

本案例介绍如何从zip文件缓存内容。

## 依赖注入

本案例使用service worker充当依赖注入器，解决避免高级组件复杂的线性依赖问题。

## Request Deferrer

本案例实现了一个类似发件箱的缓存区，可以在离线情况下将请求加入队列，以便在重新获取连接后执行操作。

## 缓存和网络的请求

本案例介绍从缓存或者网络返回应用网络请求的方法。

## 渲染仓库

本案例是NGA的一项提案，缓存插值模版内容，以避免在连续请求时获取模型和渲染的时间。



