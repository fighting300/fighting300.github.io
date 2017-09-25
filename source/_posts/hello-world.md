---
title: 如何搭建我的博客 - Hexo之Hello World
date: 2017-03-28 01:16:11
tags: blog
categories: tools
---

其实很久以前就搭建完博客的基础部分，只是懒癌发作，迟迟没有补上文章来总结，但是记录自己的心得还是必要的。下面描述下搭建博客的整个流程和一些问题的解决方案。搭建博客其实有很多种框架，比如可能大家熟悉的[WordPress](https://cn.wordpress.org/)、[Hexo](https://hexo.io/zh-cn/)等，这里我们将使用很火很流程的Hexo来作为博客基础框架。

Hexo是一个快速、简洁且高效的博客框架。Hexo使用Markdown（或其他渲染引擎）解析文章，在几秒内，即可利用靓丽的主题生成静态网页。
搭建博客的大体流程主要分为：`GitHub配置`、`Hexo管理`、`域名绑定`三个步骤，非常简单。

### GitHub配置  
首先需要在[GitHub](https://github.com/)上注册一个账号(当然估计你们都有了)。然后创建一个仓库，命名为`xxx.github.io`，xxx会是你注册时的用户名，所以注册账号时认真想个不错的名字吧，因为这个地址会是你的域名(假如你不单独申请域名的话)。

![新建GitHub仓库](http://ojca2gwha.bkt.clouddn.com/Hexo-blog-github.png)  

### Hexo管理

#### 环境安装
在进行下一步前，你需要在你的Mac上安装好git(没安装过的，直接去官网找吧，顺便面壁思过下)和Node.js。

Node安装的最佳方式是使用nvm，这是Node.js的版本管理器，安装方式很简单。

```
// 1. Homebrew 安装方式，此安装方式无需重启
$ brew install nvm  
$ mkdir ~/.nvm
$ export NVM_DIR=~/.nvm
$ . $(brew --prefix nvm)/nvm.sh
// 2. curl安装
$ curl https://raw.github.com/creationix/nvm/master/install.sh | sh
```

安装完nvm后，执行以下命令即可安装Node。
```
$ nvm install stable
```

Hexo安装：  
```
$ sudo npm install -g hexo-cli
```
所有的工具已经安装完成，下面可以创建博客内容，并上传到我们的github仓库了。

<!--more-->

#### 创建博客  

接下来需要用Hexo初始化博客，并更改设置，来发布内容到之前创建的GitHub仓库。

##### 初始化博客  

安装 Hexo 完成后执行下列命令，Hexo将会在指定文件夹中新建博客所需要的文件。
```
$ hexo init xxx.github.io
$ cd xxx.github.io
$ npm install
```
初始化后，你的文件夹里会有这些内容：
```
  .
  ├── _config.yml        #配置文件
  ├── package.json       #数据
  ├── scaffolds          #草稿
  ├── source             #网站内容
  |   ├── _drafts        #草稿
  |   └── _posts         #文章
  └── themes             #主题
```

##### 更新配置

当然我们自己的博客，要选择好看的主题。Hexo比较好的一点是现在有很多免费的主题可以使用，只需要把主题文件文件拷贝到你的/themes目录下，做简单配置就可以使用该主题了。
知乎上有一篇文章罗列了一些[不错的主题](https://www.zhihu.com/question/24422335)，大家可以参考下。这里我们使用了极简的[NexT](http://theme-next.iissnan.com/getting-started.html)这款非常火爆常见的主题。  

```
$ cd xxx.github.io
$ git clone https://github.com/iissnan/hexo-theme-next themes/next
```

之后你需要在`xxx.github.io/_config.yml`中配置博客名、语言、作者等基础配置如下(键值之间需要有空白)，可配置的选项可以参考[Hexo配置](https://hexo.io/zh-cn/docs/configuration.html)。

```
  title: DragonFly的博客
  subtitle: 我曾七次鄙视自己的灵魂
  description: iOS开发、ReactNative开发、AR
  author: DragonFly
  language: zh-Hans
  ......
  theme: next         // 之前下载的主题名称
```

这儿需要重点注意的是要配置博客提交的github地址。
```
  deploy:
   type: git    //使用Git 发布
   repo: https://github.com/xxx/xxx.github.io.git    // 刚创建的Github仓库
```
另外主题的配置文件也需要做一些修改，相应的文件目录为`xxx.github.io/themes/next/_config.yml`，可配置的选项可以参考[NexT主题配置](http://theme-next.iissnan.com/theme-settings.html)。

##### 新建一篇文章

所有基础框架已经创建完成，接下来你可以开始写第一篇博客了，hexo的命令也很简单，使用两次就记住了。

```
$ hexo new "My-New-Post"  // 这儿是你的文章标题，创建后可以在文档中修改
```
更多细节可以参考： [Writing](https://hexo.io/docs/writing.html)

##### 启用本地服务

写完文章后，可以在本地启用服务，来看你的成果。在浏览器中输入`https://localhost:4000`即可访问你的博客主目录。
```
$ hexo server //hexo s
```

##### 发布内容  

测试没问题后，我们就可以生成静态网页文件并发布到GitHub Pages上了。

生成静态文件的命令：
```
$ hexo generate  //hexo g
```
发布的命令：
```
$ hexo deploy //hexo d
```

以上命令可以简写为`hexo g -d`，这样你的博客已经上传到GitHub了。此时你可以在浏览器里输入fighting300.github.io来访问你的博客。
另外第一次使用时，你需要在终端配置Github的邮箱和密码。

### 域名绑定  

或许你对你的github域名不满意，想要使用自己独立的域名。那么你可以在阿里云旗下的[万网](https://wanwang.aliyun.com/)购买域名(如`http://fighting300.com`)，购买之后进行实名认证。

![域名购买](http://ojca2gwha.bkt.clouddn.com/Hexo-blog-domain.png)

#### 创建CNAME文件
为了绑定域名，首先需要在source文件夹下创建一个CNAME文件，文件内容为你要设置的域名(如fighting300.com)，这样将你的域名指向了Github服务器。用Hexo命令deploy后，几分钟后生效，你会看到你的仓库下GitHub Pages的信息已经发生变化。

![GitHub domain](http://ojca2gwha.bkt.clouddn.com/Hexo-blog-domain-page.png)  

#### 添加DNS解析
阿里云其实有自带的DNS解析服务，但是有小伙伴反馈他的DNS解析服务会有问题(目前使用过程中暂未发现)，可以切换为[DNSPod](https://www.dnspod.cn/)的解析服务。两种DNS的解析配置都很简单，只需要添加对应的IP地址到你的域名配置中即可。该IP地址使用`ping fighting300.github.io`来获取到。

![阿里云DNS配置](http://ojca2gwha.bkt.clouddn.com/Hexo-blog-aliyun.png)

![DNSPod配置](http://ojca2gwha.bkt.clouddn.com/Hexo-blog-dnspod.png)

完成上述步骤后，需要等一段时间上述配置才会生效，最后你的博客已经搭建完成，你可以开启你的博客之旅了。

![My blog](http://ojca2gwha.bkt.clouddn.com/Hexo-blog-review.png)   

本文将持续记录一些hexo使用方法和常见错误的解决方式，如果喜欢，请持续关注本博客。



#### 常见问题
##### 1.首页文章显示部分  
最简单的方案是在文章中添加以下标签内容：  
`<!--more-->`

##### 2.评论系统支持
试了几种评论服务，感觉还是[Disqus](https://disqus.com/)的服务比较好，比较麻烦的一点是需要翻墙才能评论。不过为了良好的体验，目前还是使用Disqus吧，最主要它是免费的。接入很简单，现在[网站注册](https://disqus.com/admin/create/),然后在主题配置文件下填写以下内容即可：

```
# Disqus
disqus:
  enable: true
  shortname: ***** //这儿填写相应的服务名  
  count: true
```

注意使用过程中，要配置Guest Commenting,这样就可以允许所有人在页面评论了。   
注意，在标签需要关闭评论,添加以下标签即可。  
```
  ---
  comments: false
  ---
```

##### 3.统计支持  
目前发现GoogleAnalysis的统计数据非常丰富，翻墙去体验下吧。

##### 4.添加标签

新建一个页面，命名为tags。命令如下：  

```
$ hexo new page "tags"
```

编辑刚新建的页面，将页面的类型设置为 tags ，主题将自动为这个页面显示标签云。页面内容如下：  

```
title: tags
date: 2016-12-22 12:39:04
type: "tags"
```

注意： 如果有启用Disqus评论，默认页面也会带有评论。如果需要关闭，请添加comments字段并将值设置为false，如：

```
comments: false
```

然后在菜单中添加链接，编辑主题配置文件，添加tags到menu中，如下：  

```
menu:
  home: /
  archives: /archives
  tags: /tags
```



##### 参考文档
1. [Hexo](https://hexo.io/)!
2. [NexT](http://theme-next.iissnan.com/getting-started.html)
3. [troubleshooting](https://hexo.io/docs/troubleshooting.html)
4. [GitHub](https://github.com/hexojs/hexo/issues).
