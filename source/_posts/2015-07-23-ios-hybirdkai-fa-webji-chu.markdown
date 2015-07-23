---
layout: post
title: "IOS Hybird开发 - Web基础"
date: 2015-07-23 17:21:17 +0800
comments: true
categories: [iOS, Hybird, Web]
---

Hybird开发就是native和web混合开发。他有两方面的优势：跨平台和更新快。
他同时涉及到web开发和native开发。
这里简单讲讲web开发。

 * Web常用目录结构:

[name].html 页面文件

style.css 主样式文件

m-style.css 移动平台主样式文件

img/ 图片目录

js/ javascript文件目录

css/ css文件目录
<!-- more -->

 * html结构示意：
```
<html>
<head>
    <meta charset="utf-8">
    <title>抢红包</title>
    <meta name="viewport" content="width=device-width, initial-scale=1.0, minimum-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <link href="m-style.css" rel="stylesheet" />
    <script src="../vendor/jquery.js"></script>
</head>
<body>
  <div class="topbk">
    	<img src="img/packet_bg@2x.png" width="100%">
    	<p class="top_title">红包，先抢先得！</p>
      <div class="packet_long">
        <div id="head_icon"></div>
        <div id="scramble_button">光速抢</div>
        <p id="packet_history">看看大家的手气></p>
      </div>
  </div>
    <div class="packet_rules">
    <p>
      红包规则：
    </p>
    <p>
      1.手机绑定用户专享； <br>
      2.每个红包，每个账户限领一次；
    </p>
  </div>
    <script src="nativeBridge.js"></script>
    <script src="app.js"></script>
</body>
</html>
```

`<head>`段声明页面属性、包含CSS文件、js文件。head会在body前被加载，一些需要预先加载的文件可以放在head中。
`<body>`段是页面内容
`<p class="top_title">`p是段落元素, `class="top_title"`是元素属性，指定了该元素的类。
`<div id="head_icon">`div是块元素， `id="head_icon"`指定了元素的id.
`<img src="img/packet_bg@2x.png">`img是图像元素, 这里通过src属性指定了该图像的图片地址。

 * CSS结构示意：

```
html {
	height: 100%;
}
body {
	min-width: 320px;
	margin: 0;padding: 0;
	background-color:#FCECD3;
}

.topbk{
	text-align: center;
}
#scramble_button{
	position: absolute;

	background: url(img/packet_button@2x.png) no-repeat;
	background-size: 100% 100%;

	top:150px;
	left:50%;
	width:180px;
	height:27px;
	margin-left:-40%;
	padding-top:10px;
}
```
   这是CSS样式的基本语法
   "XXX"是元素选择器，选择所有XXX元素。
   ".XXX"是类选择器，选择所有class=XXX的元素。
   "#XXX"是id选择器，选择所有id=XXX的元素。

 * js结构示意
 js可以独立成js文件，也可以嵌入在html页面中。

  * 引用js文件:
  ```
     <script src="../vendor/jquery.js"></script>
  ```

  * 内嵌js：
  ```
  <script>
  require(['app/packet_history'], function(model) {
      model.init();
  });
  </script>
  ```

 因为浏览器会按顺序解析标签，将script放到body的最后可以防止script的加载阻塞整个页面的渲染。

 * Web技术（html,css,javascript）参考网站：

 https://developer.mozilla.org/zh-CN/docs/Web

 http://www.w3school.com.cn/

 * 使用jquery：

 [jquery](http://jquery.com/)是一个极通用的JavaScript库，通过他提供的选择器和封装的功能函数，可以大大简化web开发。

 jquery初始化后，定义全局变量$符号作为jquery对象的简写。

 简单示例：
```
$(document).ready(function(){
  $("button").click(function(){
    $("p").hide();
  });
});
```
这段代码功能：当页面里有按钮被点击后，隐藏所有的段落元素。

  * jquery选择器

  $("p") 选取所有 p 元素。

  $('.more') 选取class="more"的元素

  $('#scrumble_button') 选取 id="scrumble_button"的元素

  * ajax请求

```
  $.ajax({
     url: '...',
     type: 'POST',
     dataType: 'json',
     data: {pkgId: '1'},
     success:function(data){
         //解析返回数据
     }
     error:function(request, description, exception) {

     }
 })
```
 [jquery中文学习地址](http://www.w3school.com.cn/jquery/)

 * js模块化

 javascript本身没有提供模块化的功能，所有的js代码都在全局命名空间运行，并且不支持依赖关系。
 幸好我们有第三方可框架可以给我们提供模块化功能。目前主流的模块化规范有两种。

  * AMD

  [RequireJS](http://www.requirejs.cn/)

  * CMD

  [SeaJS](http://seajs.org/docs/)


 [js模块化参考](http://justineo.github.io/singles/writing-modular-js/)

 * Apache Tomcat使用
  * 到其[官网](http://tomcat.apache.org/)下载最新安装包，
  * 解压后运行bin/startup.sh就会启动tomcat服务器。
  * 如果之前没有配置过java的话，你可能需要配置JAVA\_HOME环境变量：
  ```
  export JAVA_HOME=/Library/Java/JavaVirtualMachines/jdk***/Contents/Home
  ```
  * 将网页拷贝到webapps中，这样我们的网页就可以通过url远程访问。本例地址：`http://127.0.0.1:8080/wap/test\_packet/packet.html`
