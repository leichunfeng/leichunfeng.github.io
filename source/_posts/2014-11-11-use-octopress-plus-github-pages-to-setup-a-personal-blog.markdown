---
layout: post
title: "使用 Octopress+GitHub Pages 搭建个人博客"
date: 2014-11-11 21:47:07 +0800
comments: true
categories: 
keywords: Octopress, GitHub Pages, GitHub, Pages, 搭建, 个人博客, 博客
---

序言中提到周末折腾了两天时间，终于用 Octopress 成功搭建起了自己专属的技术博客，那么本文就来细说一下如何使用 Octopress+GitHub Pages 搭建个人博客。

## Octopress

[Octopress](http://octopress.org/) 是一款基于 [Jekyll](https://github.com/jekyll/jekyll) 的功能强大的博客框架，是对 Jekyll 的进一步封装，号称是给黑客使用的，用起来比 Jekyll 简单很多，不需要太多的设置，只需要 [clone 或者 fork Octopress](https://github.com/imathis/octopress)，安装依赖和主题就可以了。当然，这个时候你的博客还比较粗糙，如果你想让你的博客访问速度更快，有分享和评论功能，以及每篇博文的最后都带上原文链接的话，你需要做的还比较多，不过不用担心，Follow me，我会为你娓娓道来。

## GitHub Pages

[GitHub Pages](https://pages.github.com/) 是 GitHub 推出的用于建立用户、组织和项目站点的工具，每个 GitHub 账号和组织都可以建立一个站点以及无数个项目站点。我们要建立个人博客自然使用的是 GitHub Pages 的用户站点功能，建立用户站点非常简单，稍后我们再展开来谈。

## 安装 Octopress

前面我们提到了安装 Octopress 只需要简单的几个步骤就可以了，下面我们一步步来。

### 准备工作

1. 安装 [Git](http://git-scm.com/) 。
2. 安装 Ruby 1.9.3 及以上版本。

你可以使用 `ruby --version` 查看一下你安装的 `Ruby` 版本，如果低于 1.9.3，你可以使用 [rbenv](http://octopress.org/docs/setup/rbenv/) 或 [RVM](http://octopress.org/docs/setup/rvm/) 来安装更高版本。

### 克隆 Octopress

``` ruby
git clone git://github.com/imathis/octopress.git octopress
cd octopress
```

### 安装依赖

``` ruby
gem install bundler
rbenv rehash # 如果你使用的是 rbenv 的话，执行下这句，不是的话可以略过
bundle install
```

### 安装 Octopress 默认主题

``` ruby
rake install
```

### 配置 Octopress

至此，你的 Octopress 就已经安装好了，接下来我们对 Octopress 进行一些简单的配置。我们需要修改的只有 `_config.yml` 一个文件，这个文件包含 `Main Configs` 、`Jekyll & Plugins` 和 `3rd Party Settings` 三个部分。在这里，我们只需要修改 `Main Configs` 中的 `title` 、`subtitle` 和 `author` 。说明，后面我们会通过 Octopress 提供的 `rake setup_github_pages` 命令自动修改 `url` ，这里先不用管。

``` ruby
title: 雷纯锋的技术博客
subtitle: 记录自己学习的点滴，收获分享知识的乐趣
author: 雷纯锋
```

## 写博文的方法

用 Octopress 写博文主要是通过执行 Octopress 提供的 `rake` 命令来完成的，下面简单介绍一下，更多的详细信息可以查看 Octopress 官方文档中的 [Blogging Basics](http://octopress.org/docs/blogging/) 。

``` ruby
rake new_post["title"] # 在 source/_posts 目录下创建一篇新博文
rake generate          # 生成博文到 public 目录下
rake watch             # 查看 source 和 sass 目录的变化，且有变化时重新生成博文
rake preview           # 在 http://localhost:4000/ 预览博文
```

Octopress 博文采用 [Markdown](http://daringfireball.net/projects/markdown/) 语法进行书写，Markdown 的语法全由一些符号所组成，这些符号经过精挑细选，它的作用一目了然，因此你可能只需要 5-10 分钟就能快速上手。Markdown 免费编辑器非常多，我本人非常酷爱 Sublime Text ，它绝对可以称得上编辑器中的神器。所以我的博文都是使用 [Sublime Text 3](http://www.sublimetext.com/3)+[MarkdownEditing](https://github.com/SublimeText-Markdown/MarkdownEditing) 插件进行书写的，书写起来感觉非常惬意。这感觉美妙极了，很难用言语来表达，是我梦寐以求的书写感受，所以也**强烈推荐**你试一试，Who with who knows 。

## 发布 Octopress 到 GitHub Pages

首先，你需要创建一个新的 [GitHub 仓库](https://github.com/repositories/new) ，这个仓库的名称为 `username.github.io` ，其中 `username` 为你的 GitHub 用户名，例如 `leichunfeng.github.io` 。接下来，我们使用 Octopress 提供的 `rake setup_github_pages` 命令来完成一些与 GitHub Pages 相关的配置工作，其中就包括我们前面提到的 `_config.yml` 文件中的 `url` 。

``` ruby
rake setup_github_pages
```

根据提示需要我们输入 GitHub 仓库的 URL ，拷贝我们刚刚创建的仓库的 SSH 或 HTTPS URL ，例如 `git@github.com:leichunfeng/leichunfeng.github.io.git` ，粘贴并回车。这个看似简单的 `rake` 命令其实偷偷地为我们做了很多复杂的工作，包括修改 octopress 远程仓库 origin 的名称为 octopress 、添加我们的 GitHub 仓库为新的 origin 远程仓库和配置 `_config.yml` 文件中的 `url` 等等，更多的详细信息可以查看 Octopress 官方文档中的 [Deploying to GitHub Pages](http://octopress.org/docs/deploying/github/) 。

接下来，运行以下命令。

``` ruby
rake generate
rake deploy # 发布博文到 GitHub Pages
```

最后，别忘记提交你的博客源文件。

``` ruby
git add .
git commit -m 'your message'
git push origin source
```

在这里，我还是想大概谈一谈 Octopress 的工作原理，不然你可能也会跟我刚接触 Octopress 时一样充满疑惑。Octopress 其实为我们建立了两个分支，一个是 `source` 分支，就像我们的书桌，用于存放我们书写时需要用到的各种工具，包括原始的 markdown 文件、生成博文用的插件、主题和脚本等等。另一个是 `master` 分支，其实就是 `public` 目录中的内容，用于存放最终生成的 `HTML` 博文。当我们执行 `rake generate` 命令时，Octopress 会为我们生成 `HTML` 博文到 `public` 目录下。当执行 `rake deploy` 命令时，Octopress 则会将 `public` 目录中的内容提交并同步到 `GitHub Pages` 。建议你自己亲自对比一下运行以上命令前后 octopress 目录中文件的变化，这样可以快速地熟悉 Octopress 的工作原理。

到这里，你的个人博客就已经搭建成功了，你可以使用 `http://username.github.io` 来访问你的博客了，给自己一点掌声吧！不过如果你希望你的博客能够更加与众不同的话，那么我们还有一些事情需要去做，休息一会，我们继续。

### 自定义域名

如果你也跟我一样有自己的域名的话，可以使用自己的域名指向你的博客地址。首先，你需要在 `source` 目录下建立一个名为 `CNAME` 的文件，这个文件只需要一行内容，就是你自己的域名。不过需要特别注意的一点就是这个域名不要带上 `http(s)://` 网址前缀，例如我的域名 `blog.leichunfeng.com` 。最后，运行 `rake` 命令重新生成并发布博文。

``` ruby
echo 'your-domain.com' >> source/CNAME
# 或者
echo 'blog.your-domain.com' >> source/CNAME

rake generate
rake deploy
```

接下来，你需要登陆你的域名注册商的管理平台，在你的域名下新增一条解析记录。如果你想使用子域名，例如 `blog.example.com` ，你需要新增的就是一条记录类型为 `CNAME` ，记录值为 `username.github.io` 的解析记录。如果你想使用顶级域名，例如 `example.com` ，那么你需要新增的就是一条记录类型为 `A` ，记录值为 `192.30.252.153` or `192.30.252.154` 的解析记录。大概过 1 个小时左右，你就可以使用你自己的域名来访问你的博客了。怎么样，这种感觉是不是非常美妙？

## 高级配置

如果你希望你的博客具有微博分享和评论功能的话，可以参考下`唐巧`的博文 [《象写程序一样写博客：搭建基于github的博客》](http://blog.devtang.com/blog/2012/02/10/setup-blog-based-on-github/) 中的`高级配置`章节，这里不再赘述。不过，我建议你最好在注册了[友言](http://www.uyan.cc/)或 [JiaThis](http://www.jiathis.com/) 的账号后再点击`获取代码`，这样获取到的代码就会带有你的 `uid` ，那么你就可以通过友言或 JiaThis 的管理后台查看你的统计数据了。

### 优化博客的访问速度

1. 删除或注释配置 `_config.yml` 文件中有关 `Twitter` 的部分。
2. 修改 `source/_includes/custom/head.html` 文件，删除 `google` 的自定义字体。**注意**，如果使用注释的方式会造成最终生成出来的 `HTML` 页面的 `body`
部分也被注释。
3. 修改 `source/_includes/head.html` 文件中 `jquery.min.js` 的链接地址。
``` html
-  <script src="//ajax.googleapis.com/ajax/libs/jquery/1.9.1/jquery.min.js"></script>
+  <script src="http://cdn.staticfile.org/jquery/1.9.1/jquery.min.js"></script>
```

### 添加原文链接

给你的每篇博文最后添加上原文链接，这样不管是其他人转载或是分享你的博文，读者都能够根据原文链接轻松地访问到你的原始博文。要实现这个目的并不难，但是我不想因为这样而破坏了博文原本的结构和布局，还要让读者在阅读博文的时候毫无违和感，谁让我是一个偏执狂呢！我查看过不少这方面的博文，其中[《为octopress每篇文章添加一个文章信息》](http://codemacro.com/2012/07/26/post-footer-plugin-for-octopress/)还算比较接近我的需求，但还不够完美，且太复杂。于是我决定自己动手实现，结果已如你所见。

- 创建 `source/_includes/post/original_link.html` 文件，内容只有一行代码。
``` ruby
echo '<br>原文链接：<a href="http://blog.leichunfeng.com{#{page.url}#}">http://blog.leichunfeng.com{#{page.url}#}</a>' >> source/_includes/post/original_link.html
```
- 修改 `source/_layouts/post.html` 文件，在 `{#% include post/categories.html %#}` 后面添加一行代码。
``` ruby
{#% include post/categories.html %#}
{#% include post/original_link.html %#} <!-- 添加这一行代码 -->
```

**注意**，请记得将第一步中的 `blog.leichunfeng.com` 替换成你自己的博客域名，将上述两步中的 `#` 号去掉。

到这里，本文就已经接近尾声了，不过我们对博客访问速度的追求仍然没有结束。因为 GitHub 毕竟是国外的（你懂的）托管网站，所以我们博客的访问速度始终还是比较慢的。在下一篇博文中我将向大家介绍如何《将 Octopress 博客从 GitHub 迁移到 GitCafe》，到时你的博客的访问速度也会跟我的一样，飞一般的感觉。
