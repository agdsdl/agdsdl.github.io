---
layout: post
title: "搭建基于Octopress的博客"
date: 2015-07-29 10:25:47
comments: true
categories: [Octopress]
keywords: "Octopress,blog"
---
## 2019.11.12更新
关于系统更新到MacOS 10.14之后ruby环境的调整，以及多说关闭之后评论系统的选择。
### 系统更新到10.14之后
`gem install`工具会出错：
```
$ sudo gem install bundler
Password:
Fetching: bundler-2.0.2.gem (100%)
ERROR:  While executing gem ... (Gem::FilePermissionError)
    You don't have write permissions for the /usr/bin directory.
```
这是因为Apple在OS X El Capitan中全面启用了名为System Integrity Protection (SIP)的系统完整性保护技术。大部分系统文件禁止直接修改。

要退出SIP需要重启到recovery模式下进行（重启按住command+r），这里采取另一个办法：
`/usr/bin`禁止写入后，一般都用`/usr/local/bin`目录替代。`gem install`可以使用 `-n install_path`指定安装路径：
```
sudo gem install xxx -n /usr/local/bin
```

bundle install的时候还会出错:
```
$ bundle install --path vendor/bundle
Fetching gem metadata from https://rubygems.org/..........
Using rake 10.5.0
Fetching RedCloth 4.2.9
Installing RedCloth 4.2.9 with native extensions
Gem::Ext::BuildError: ERROR: Failed to build gem native extension.
  ...
mkmf.rb can't find header files for ruby at
/System/Library/Frameworks/Ruby.framework/Versions/2.3/usr/lib/ruby/include/ruby.h

extconf failed, exit code 1
```

因为macos10.14之后xcode11内置了macos10.15的SDK，预装了ruby2.6，没有ruby2.3的目录
```
/Applications/Xcode.app/Contents/Developer/Platforms/MacOSX.platform/Developer/SDKs/MacOSX10.15.sdk/System/Library/Frameworks/Ruby.framework/Versions/2.6/usr/include/ruby-2.6.0
```

但是CommandLineTools里面还有macos10.14的SDK：
```
/Library/Developer/CommandLineTools/SDKs/MacOSX10.14.sdk/System/Library/Frameworks/Ruby.framework/Versions/2.3/usr/include/ruby-2.3.0
```

通过
```
sudo xcode-select --switch /Library/Developer/CommandLineTools
```
来切换SDK路径

用
```
ruby -rrbconfig -e 'puts RbConfig::CONFIG["rubyhdrdir"]'
```
可以看到路径正确。

之后`bundle install`可以正常安装。

需要切换回xode11的sdk时：
```
sudo xcode-select --switch /Applications/Xcode.app
```

### 评论系统的选择
博客之前采用多说作为评论系统，可是多说关闭了，类似的友言、网易云跟帖等国内评论系统也相继停止服务。Disqus在国内一直被墙，需要重新选择一个评论系统。

看了几天，基本瞄中了基于github issue的评论系统，如果blog也host在github上，那么就很相得益彰。它的数据也没那么容易丢失，除非github倒闭。。。

一开始选择了名气较大的gitment，试用用了一下发现目前已不可用了：
由于采用github API操作issue，因此带来了跨站(CORS)问题，而github禁止跨站，无法直接访问。

解决办法是加一个中间代理服务器，给服务器返回头上添加跨站运行标志。
gitment作者本来自己搭了个代理服务器给大家使用，可是现在已经停服了，如果自己搭代理服务器代价还是颇高的。
而作者已经弃坑，两年多没有更新了，无奈弃之。

后来发现了类似的评论系统：[gitalk](https://github.com/gitalk/gitalk)，最近一年还在更新，试了下目前可用。
它是怎么解决跨站问题呢？抓包看了下，发现它采用了Heroku提供的API，专门用于转发请求并允许CORS访问：https://cors-anywhere.herokuapp.com

看来允许跨站请求头是个比较常见的问题。
gitalk的接入方式类似gitment：

- 首先要在github中新建一个OAuth Application，记下Client ID和Client Secret
- 修改_layouts下面的post.html，在最下面添加：
```
<section id="comment">
    <h1 class="title">Comments</h1>
    <div id="gitalk-container"></div>
    <link rel="stylesheet" href="https://unpkg.com/gitalk/dist/gitalk.css">
    <script src="https://unpkg.com/gitalk/dist/gitalk.min.js"></script>
    <script>
    const gitalk = new Gitalk({
      clientID: 'GitHub Application Client ID',
      clientSecret: 'GitHub Application Client Secret',
      repo: 'GitHub repo',
      owner: 'GitHub repo owner',
      admin: ['GitHub repo owner and collaborators, only these guys can initialize github issues'],
      id: location.pathname,      // Ensure uniqueness and length less than 50
      distractionFreeMode: false  // Facebook-like distraction free mode
    })
    gitalk.render('gitalk-container')
    </script>
</section>
```
GitHub repo可以直接填blog所在的repo，方便集中管理。

博客发出之后需要自己先浏览并登录github账号，以初始化`issue page`，之后就可以正常评论了。
<!-- more -->

## 概要
不同于其他博客网站，Octopress不需要在网站上编辑页面和发布；
你只需本地写好markdown文件，然后`rake generate`就可以生成博客空间的所有静态网页，
然后`rake deploy`就可以将网站更新到服务器。
Octopress支持两种更新方式：github和Rsync
### github
github提供了免费的静态网页托管服务，你需要新建一个名为username.github.io的代码仓库，将你的网站推送到这个仓库，
之后就可以用浏览器访问你的网站了：username.github.io
采用github更新方式，本质上就是将Octopress生成的本地页面推送到username.github.io
由于github是免费的，通过git管理网站，并且支持CNAME域名指向，这真是一种方便又快捷的建站方式，
因此Octopress推荐使用这种方式，[唐巧的博客](http://blog.devtang.com/2012/02/10/setup-blog-based-on-github/)和本博采用的也是这种方式。

### Rsync
然而你可能有了自己的服务器，或者站点里包含一些动态页面，必须使用自己的服务器。
这时你就可以用Rsync同步方法将网站同步到你的服务器。
Rsync本质是通过ssh将文件拷贝到服务器，
你可以看[这篇官方文档](http://octopress.org/docs/deploying/rsync/)，这里就不深入阐述了。

## Octopress安装
如果你运行的是苹果系统（10.9）并且安装了Xcode，你可以先运行以下命令验证一下：
```
ruby --version  # Should be greater than 1.9.2
```
如果版本号大于等于1.9.2，那么唐巧的博客中下面几步是不需要的：
```
brew update
brew install rbenv
brew install ruby-build

rbenv install 1.9.3-p0
rbenv local 1.9.3-p0
rbenv rehash

brew tap homebrew/dupes
brew install apple-gcc42
```

按照[官网](http://octopress.org/docs/setup/)指导，只需要下面几步：
```
git clone git://github.com/imathis/octopress.git octopress
cd octopress

# Next, install dependencies.
gem install bundler
bundle install
```
bundle install的时候可能会出错，这是因为国内的墙，得把RubyGems源换成taobao的源：
```
$ bundle config mirror.https://rubygems.org https://ruby.taobao.org
```
见http://ruby.taobao.org/

然后再
```
bundle install
```
这样就安装好啦！

## 第一篇博客
现在我们先试着写一篇：（保持当前目录在octopress目前）
```
rake new_post["hello"]
```
如果正确无误的话，命令行会输出：
```
mkdir -p source/_posts
Creating new post: source/_posts/2015-07-27-hello.markdown
```
我们再依次键入以下命令：
```
echo "hello world" > source/_posts/2015-07-27-hello.markdown
rake generate
rake preview
```
我们可以看到web服务已经启动，用浏览器打开127.0.0.1:4000就能看到自动生成的博客站点，怎么样，是不是很漂亮？

在终端上按下Ctrl-C可以终止web服务。

## 配置
我们的web站点已经有了基本框架！可是站点的标题，边栏，风格还都是默认的，这些要怎么改？
主要配置文件是`_config.yml`，网站标题、副标题等都在这里设置：
```
url: http://agdsdl.github.io/             #网站地址
title: 乐の博客                            #网站标题
subtitle: 行而不思则罔，思而不行则怠          #网站副标题
author: agdsdl                            #网站作者，通常显示在页尾和每篇文章的尾部
simple_search: http://google.com/search   # 搜索引擎
description: 一个软件工程师的呓语            #网站的描述，出现在HTML页面中的 meta 中的 description
```

### 导航栏配置：
修改`source\_includes\custom\navigation.html`，该文件是html模板，将其修改成你想要的样子。
如：

```html
<ul class="main-navigation">
  <li><a href="{{ root_url }}/">首页</a></li>
  <li><a href="{{ root_url }}/blog/archives">归档</a></li>
  <li><a href="{{ root_url }}/about">关于</a></li>
</ul>
```
上面的about需要你新建一个页面：
```bash
rake new_page['about']  #创建一个页面，页面路径为source\about\index.markdown
```

### 侧边栏配置
修改`_config.yml`中：

```
default_asides: [asides/recent_posts.html, asides/github.html]      #设置默认边栏包含哪些组件

# blog_index_asides:                                                #索引页面的侧边栏
# post_asides:                                                      #日志页面的侧边栏(通过rake new_post 创建的)
# page_asides:                                                      #静态页面的侧边栏(通过rake new_page 创建的)
```

### 添加文章分类列表到侧边栏
octopress没有自带的支持，所以要自己动手。
这位网友写好了轮子并开源了出来，见https://github.com/tokkonopapa/octopress-tagcloud
将其下载下来后，将tag_cloud.rb拷贝到plugins目录，
然后在source/_includes/custom/aside 中新建category_list.html，键入以下内容：

```html
<section>
  <h1>Categories</h1>
  <ul id="categories">
    { % category_list [counter:true] % }
  </ul>
</section>
```
拷贝代码的话，请将{和%之间的空格去掉。

再修改_config.yml:
```
default_asides: [custom/asides/category_list.html, asides/recent_posts.html, asides/github.html]
```

### 将关于我们添加到侧边栏
octopress自带了关于我们模板：source/_includes/custom/asides/about.html
修改后自行把它加到边栏：
default_asides:     [custom/asides/category_list.html, asides/recent_posts.html, asides/github.html, custom/asides/about.html]

将新浪微博秀添加到边栏的关于我们：

在<http://app.weibo.com/tool/weiboshow>这里获得自己的秀代码，嵌入到about.html中：
```
<section>
  <h1>About me</h1>
  <!-- 嵌入微博秀代码 -->
  <p></p>
</section>
```

### 添加留言评论功能
- octopress自带了DISQUS支持，你需要到DISQUS官网申请好账号，之后再新建一个site。
- 新建site的入口改到了settings->admin->(汉堡菜单)Add new site
- 最后将site短地址填到disqus_short_name中（_config.yml）

### 添加国内社交网站分享和留言评论功能
由于DISQUS服务器在国外，访问较慢，你可以使用国内的留言评论服务。
这部分[唐巧的博客](http://www.devtang.com/blog/2012/02/10/setup-blog-based-on-github/)已经阐述过了，
这里不再重复，请参见唐巧的博客中"高级配置"段。

### 加入google analytics
- octopress自带了google analytics支持。
- 在<http://www.google.com/analytics/>创建帐号和站点。
- 修改_config.yml的google_analytics_tracking_id字段。
- 如果你的博客域名是github的二级域名（比如我这个agdsdl.github.io），你还需要修改source/_includes/google_analytics.html：
```
_gaq.push(['_setAccount', '{{ site.google_analytics_tracking_id }}']);
_gaq.push(['_setDomainName','github.io']);
_gaq.push(['_trackPageview']);
```
好了，到这里我们一些基本设置都完成了，但是octopress里每个文件都是可以修改的，你可以自己修改成喜欢的样式。
网上有很多这样的文章，你可以在文末的参考文章中进一步阅读。

## 写博客
用这个命令生成一个markdown文件，
```
rake new_post[post-name]
```
然后修改这个markdown文件。[markdown语法简略版](http://www.ituring.com.cn/article/23),[markdown语法完整版](http://wowubuntu.com/markdown/)

### 博客首页只显示摘要
在文章中插入
```html
<！-- more -->
```
(拷贝代码的话，请将中文感叹号改成英文感叹号)

在首页列表中文章就只显示到这个位置并在下方显示加载更多（Read on →）

## 写静态页面
```
rake new_page[super-awesome]
```

## 部署
在rake deploy之前，你必须运行：
```
rake setup_github_pages
```
将你的`Git Url` 输入进去完成github配置。

之后就能
```
rake generate  #生成本地网页
rake deploy    #推送到github
```

## source管理
上面将生成的网页都推送到github了。可是本地的配置文件和blog源文件要怎么管理？
用下面的命令可以将源码推送到github上的source分支：
```
git add .
git commit -m "your message"
git push origin source
```

## 在其他电脑上同步你的Octopress博客
``` bash
git clone https://github.com/your_account/your_account.github.io.git
cd your_account
git checkout source #切换到source分支
rake preview #先preview测试一下
#CTRL-C结束preview
#在新的电脑环境需要重新设置github账户：
rake setup_github_pages
#然后就可以发布了：
rake deploy
```

## OS X EI Capitan 更新
升级到新系统后，`rake preview`可能会报错：
``` bash
rake preview
Starting to watch source with Jekyll and Compass. Starting Rack on port 4000
rake aborted!
Errno::ENOENT: No such file or directory - jekyll
```
这需要更新到最新ruby（2.2.3），用：
``` bash
ruby --version
ruby 2.0.0p645 (2015-04-13 revision 50299) [universal.x86_64-darwin15]
```
来确定ruby的版本号。

rbenv是很好的ruby版本管理工具，我们用rbenv来升级ruby。

首先升级rbenv，而rbenv是通过Homebrew管理的，所以要先升级Homebrew。

可以通过下面命令来升级brew和rbenv：
``` bash
brew update
brew install rbenv ruby-build
```
可是我更推荐用一个图形化管理工具[Cakebrew](https://www.cakebrew.com/)来升级。

更新完rbenv后，再更新ruby：
``` bash
rbenv install 2.2.3
```
然后设置要使用的ruby版本：
``` bash
rbenv global 2.2.3 #设置全局ruby版本
rbenv local 2.2.3 #设置本地ruby版本
```
要使rbenv的设置生效，需要在`~/.bash_profile`中加入这一句：`eval "$(rbenv init -)"`,然后重启终端。

验证一下ruby版本：
``` bash
ruby --version
ruby 2.2.3p173 (2015-08-18 revision 51636) [x86_64-darwin15]
```
重新安装Octopress的依赖：
``` bash
gem install bundler
When the above completes
rbenv rehash
bundle install
```
这时应该没问题了。

## 参考

[系统升级10.14后ruby路径问题](https://stackoverflow.com/questions/46377667/docker-for-mac-mkmf-rb-cant-find-header-files-for-ruby)

[一些评论系统简述](https://blog.shuiba.co/comment-systems-recommendation)
[gitment跨站问题](https://aimingoo.github.io/1-1722.html)

http://blog.devtang.com/2012/02/10/setup-blog-based-on-github/

http://www.cnblogs.com/oec2003/archive/2013/05/31/3109577.html

http://yulingtianxia.com/blog/2014/04/05/macosx10-dot-9shang-yong-octopresshe-githubda-jian-ge-ren-bo-ke/

http://octopress.org/docs/

http://schalkneethling.github.io/blog/2015/10/16/errno-enoent-no-such-file-or-directory-jekyll-octopress-el-capitan/

http://about.ac/2012/04/install-ruby-with-rbenv.html
