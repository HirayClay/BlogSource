---
title: Hexo+github搭建个人博客
date: 2017-09-05 14:04:10
tags:
    - hexo
    - 博客
---
很早之前就有人用hexo和github提供的page服务做个人博客了,不过了解一下就没有怎么关注了，最近有时间，看了一下官方文档，花了两个多小时，搭建了一个简单的个人博客，除了最开始搭建配置繁琐一点，后面写完一篇文章一个命令就发布，体验非常棒！
### 新建主页仓库
登录自己的github账户，新建一个仓库，比如我的用户名是HirayClay，那么我就新建一个名为
HirayClay.github.io的仓库

### hexo环境搭建

首先要安装必要的软件，[Node.js](https://nodejs.org)和[Git](https://git-scm.com/),安装完成之后安装hexo
```shell
   $ npm install -g hexo-cli
```
我安装的是 3.3.3版本
hexo安装好之后就可以用hexo命令创建一个站点了
```shell
    $ hexo init <folder_name>
    $ cd <folder>
    $ npm install
```
看一下创建的目录结构
```
    .
    ├── _config.yml
    ├── package.json
    ├── scaffolds
    ├── source
    |   └── _posts
    └── themes
```

其中_config.yml是配置文件，一些全局的重要配置都在这里面；package.json文件中声明了版本信息和依赖信息，scaffolds，即脚手架的意思，我们创建post的时候就是用的这个文件下的模板，里面默认有三种模板：draft、page、post，当然你也可以创建自己的模板；source目录下有个子目录_posts，顾名思义就是放我们文章的地方；最后themes就是存放主题的地方，可以下载三方的主题放在里面

我们需要重点关注一下_config.yml文件里面几个地方

```yaml
    title: Blog
    subtitle: hirayclay's blog
    description:
    author: hirayclay
    language: zh
    timezone: Asia/Shanghai
```
title：站点的标题，subtitle：站点子标题，description：站点描述 language:站点语言，这里配置的是中文，其他语言的参考这里[>>](https://en.wikipedia.org/wiki/List_of_ISO_639-1_codes),timezone即时区，这里用的中国北京时区，其他时区参考[>>](https://en.wikipedia.org/wiki/List_of_tz_database_time_zones)

要注意一点的是，这里所有的配置“：”后面都需要有一个空格，不然最后解析生成时候会失败，这是YAML的语法

```yaml
    new_post_name: :title
    default_layout: post
```

new_post_name 即生成的post的名称，有一下几个配置：
:title  
:year   
:month  
:i_month    
:day    
:i_day 
我这里直接用title命名生成的post文件

比如执行一下命令
```shell
    hexo new post "MyNewPost" 
```

就会在source/_posts目录下生成 MyNewPost.md文件

再看下语法高亮配置，比较简单，常用到的是否禁用和代码行数

```yaml
    highlight:
    enable: false
    line_number: true
    auto_detect: false
    tab_replace:
```

部署配置，第一步新建仓库的作用到了
```yaml
  deploy:
  type: git
  repo: git@github.com:HirayClay/HirayClay.github.io.git
  branch: master
```

因为我们用的git提交到远程仓库的，所以type = git,以及仓库地址和分支名，这里的repo就是之前新建的仓库地址


配置说完了，基本可以开写了

首先 用 hexo new post <your_post_name> 创建一篇博客，然后source/_posts目录找到对应的博客打开编辑即可,可以给博客加tag，比如本博客的tag

```yaml
   ---
    title: Hexo+github搭建个人博客
    date: 2017-09-05 14:04:10
    tags:
        - hexo
        - 博客
    ---
```

至此基本的都配置完成了，用以下命令生成静态资源
```shell
    hexo generate
```
也可以写成
```shell
    hexo g
```

然后可以本地起一个服务进行预览
```shell
    hexo server
```

浏览器输入http://localhost:4000 进行查看


用以下命令发布到git远程仓库
```shell
    hexo deploy
```

应该会弹出一个窗口让你你输入ssh-key的密码


最后如果觉得默认主题不合适，可以去下载其他[主题](https://hexo.io/themes/)到 themes目录下，可以随意命名该目录下的主题文件夹，但是最后在_config.yml文件中配置主题时候一定要用文件夹的名字
```yaml
    theme: theme_folder_name
```

The End