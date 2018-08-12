---
title: 在Github上部署Hexo博客
date: 2016-03-03 22:16:18
tags: hexo
---

### 为什么从[jekyll-now][1]换成[hexo][2]
事情得从我在几天前收到的邮件说起，Github提示我Github Pages服务即将升级到Jekyll 3.0，而且到[2016年5月1号将只支持kramdown引擎][3]，所以我按照提示升级之后，发现自己之前发布的文章都出现了#号不能正常渲染的错误，于是我踏上了更换博客框架的漫漫长路。(渲染出错其实是我自己的原因，这是后话~无奈)
<!-- more -->


----------


### Hexo好处都有啥？
经过这几天的体验，发现Hexo比之前的jekyll-now有以下的好处：
 1. 有众多主题可供选择（例如你现在看到的这个）
 2. 插件繁多，通过插件可以在开启自己的server，预览效果，也可以通过命令行一次性部署到远端服务器(Github、Heroku等)
 3. 有草稿机制，可以先写草稿再发布出去
 4. 有源源不断的开发者对Hexo更新
虽然jekyll-now可以做到快速建站，但是在自定义这方面却完全输给了Hexo。


----------


### 安装步骤
想要使用Hexo来搭建自己的博客，首先你要安装[Node.js][4]和[Git][5]，Windows用户建议在官网直接下载安装包进行安装，比较省事。
当你安装完这两个东西之后，打开命令行(cmd)输入以下命令安装hexo在你的电脑上。

> $ npm install -g hexo-cli

稍等片刻，hexo即安装成功。

### 部署到Github
一旦Hexo安装成功，你就可以在命令行中输入以下命令开始初始化你的博客系统。

     $ hexo init <folder>
     $ cd <folder>
     $ npm install
     
其中，<folder>替换成你的想要存放的Hexo博客目录。如d:/hexo。

> 如果遇到cd命令无效的问题，可以加强制跳转命令"/d"，例如：cd /d d:/hexo

> 在执行这些命令之前你最好检查系统环境变量Path是否包含了git的主目录。因为init命令依靠的是git的clone实现，如果git在环境变量中没有配置好，将会以复制文件的方式替代。

> 需要说明的是，如果你是使用git的安装包形式安装的，在安装过程中有选项可以选择自动导入环境变量！

在初始化完成后，你将在<folder>目录看到如下的目录结构：

> .
├── _config.yml
├── package.json
├── scaffolds
├── source
|   ├── _drafts
|   └── _posts
└── themes

其中，_config.yml是主要的配置文件，在里面可以修改站点的信息。

> 最好使用Sublime text之类代码编辑器打开这个文件，而不要使用记事本打开，因为记事本可能会加入某些特殊字符导致修改异常

source文件夹存放的是你用markdown存放的博文或者页面，里面有_post(博文)等其他文件夹，还可能有assets\images文件夹，里面存放公共图片资源等。

theme文件夹则是存放着你的博客主题资源，你可以在[这里][6]找到适合你的主题并应用到你的博客上去！

### _config.yml配置说明
直接用文本编辑器打开_config.yml，建议修改以下变量：

 - title 你博客的主标题
 - subtitle 你博客的副标题
 - description 你博客的描述
 - author 作者名
 - language 使用的语言，简体中文就输入zh-cn
 - per_page 一页展示的文章数量，0表示不分页
 - theme 博客使用的主题，false表示不使用主题
 - deploy 一键部署的属性

### 配置一键部署到github
首先登录github并新建一个新的respository，名字格式为<你的github账户名>.github.io，例如microstudent.github.io，然后复制其SSH形式的clone urls，例如：git@github.com:microstudent/microstudent.github.io.git。

接着运行命令行，输入

    $ npm install hexo-deployer-git --save

安装hexo-deployer-git，hexo将一键部署组件分解成了许多模块，这是部署到git类服务器的模块。

在_config.yml，修改deploy属性如下：

    deploy:
        type: git
        repo: <repository url>
        branch: master

其中，< repository url>替换成刚才复制的clone urls。

然后就是见证奇迹的时刻了。

在命令行中用cd命令进入hexo的目录，输入

    hexo g -d
    
这之中包含了两层命令，第一层是hexo generate，生成博客页面，附带一个-d指的是生成后直接部署（hexo deploy
），然后打开< yourusername>.github.io，如果没有出现error404就说明部署成功！

### 开启本地服务器
你可以在本地开启一个服务器浏览博客。首先命令行进入hexo的目录，然后输入

    $ hexo server

然后打开浏览器输入：http://localhost:4000/ 即为博客的效果。

### 写作
你可以在hexo目录下执行

    $ hexo new <title>

创建新文章，< title>替换为文章标题。
然后在source/_post下找到生成的md文件，在这个文件上即可开始自己的写作！
### 一些可能你会遇到的问题

 1. 错误1：
    fatal: Could not read from remote repository.
    Please make sure you have the correct access rights
    and the repository exists.
除了提示消息本身的原因之外，请检查你安装的Git用的SSH key是否过期或者失效，如果不能确定最好重新生成一个并在github上设置权限。
 2. 错误2：
    deploy deployer not found: github
3.0后的版本，Hexo需要将type设置成git并且使用
    $ npm install hexo-deployer-git --save
安装对应的插件，才能上传。

 3. 错误3：
    deploy deployer not found: git
这个文件很可能是因为你没有正确安装好hexo-deployer-git这个模块，建议卸载后使用  $ npm install hexo-deployer-git --save命令重新安装
 4. 错误4：在部署过程中出现乱码，并且部署失败。
很可能因为你的git配置不正确，建议卸载git/github for windows后重新安装最新版本，而且最好不要同时安装git和github for windows。
 5. 错误5：在部署过程中异常报错。
请检查你的_config.yml文件是否填写正确，在每个变量后的冒号都要有一个空格，否则会识别错误！同时据说在里面输入中文字符也可能会导致这个问题，不过我尝试了之后并没有出现这个问题，这很可能是编码的问题，使用UTF编码应该就没问题了。


### 还有问题？ 
如果你觉得介绍不够详细，或者是还有很多疑问没有解决，那么你可以登录hexo官网查看相关文档https://hexo.io/docs/  。部分页面已经有简中版了！

### 关于渲染错误的问题
为什么升级了markdown引擎后网页显示不正确了呢？
其实是因为语法有差异。
新的markdown引擎要求在标题符号#后加一个空格，否则就不能正常识别成一个标题，o(╯□╰)o。

  [1]: https://github.com/barryclark/jekyll-now
  [2]: https://hexo.io
  [3]: https://github.com/blog/2100-github-pages-now-faster-and-simpler-with-jekyll-3-0
  [4]: https://nodejs.org/en/
  [5]: http://git-scm.com/
  [6]: https://hexo.io/themes/