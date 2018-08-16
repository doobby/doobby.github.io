---
title: Github Page 搭建小记
tags: web
author: larryzju
catalog: true
---

# 概述

[Github Pages](https://pages.github.com/) 可以将 github 仓库中的内容作为静态网页提供服务，很适合作为项目主页。
此外，gitpage 还集成了 jekyll 工具，在每次提交时将 jekyll 类型的仓库生成网页，适合作为博客。

本文记录如何搭建 https://doobby.github.io 开发环境，日常维护的一些基本操作

# 本地 jekyll 环境搭建

参考 [Setting up your GitHub Pages site locally with Jekyll](https://help.github.com/articles/setting-up-your-github-pages-site-locally-with-jekyll/)

基本步骤如下

1. 安装 ruby 及 bundler（假设 ruby 已经安装好）

   `gem install bundler`
   
2. 克隆 github page 仓库

   `git clone https://github.com/doobby/doobby.github.io`
   
3. 使用 bundler 安装相应版本的 jekyll（Gemfile 已经在仓库里）

   `bundle install`
   
4. 运行 jekyll，在 http://localhost:4000 页面实时查看修改

   `bundle exec jekyll serve`
   
   
# 仓库目录结构

仓库 fork 自 Huxpro/huxpro.github.io。
整体目录结构符合 [jekyll 目录结构](https://jekyllrb.com/docs/structure/)

其中几个关键目录结构如下所示：

| 目录/文件   | 说明                                 |
|-------------|--------------------------------------|
| Gemfile     | bundler jekyll 依赖配置              |
| _config.yml | 全局配置                             |
| index.html  | 主页面                               |
| _layout/    | 页面布局                             |
| _include/   | 局部页面实现                         |
| _site/      | 编辑生成的静态页面（不要提交到 git） |
| _posts/     | 博客文档                             |

其中 html 或 markdown 文件均可以 YAML 配置开头（元数据，称为 [Front Matter](https://jekyllrb.com/docs/frontmatter/) ），
会在生成页面时作为元数据，或展开为具体的 layout


# Front Matter

Post 文档的开头可以加上 YAML 的元数据，常用的一些如下所示

| key      | description              |
|----------|--------------------------|
| layout   | 布局样式，用 post        |
| title    | 标题                     |
| subtitle | 子标题                   |
| author   | 作者                     |
| date     | 编写日期                 |
| catalog  | true/false：是否生成大纲 |
| tags     | 空格分隔的 TAG 名称      |
