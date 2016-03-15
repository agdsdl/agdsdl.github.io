---
layout: post
title: "Nodejs 环境搭建"
date: 2016-03-15 12:37:13 +0800
comments: true
categories: [nodejs, vscode]
---
开始学习Nodejs了，在这里记录一些轨迹吧。

工欲善其事，必先利其器。在开始之前，先搭好顺手的工具。
Nodejs的安装网上有很多教程了，这里着重介绍开发工具vscode的安装和配置。
vscode是挺不错的nodejs编辑工具，支持代码提示、自动完成和debug，并且它本身也是nodejs编写的。
当然如果你发现有更好用的工具，也请告诉我。

##nodejs安装
安装Nodejs，我是从这里看的：[七天学会nodejs](http://nqdeng.github.io/7-days-nodejs/), 还有[官网](https://nodejs.org/).

##下载和安装
下载地址：http://code.visualstudio.com/

##配置命令行
配置命令行，使其可以用命令行这样启动：（仅限MacOS）
``` bash
vscode .
```
###MacOS系统：
在.bash_profile文件中加入下面代码：
```
function vscode () { VSCODE_CWD="$PWD" open -n -b "com.microsoft.VSCode" --args $*; }
```
上面代码中的`com.microsoft.VSCode`是应用的`bundle identifier`。查看方法：
1. 在应用上点击右键，选择“显示包内容”。
2. 打开`Contents`目录中的`info.plist`
3. 其中`CFBundleIdentifier`键下面的值就是应用的`bundle identifier`。
其实这种方式适用于大多数Mac应用，比如你可以参考我的[文件](https://github.com/agdsdl/dotfiles/blob/master/source/52_editors.sh)

###其他系统：
请参考https://code.visualstudio.com/docs/editor/setup

##配置代码提示和自动完成
vscode使用TypeScript定义文件（比如`node.d.ts`）来提供代码提示和自动完成功能。
vscode推荐使用typings安装和管理TypeScript文件。
首先使用下面代码安装typings：
``` bash
npm install -g typings
```
这将在全局安装typings。

然后，在你的工程目录下，安装你需要的TypeScript文件：
``` bash
typings install node --ambient
typings install express --ambient
# 或者合并成一行：
# typings install node express --ambient
```
安装完之后，应该就可以看到代码提示了。

##更多配置选项
在工程目录下新建一个文件`jsconfig.json`，填入以下内容：
```
{
    "compilerOptions": {
        "target": "es5",
        "module": "commonjs"
    }
}
```
这告诉vscode你正在使用ES5的javascript标准写代码，下面一行表示支持使用commonjs导出的模块（来进行代码提示）。

<!-- more -->

##调试
###Launch
点击![按钮](https://code.visualstudio.com/images/nodejs_debugicon.png)切换到调试视图。
在调试视图的顶部，点击小齿轮，选择Launch，这会创建一个launch.json配置文件。你可能要改的是`program`字段，其他默认即可。
在代码行的左侧点击来下断点，然后点击启动或者F5即可启动调试：
![调试图](https://code.visualstudio.com/images/nodejs_debugsession.png)

###Attach
如果要Attach到现有node程序，那么你需要这样启动node：
```
node --debug program.js #debug模式启动
node --debug-brk program.js #debug模式启动并停在第一行
```
然后将`launch.json`配置文件改成这样的：
```
{
    "version": "0.2.0",
    "configurations": [
        {
            "name": "Attach to Node",
            "type": "node",
            "request": "attach",
            "port": 5858
        }
    ]
}
```
以上。

参考：
https://code.visualstudio.com/docs/runtimes/nodejs
