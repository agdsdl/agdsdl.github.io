---
layout: post
title: "IOS Hybird开发 - 与OC交互"
date: 2015-07-23 17:23:57 +0800
comments: true
categories: [iOS, Hybird, Web]
keywords: "iOS,Hybird,Web"
---
* 原理：

OC调用javascript是通过
```
[webView stringByEvaluatingJavaScriptFromString:@"document.title"];
```

javascript调用OC一般是通过在页面中发起一个特定的url请求，然后在OC中响应
```
-(BOOL)webView:(UIWebView *)webView shouldStartLoadWithRequest:(NSURLRequest *)request navigationType:(UIWebViewNavigationType)navigationType
```
在响应中判断此url，并做相应的动作。

开源库：https://github.com/marcuswestin/WebViewJavascriptBridge
<!-- more -->

* WebView体验优化

若页面结构多于一级，如这个导航顺序pageA->pageB->pageC，这时支持页面的返回就很必要。
也就是从pageC可以返回到pageB，pageB可以返回到pageA。

一般我们通过pushViewController来打开webViewController，而webViewController中加载pageA。
导航到pageB，pageC都是在webViewController中进行。

在pageC中点击导航栏的返回按钮的话，默认会退回到webViewController的上一级页面，而不是从pageC返回到pageB。
这就需要对导航栏的返回按钮做处理。为了优化体验，还需要到滑动手势做特殊处理。

开源库：https://github.com/agdsdl/DLPanableWebView

* 参数传递

如上例，我们打开pageA时可以通过`[webView stringByEvaluatingJavaScriptFromString:@"document.title"];`来传参数给pageA
如果从pageA导航到pageB，要传递一个id到pageB，这时参数要怎么传递？
`stringByEvaluatingJavaScriptFromString`会将参数传给所有web页，用这个函数会导致我们的代码无比混乱。
目前业界常用的方式是通过sessionStorage, localStorage。localStorage是本地存储，长期有效。sessionStorage是会话期间的存储，一旦关闭webView，存储就会消失。
我们在项目中使用的时sessionStorage.

* 插件化

为了节省加载时间，提升用户体验，我们可以web功能模块打包成zip文件，一次性下载到本地。
使用时直接从本地加载，可以大大减少加载时间，同时也能节省不少带宽。

* 调试

纯web页面用chrome调试最方便。

hybird开发中的webView用safari调试即可。
 * 在safari选项中勾选“显示开发菜单”。
 * 在手机或模拟器打开web页面的时候，在开发菜单中选择调试的页面。

* 一些坑
 * stringByEvaluatingJavaScriptFromString要放在主线程调用。
 * 从OC回调到js时，不能立即弹出alert，哪怕延时0秒弹出都行。
