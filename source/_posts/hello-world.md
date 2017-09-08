---
title: Hello World 之 Hexo的使用
date: 2017-03-28 01:16:11

---

Hexo是一个快速、简洁且高效的博客框架。Hexo使用Markdown（或其他渲染引擎）解析文章，在几秒内，即可利用靓丽的主题生成静态网页。
本文将持续记录一些hexo使用方法和常见错误的解决方式。

## Quick Start

安装Hexo前，首先需要在电脑上安装git和Node.js。

Node安装：
最佳方式是使用nvm
```
$ curl https://raw.github.com/creationix/nvm/master/install.sh | sh
```

安装完nvm后，执行一下命令安装Node
```
$ nvm install stable
```

Hexo安装：  
```
$ npm install -g hexo-cli
```

### 建站

安装 Hexo 完成后执行下列命令，Hexo将会在指定文件夹中新建所需要的文件。
```
$ hexo init <folder>
$ cd <folder>
$ npm install
```

### 新建一片文章

``` bash
$ hexo new "My New Post"
```

More info: [Writing](https://hexo.io/docs/writing.html)

### Run server

``` bash
$ hexo server //hexo s
```

<!--more-->
More info: [Server](https://hexo.io/docs/server.html)

### Generate static files

``` bash
$ hexo generate  //hexo g
```

More info: [Generating](https://hexo.io/docs/generating.html)

### Deploy to remote sites

``` bash
$ hexo deploy //hexo d
```

More info: [Deployment](https://hexo.io/docs/deployment.html)


### 添加标签

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
2. [documentation](https://hexo.io/docs/)
3. [troubleshooting](https://hexo.io/docs/troubleshooting.html)
4. [GitHub](https://github.com/hexojs/hexo/issues).
