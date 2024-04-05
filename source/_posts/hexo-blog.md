---
title: 使用Github + Hexo搭建博客
date: 2022-04-05 03:26:42
categories:
- 博客
tags:
- hexo
---

目前CSDN、掘金、博客园、简书等虽然使用方便，而且能被搜索引擎检索到。但是总归是别人的平台，经常会受限，因此个人网站是一个不错的选择。GitHub Pages + Hexo 可以搭建自己的博客，该方式完全免费且非常稳定。
<!--more-->

# 简介

**GitHub Pages 是什么？**

[What is GitHub Pages? - GitHub Help](https://docs.github.com/en/pages/getting-started-with-github-pages/about-github-pages)

GitHub Pages 是由 GitHub 官方提供的一种免费的静态站点托管服务，让我们可以在 GitHub 仓库里托管和发布自己的静态网站页面。

**Hexo 是什么？**

官网：[hexo.io](https://hexo.io/zh-cn/)

Hexo 是一个快速、简洁且高效的静态博客框架，它基于 Node.js 运行，可以将我们撰写的 Markdown 文档解析渲染成静态的 HTML 网页。

**Hexo + GitHub 文章发布原理**

在本地撰写 Markdown 格式文章后，通过 Hexo 解析文档，渲染生成具有主题样式的 HTML 静态网页，再推送到 GitHub 上完成博文的发布。

# 环境搭建

Hexo 基于 Node.js，搭建过程中还需要使用 npm（Node.js 已带） 和 git，因此先搭建本地操作环境，安装 Node.js 和 Git。

- [Node.js](https://nodejs.org/zh-cn)
- [Git](https://git-scm.com/downloads)

**安装 Node.js**

使用CentOS 8 yum源安装的Node.js版本较低，使用hexo d等命令时会报错。所以采用源码编译安装
1、官网下载最新版源码

```
cd /usr/local/src
wget https://nodejs.org/dist/v16.14.2/node-v16.14.2.tar.gz
```

2、安装编译所需要的工具

```
dnf install gcc gcc-c++
```

3、解压缩并编译

```
tar zxvf node-v16.14.2.tar.gz
cd node-v6.10.0
./configure
make && make install
```

**安装 Git**

```
dnf install -y git
```

**检查版本**

```
node -v
git --version
```

# 连接博客

使用邮箱注册 GitHub 账户，选择免费账户（Free），并完成邮件验证。

**设置用户名和邮箱**：

```
git config --global user.name "GitHub 用户名"
git config --global user.email "GitHub 邮箱"
```

**创建 SSH 密匙**

> 输入 ssh-keygen -t rsa -C “GitHub 邮箱”，然后一路回车。

**添加密匙**：

> 进入 `/root/.ssh/` 目录，打开公钥 id_rsa.pub 文件并复制里面的内容。

登陆 GitHub ，进入 Settings 页面，选择左边栏的 SSH and GPG keys，点击 New SSH key。 Title 随便取个名字，粘贴复制的 id_rsa.pub 内容到 Key 中，点击 Add SSH key 完成添加。

**验证连接**：

> 打开 Git Bash，输入 ssh -T [git@github.com](mailto:git@github.com) 出现 “Are you sure……”，输入 yes 回车确认

显示 `Hi xxx! You've successfully……` 即连接成功。

# 创建 Github pages 仓库

GitHub 主页右上角加号 -> New repository：

- Repository name 中输入：用户名.github.io
- 勾选 Add a README file，会自动设置分支（分支名设置成master）：This will set master as the default branch.
- create repository

# 创建保存源码的分支

GitHub Pages 会自动部署静态网页文件，并将 master 分支作为部署的默认分支。为将静态网页和源文件（包含文章、主题等）分离开，强烈建议创建新分支，这样 master 分支只用来发布静态网页，而文档编辑和 Hexo 操作都在另一个分支上完成。

打开博客所在本地的目录，将 git 仓库 clone 至本地：

```
git clone https://github.com/用户名/用户名.github.io.git
cd 命令进入仓库目录,再创建本地分支
# 新建并切换到博客源码分支
git checkout -b hexo
# 查看本地分支
git branch -l
```

# 本地安装 Hexo 博客程序

由于只能在空文件夹中生成 Hexo 项目,所以我们先将 `.git` 以及其他文件(如 `README.MD`)移出去,完成初始化后再移回来.

**安装 Hexo**

```
npm install -g hexo-cli
```

**Hexo 初始化**

```
# 初始化
hexo init
# 安装组件
npm install
```

**本地预览**

```
# 生成页面
hexo g
# 启动预览
hexo s
```

访问 http://localhost:4000， 出现 Hexo 默认页面，本地博客安装成功！

# 部署 Hexo 到 Github Pages

本地博客测试成功后，就是上传到 GitHub 进行部署，使其能够在网络上访问。

首先安装 hexo-deployer-git：

```
npm install hexo-deployer-git --save
```

然后修改 _config.yml 文件末尾的 deploy 部分，修改成如下：

```
deploy:
  type: git
  repo: git@github.com:用户名/用户名.github.io.git
  branch: master
```

执行 `hexo g -d`部署静态页面至 Github Pages.

如果成功,此时通过 https://用户名.github.io/ 会出现 Hexo 默认页面.

# 部署 源文件到 Github Pages

发现 github pages 仓库中没有 hexo 分支,因为没有将本地 `git pull`到 github 上

先查看本地分支和远程仓库分支,发现本地和远程不一致,本地存在我创建的 hexo 分支

```
git branch -a
```

将本地创建的分支 push 到 github 仓库,两个 hexo,一个是本地名,一个是远程仓库里的命名.

```
git push origin hexo:hexo
```

由于有部分是 Hexo 初始化的文件,不需要上传,可以过滤掉.打开 `.gitignore`文件,选择 过滤的文件

```
.DS_Store
Thumbs.db
db.json
*.log
node_modules/
public/
.deploy*/
```

将本地的源文件 push 到 github 仓库.

```
git add .
# 引号内是描述
git commit -m 'hexo source post'
# hexo 是分支名
git push origin hexo-source
```

注意：如果是通过 git clone 下载配置的主题， push 源文件时需要将主题的 `.git`文件夹删除或改名备份。

# 更换主题

在 Themes | Hexo 选择一个喜欢的主题，比如 NexT，进入网站目录打开 Git Bash Here 下载主题：

> 主题链接: https://hexo.io/themes/
>
> 我所用的 NexT主题说明文档: http://theme-next.iissnan.com/getting-started.html

# 发布文章

进入博客所在目录，创建博文：

```
hexo new "My New Post"
```

然后 source 文件夹中会出现一个 My New Post.md 文件，就可以使用 Markdown 编辑器在该文件中撰写文章了。

写完后运行下面代码将文章渲染并部署到 GitHub Pages 上完成发布。以后每次发布文章都是这两条命令。

```
# 生成页面
hexo g
# 部署发布
hexo d
```

也可以不使用命令自己创建 .md 文件，只需在文件开头手动加入如下格式 Front-matter 即可，写完后运行 hexo g 和 hexo d 发布。

```
---
title: Hello World # 标题
date: 2019/3/26 hh:mm:ss # 时间
categories: # 分类
- test
tags: # 标签
- test
---
摘要
<!--more-->
正文
```

# next主题优化

**设置圆角**
在`source/_data/variables.styl`中输入以下代码，注意，`$`并不是多余的
```
// 圆角设置
$border-radius-inner     = 20px 20px 20px 20px;
$border-radius           = 20px;
```
然后在 NexT 的配置文件`_config.next.yml`中取消`variables.styl`的注释:
```
custom_file_path:
  variable: source/_data/variables.styl
```
**去除底部“由 Hexo 强力驱动”**
在`themes/next/_config.yml`，找到`Powered by Hexo & NexT`字段，将powered: true改为powered: false
**设置个人头像**
在`themes/next/_config.yml`中找到如下代码：
```
# Sidebar Avatar
avatar:
  # Replace the default image and set the url here.
  url: /images/头像名.后缀
  # If true, the avatar will be dispalyed in circle.
  rounded: true
  # If true, the avatar will be rotated with the cursor.
  rotated: true
```
- `url`后接头像路径，一般将图片放在`themes\next\source\images`下，并修改`头像名.后缀`即可
- `rounded`即以**圆圈**方式显示头像
- `rotated`即为当鼠标放置在头像上时头像会**旋转**

**添加游客与访问统计**
在`themes/next/_config.yml`修改代码：
```
busuanzi_count:
  enable: true
  total_visitors: true
  total_visitors_icon: fa fa-user
  total_views: true
  total_views_icon: fa fa-eye
  post_views: true
  post_views_icon: far fa-eye
```

**设置打赏**
只需要`themes/next/_config.yml`中填入微信和支付宝收款二维码图片地址,即可开启该功能
```
reward_comment: 坚持原创技术分享，您的支持将鼓励我继续创作！
wechatpay: /path/to/wechat-reward-image
alipay: /path/to/alipay-reward-image
```

**腾讯公益404**
腾讯公益404页面，寻找丢失儿童，让大家一起关注此项公益事业！
使用方法，新建 404.html 页面，放到主题的 `source` 目录下，内容如下：
```
<!DOCTYPE HTML>
<html>
<head>
  <meta http-equiv="content-type" content="text/html;charset=utf-8;"/>
  <meta http-equiv="X-UA-Compatible" content="IE=edge,chrome=1" />
  <meta name="robots" content="all" />
  <meta name="robots" content="index,follow"/>
  <link rel="stylesheet" type="text/css" href="https://qzone.qq.com/gy/404/style/404style.css">
</head>
<body>
  <script type="text/plain" src="http://www.qq.com/404/search_children.js"
          charset="utf-8" homePageUrl="/"
          homePageName="回到我的主页">
  </script>
  <script src="https://qzone.qq.com/gy/404/data.js" charset="utf-8"></script>
  <script src="https://qzone.qq.com/gy/404/page.js" charset="utf-8"></script>
</body>
</html>
```

**添加「标签」页面**
新建「标签」页面，并在菜单中显示「标签」链接。「标签」页面将展示站点的所有标签，若你的所有文章都未包含标签，此页面将是空的。底下代码是一篇包含标签的文章的例子：
```
# 新建页面
$ cd your-hexo-site
$ hexo new page tags
```
编辑刚新建的页面，将页面的`type`设置为`tags`，主题将自动为这个页面显示分类。页面内容如下：
```
title: 标签
date: 2022-04-05 12:39:04
type: "tags"
comments: false
---
```
**注意**：如果有集成评论服务，页面也会带有评论。若需要关闭的话，请添加字段`comments`并将值设置为`false`

**添加「分类」页面**
新建「分类」页面，并在菜单中显示「分类」链接。「分类」页面将展示站点的所有分类，若你的所有文章都未包含分类，此页面将是空的。 底下代码是一篇包含分类的文章的例子：
```
# 新建页面
$ cd your-hexo-site
$ hexo new page categories
```
编辑刚新建的页面，将页面的`type`设置为`categories`，主题将自动为这个页面显示分类。页面内容如下：
```
title: 分类
date: 2014-12-22 12:39:04
type: "categories"
comments: false
---
```

**本地搜索**
安装`hexo-generator-searchdb`，在站点的根目录下执行以下命令：
```
npm install hexo-generator-searchdb --save
```
编辑`_config.yml`，新增以下内容到任意位置：
```
search:
  path: search.xml
  field: post
  format: html
  limit: 10000
```
编辑`themes/next/_config.yml`，启用本地搜索功能：
```
# Local search
local_search:
  enable: true
```

**代码复制**
编辑`themes/next/_config.yml`，启用代码复制功能：
```
# Add copy button on codeblock
copy_button:
  enable: false
  # Available values: default | flat | mac
  style:
```

# 常见命令

```
hexo new "name"       # 新建文章
hexo new page "name"  # 新建页面
hexo g                # 生成页面
hexo d                # 部署
hexo g -d             # 生成页面并部署
hexo s                # 本地预览
hexo clean            # 清除缓存和已生成的静态文件
hexo help             # 帮助
```

# 参考

[使用 Hexo+GitHub 搭建个人免费博客教程（小白向） - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/60578464#:~:text=使用)
[Hexo+Github Page｜基础教程(二)：NexT 主题基本美化｜全网最细致全面的教程 - 少数派 (sspai.com)](https://sspai.com/post/85116#!)
[Hexo 备份和恢复 | 子幽博客 (liukuan.cc)](https://blog.liukuan.cc/Hexo/backup-restore/)
[主题配置 - NexT 使用文档 (iissnan.com)](http://theme-next.iissnan.com/theme-settings.html)
[Hexo+Next主题搭建个人博客+优化全过程（完整详细版） - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/618864711)
