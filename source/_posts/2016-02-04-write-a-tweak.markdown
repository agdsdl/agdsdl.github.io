---
layout: post
title: "编写一个iOS越狱插件(tweak)"
date: 2016-02-04 15:38:29 +0800
comments: true
categories: [iOS, Jailbreak]
---


折腾了几天，搞定一个tweak(FPSCounter)，把一些重要步骤记录一下。

## 关于这个插件
这是个帧速计数器(FPSCounter)，方便我们查看应用的帧速，从而做出相应优化。这里不仅可以看自己的应用，也可以看系统中其他应用。

###截图：
![preview1](https://github.com/agdsdl/fpscounter/blob/master/images/preview1.jpg?raw=true)
![preview2](https://github.com/agdsdl/fpscounter/blob/master/images/preview2.jpg?raw=true)
###如何安装
- 在Cydia中搜索`FPSCounter`.(你的Cydia中必须有BigBoss源)
- 或者到[source code](https://github.com/agdsdl/fpscounter)下载后自行编译安装。

## 前提条件

一个越狱的iOS设备。

需要越狱的设备来获取可用的CydiaSubstrate, 同时也需要越狱设备来安装和测试。

## 安装和配置Theos

[Theos](https://github.com/DHowett/theos.git)大大简化了tweak的编写，建议大家从Theos开始编写tweak。

按照[《iOS应用逆向工程》](http://iosre.com/)中所述，将Theos[下载](https://github.com/DHowett/theos.git)下来之后，还有一系列的配置。

我将这些配置工作写成了一个[脚本](https://github.com/agdsdl/dotfiles/blob/master/init/82_jailbreak_tools.sh),省去每次都做这些繁琐的配置。

这个脚本默认Theos是下载到“个人目录/jailbreak/Opensource”，如果不想修改脚本，可用使用与脚本相同的目录。

## 配置默认路径

修改.bash_profile文件，将~/jailbreak/Opensource/theos/bin加入到PATH中。

保存之后重新启动一下终端。

## 获取iOS系统头文件

按照[《iOS应用逆向工程》](http://iosre.com/)的介绍，正规的获取方式是用dyld_decache从iOS设备中/System/Library/Caches/com.apple.dyld/dyld_shared_cache_armXX将二进制文件提取出来，然后用class-dump去把头文件提取出来。

为了方便起见，这里采用第二种方法，就是直接用[rpetrich的头文件](https://github.com/rpetrich/iphoneheaders.git)。

将这些头文件放到Theos的include目录下。

开始编写tweak！

<!-- more -->

## 新建一个tweak

在终端运行nic.pl, 在出现的菜单中选择想要的tweak种类。

一般的tweak是选tweak类型。preference_bundle是可自定义的设置插件，如果你选择这种，会遇到一些奇怪的坑，下面会给出解决办法：）

普通的tweak会包含以下文件：

- control 包含deb包管理的基本信息
- tweak.x 插件源文件，支持c， objectiveC，[Logos语法](http://iphonedevwiki.net/index.php/Logos)
- XXX.plist 指明插件在哪些应用中加载
- layout目录 相当于iOS设备根目录，里面的文件会被对应的安装到设备的对应位置。
- Makefile make文件

## 修改make文件

在Makefile最上面加上这两句：

``` Makefile
export ARCHS = armv7 armv7s arm64
export TARGET = iphone:clang:latest:7.0
```

上面一句表示要编译armv7 armv7s arm64三种cpu架构，

下面一句表示编译iPhone插件：用clang编译：用系统中最新SDK编译：目标系统是7.0（和以上）

## 编译

在终端输入：

``` bash
make
```

编译成功后，输入

``` bash
make package install
```

一条命令完成编译打包和安装。

如果有错误，请参考下面的问题。

## 遇到的问题

- 报错找不到IOSurfaceAPI.h。解决办法：rpetrich头文件包里有这个文件，手动拷贝到include目录。

- 编译错误，报错architecture not supported。解决办法：如果是报错在.m文件或.mm文件，Theos对.m的支持不好，将其后缀改为.x文件或.xm文件一般就可以解决。

- link错误，Undefined symbols。解决办法：Makefile中有XXX_FRAMEWORKS = UIKit，在这后面把你用到的framework都加上。

  如果还用了其他的库，需要在Makefile中加上XXX_LDFLAGS = -lx，这里x是变量，代指库名。

- make package install时报错：make install requires that you set THEOS_DEVICE_IP in your environment. It is also recommended that you have public-key authentication set up for root over SSH, or you will be entering your password a lot.

  需运行```export THEOS_DEVICE_IP=你手机的ip地址```。

  后面那句话是建议你把电脑ssh 公钥设为手机的authorized_keys，这样就不用每次输密码了。

- 在设备上插件不运行。解决办法：连上xcode看device log，是否是cpu type不支持或者其他原因。

- 如果还有问题，请仔细看编译警告、device log，并自己打log，直到确定问题所在。

- tweak默认是mrc的，一定要正确保持和释放。这里不太建议改成ARC，除非[你知道你在干嘛](http://iphonedevwiki.net/index.php/Using_ARC_in_tweaks)。

## 学习和模仿

成长离不开学习和模仿，有以下途径可以模仿：

- 在手机上用cydia查看已安装的tweak，看他们有哪些文件，然后用iFunbox将这些文件拷贝出来学习或逆向。
- github上有大量开源的代码（比如大神rpetrich、DHowett等），可以看看别人是怎么写的。

## 参考

http://iphonedevwiki.net/index.php/Main_Page

https://www.andyibanez.com/creating-aggregate-projects-theos-configurable-tweak/

https://github.com/mlnlover11/TouchIDEverywhere/issues/2

https://github.com/DHowett/theos/issues/73

http://bbs.iosre.com/t/theos-make-error-unsupported-architecture/1196
