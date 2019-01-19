# Local Download

允许用户下载客户端生成的文件。

# 难度等级

初级

# 应用场景

通常在单页面应用中会需要必要的下载功能，例如：绘图程序可能希望能够导出SVG格式文件或者bitmap格式文件。

# 解决方案

用service worker拦截form表单post请求，从请求中提取数据，然后将数据放入下载附件的请求中，将其作为附件返回给客户端，看起来文件是下载来的，但是实际上并未通过服务器。

# 方案类型

离线扩展

# 代码示例

index.html

```html
<!doctype html>
<html>
<head>
  <meta charset="utf-8">
  <title>Local Download - ServiceWorker Cookbook</title>
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
  <form action="download-file" method="POST">  
  <label for="filename">
    Choose a name for the file.
    <input type="text" name="filename" value="textfile.txt" />
  </label>
  <label> 
    Fill out file contents
    <textarea name="filebody" id="filebody" rows="12" cols="80">
Click below to download this message as a text file.
    </textarea>
  </label>
  <div>
    <input class="button" type="submit" id="download" value="Download this file"/>
  </div>
  </form>
  <script src="./index.js"></script>
</body>
</html>

```

index.js

```js
// 注册service worker
navigator.serviceWorker.register('service-worker.js', {
  scope: '.'
})
```

service-worker.js

```js
// 监听下载文件的post请求
self.addEventListener('fetch', function(event) {
  // 如果请求匹配download-file则格式化post的data并返回文件
  // 格式化转换也可以调用服务端的方法做，只要添加一个回调函数即可
  if(event.request.url.indexOf("download-file") !== -1) {
    event.respondWith(event.request.formData().then(function (formdata){
      var filename = formdata.get("filename");
      var body = formdata.get("filebody");
      var response = new Response(body);
      response.headers.append('Content-Disposition', 'attachment; filename="' + filename + '"');
        return response;
      }));
    }
});
```



