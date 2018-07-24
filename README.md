# 介绍

《Service Worker Cookbook》中介绍了一系列在现代网站开发中引入service worker和如何实践的例子。

> 打开浏览器的开发者工具，在console中就可以查看fetch事件以及service worker各种api的详细信息。

# 贡献

这本书是由[Mozilla](https://mozilla.com/)创建，很多象你一样的开发者贡献完成，所有的源码都可以在[github](https://github.com/mozilla/serviceworker-cookbook)上面找到，欢迎大家贡献代码和提交request。

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

## 简单推送使用

## 消息推送的标签

## 消息推送相关指标

## 消息推送在客户端使用

## 推送订阅

## 即时声明

## 消息传递

## 请求远端资源

## 实时流程图

## 离线回调

## 离线状态

## JSON数据缓存

## 本地下载

## 虚拟服务器

## API分析

## 负载均衡

## 缓存压缩

## 依赖注入

## Request Deferrer

## 缓存和网络的请求

## 渲染仓库



