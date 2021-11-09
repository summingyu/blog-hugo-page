---
title: "给gitlab仓库添加pre-receive限制提交"
subtitle: ""
date: 2021-11-09T15:56:46+08:00
lastmod: 2021-11-09T15:56:46+08:00
authors: ["summingyu"]
description: "使用gitlab的pre-receive进行提交限制,防止错误提交"

tags: [git,hooks, flyway]
categories: ["标准化"]
series: []

hiddenFromHomePage: false
hiddenFromSearch: false

featuredImage: ""
featuredImagePreview: ""

toc:
  enable: true
math:
  enable: false
lightgallery: false
license: ""
---

<!--more-->

## 背景

由于公司使用了`flyway`来进行mysql语句的版本管理,但是flyway对提交的目录格式存在格式的限定.
当一些开发同学有不规范提交时,容易导致执行flyway检查版本号获取有问题,从而影响sql执行的顺序.
所以需要借助`gitlab`的`pre-receive`功能,进行提交代码的预检查.

## 单个项目配置

### 1. 在gitlab服务器上的项目仓库新建目录

```bash
cd /var/opt/gitlab/git-data/repositories/SOME_USER/SOME_REPO.git
mkdir -p custom_hooks/pre-receive.d/
```

### 2. 上传`flyway-format-check.sh`脚本

将脚本放置到新建目录
`/var/opt/gitlab/git-data/repositories/SOME_USER/SOME_REPO.git/custom_hooks/pre-receive.d`内

```bash
cat flyway-format-check.sh < EOF
#!/usr/bin/env bash

#
# Pre-receive hook that will block any new commits that contain files ending
# with .gz, .zip or .tgz
#
# More details on pre-receive hooks and how to apply them can be found on
# https://help.github.com/enterprise/admin/guides/developer-workflow/managing-pre-receive-hooks-on-the-github-enterprise-appliance/
#

zero_commit="0000000000000000000000000000000000000000"

# Do not traverse over commits that are already in the repository
# (e.g. in a different branch)
# This prevents funny errors if pre-receive hooks got enabled after some
# commits got already in and then somebody tries to create a new branch
# If this is unwanted behavior, just set the variable to empty
excludeExisting="--not --all"
has_errors=0
exist_err_files=()
err_files=()

while read oldrev newrev refname; do
  # echo "payload"
  echo $refname $oldrev $newrev

  # branch or tag get deleted
  if [ "$newrev" = "$zero_commit" ]; then
    continue
  fi

  # Check for new branch or tag
  if [ "$oldrev" = "$zero_commit" ]; then
    span=`git rev-list $newrev $excludeExisting`
  else
    span=`git rev-list $oldrev..$newrev $excludeExisting`
  fi

  
  for COMMIT in $span;
  do
    for FILE  in `git log -1 --name-only --pretty=format:'' $COMMIT |grep '.sql$'`;
    do
        filter_file=`echo $FILE |grep -oP '[vV](\d+_)+[^\d_].*?($|/)'`
        echo $FILE
        if [ -n "$filter_file" ];then
            has_errors+=1
            exist_err_files+=("$FILE")
            err_files+=("$filter_file")
        fi
    done
  done
done
if [ "$has_errors" != 0 ];then
    echo "exist flyway format error file: $exist_err_files"
    echo 
    echo "error file: $err_files"
    exit 1
fi
exit 0

EOF
```

### 3. 修改权限

```shell
cd /var/opt/gitlab/git-data/repositories/SOME_REPO/SOME_REPO.git
chown -R git custom_hooks/
chmod +x custom_hooks/pre-receive.d/flyway-format-check.sh
```

## gitlab全局设置pre-receive

在全局配置文件`/etc/gitlab/gitlab.rb`配置文件中修改或者查找配置

```ruby
gitlab_shell['custom_hooks_dir'] = "/opt/gitlab/embedded/service/gitlab-shell/hooks"
```

然后如单个配置一样,新建pre-receive.d目录,放脚本

## 主体逻辑

```bash
  
  for COMMIT in $span;
  do
    # 逐个检查提交的.sql文件
    for FILE  in `git log -1 --name-only --pretty=format:'' $COMMIT |grep '.sql$'`;
    do
        # 过滤的文件为以v或V开头后面接(数字_)循环,再接不是数字或_的文件或目录
        # v8_88_test/ V2021_8_test.sql
        filter_file=`echo $FILE |grep -oP '[vV](\d+_)+[^\d_].*?($|/)'`
        echo $FILE
        if [ -n "$filter_file" ];then
            has_errors+=1
            exist_err_files+=("$FILE")
            err_files+=("$filter_file")
        fi
    done
  done
done
if [ "$has_errors" != 0 ];then
    echo "exist flyway format error file: $exist_err_files"
    echo 
    echo "error file: $err_files"
    exit 1
fi
exit 0
```

## 出现提交错误的解决方法

由于是在执行git push之后检查的,所以会在本地存在一个提交版本的记录,而服务端没有记录
就算改正确再提交,也是相当于两个提交记录,第一个错误记录还是存在的,
所以需要先回退提交

```bash
# 查找提交的commitid
git log
# 如果是最后的提交,可以用HEAD^代替
# 软回退,只撤销暂存区的提交记录,不影响实际更改
git reset --soft HEAD^
```
