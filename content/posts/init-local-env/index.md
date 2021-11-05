---
title: "本地环境搭建"
date: 2021-10-26T09:09:38+08:00
lastmod: 2021-11-04T09:09:38+08:00
authors: ["summingyu"]
description: "博客构建第一步: 本地环境搭建"

tags: ["hugo", "博客", "DoIt"]
categories: ["文档"]
series: ["start-hugo"]
series_weight: 1
seriesNavigation: true

lightgallery: true

featuredImage: "my_hugo_blog.jpg"
featuredImagePreview: ""
draft: false
toc: true
---
<!--more-->
## 0x00 前言

之前使用hexo+github.io搭建的博客由于各种原因(懒)断掉了,最近听[xuwu](https://xwlearn.com)大佬介绍.
使用基于hugo+github aciton + coding的模式搭建微博,可以实现随时随地编辑发布.
于是就重新拾起博客,持续学习,记录美好生活

## 0x01 博客发布流程

1. hugo: 用于形成框架及搭建初期进行配置调整
2. github: 将hugo生成的博客目录整体上传到仓库中,并使用`github aciton`触发hugo的静态文件生成,并push到coding仓库
3. coding: 仓库存放的是hugo生成的静态文件,并使用coding的静态网站托管功能,将静态文件push到cos中,绑定域名后可进行访问

{{< mermaid >}}flowchart LR;
subgraph coding
    direction TB
    C[coding repo]
    D[托管网站]
    C --> D
end
subgraph github
    direction TB
    A[github repo]
    B[github aciton]
    A -->|trigger| B
end

hugo -->|push| A
B -->|push| C
{{< /mermaid >}}

## 0x02 使用hugo搭建框架及配置调整

### 0o01. 安装hugo

[hugo中文文档](https://www.gohugo.org/)
[hugo英文文档](https://gohugo.io/)

- macOS使用`homebrew`安装

```bash
brew install hugo
```

### 0o02. 网站初始化及安装主题

执行命令

```bash
# path_to_site 为你要创建的博客名称目录,例如:/home/summing/summinglearn.cn
hugo new site ${path_to_site}

cd ${path_to_site}
git init

git submodule add https://github.com/HEIGE-PCloud/DoIt.git themes/DoIt
# 复制主题的案例站点配置文件到自己的站点
cp -a themes/DoIt/exampleSite/config.toml config.toml
```

### 0o03. 创建第一篇博客

```bash
hugo new posts/hello_hugo.md
```

可以发现在content/posts目录里面生成了一个`hello_hugo.md`文档
> 生成的博客都是基于content目录的

打开编辑并输入

```markdown
# hello hugo
```

### 0o04. 本地预览

```bash
hugo server -D
```

> -D 参数表示的是允许渲染draft为true的博客

执行完后,就可以用浏览器访问 `http://localhost:1313`进行预览
