---
title: "利用pre-receive进行sql检查"
subtitle: ""
date: 2021-11-16T15:56:46+08:00
lastmod: 2021-11-16T15:56:46+08:00
draft: false
authors: ["summingyu"]
description: "利用pre-receive进行sql语句检查,禁止不合规语句的提交"

tags: [git,hooks, flyway,sql]
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

上一章 `{{<link "gitlab-pre-receive" "给gitlab仓库添加pre-receive限制提交">}}` 利用pre-receive校验flyway目录格式的功能反响不错,公司提出更近一步需求,能否对提交的sql进行语句检查,阻止一些不规范的语句提交比如

1. 存在存储过程的关键字`create procedure`
2. `create table` 语句内没有`PRIMARY KEY`
3. 存在对表的重命名语句``rename table`
4. 存在使用`SET @variable = value` 语句

> 还是利用上一章的flyway-format-check.sh的脚本,在这上面进行优化

## 主体逻辑

```bash
# Colors
PURPLE='\033[35m'
RED='\033[31m'
RED_BOLD='\033[1;31m'
YELLOW='\033[33m'
YELLOW_BOLD='\033[1;33m'
GREEN='\033[32m'
GREEN_BOLD='\033[1;32m'
BLUE='\033[34m'
BLUE_BOLD='\033[1;34m'
COLOR_END='\033[0m'
  # Check for new branch or tag
  if [ "$oldrev" = "$zero_commit" ]; then
    span=`git rev-list $newrev $excludeExisting`
  else
    span=`git rev-list $oldrev..$newrev $excludeExisting`
  fi


  for COMMIT in $span;
  do
    # 检查所有有变化的文件db-migration/里面的sql文件
    for FILE  in `git log -1 --name-only --pretty=format:'' $COMMIT |grep -P 'db-migration/.*\.sql$'`;
    do
       # 检查目录或sql文件的备注少一个下划线的情况
        filter_file=`echo $FILE |grep -oP '/[vV](\d+_)+[^\d_].*?($|\/)'`
        if [ -n "$filter_file" ];then
            has_errors+=1
            echo "exist flyway format error file: ${FILE}"
            echo 
            echo -e "error file: ${RED} ${filter_file} ${COLOR_END}"
        fi
        # check sql rule
        # 检查是否存在存储过程
        sql_content=`git show ${newrev}:${FILE}|grep -vP '^\-\- '|grep -inoP '^\s*create +procedure'`
        if [ -n "$sql_content" ];then
            has_errors+=1
            echo -e "error file: ${RED} ${FILE} ${COLOR_END}"
            echo -e "ERROR TYPE: find procedure: ${RED} ${sql_content} ${COLOR_END}"
        fi
        # 检查建表语句是否存在主键
        # 由于注释中存在';',导致存在匹配不正常,去除建表语句后面注释
        no_comment_sql_content=`git show ${newrev}:${FILE}|grep -vP '^\-\- '|sed 's/COMMENT.*,$/COMMENT,/g'`
        sql_content=`echo ${no_comment_sql_content}|grep -vP '^\-\- '|grep -ioaznP 'create table(?![^;]*PRIMARY KEY)[^;]+;'`
        if [ -n "$sql_content" ];then
            has_errors+=1
            echo -e "error file: ${RED} ${FILE} ${COLOR_END}"
            echo -e "ERROR TYPE: not find pkey in create table: ${RED} ${sql_content} ${COLOR_END}"
        fi
        # 检查是否存在rename table
        sql_content=`git show ${newrev}:${FILE}|grep -vP '^\-\- '|grep -ioaznP '^\s*rename +table .*to +'`
        if [ -n "$sql_content" ];then
            has_errors+=1
            echo -e "error file: ${RED} ${FILE} ${COLOR_END}"
            echo -e "ERROR TYPE:  find rename table: ${RED} ${sql_content} ${COLOR_END}"
        fi
        # 检查是否存在set 变量
        sql_content=`git show ${newrev}:${FILE}|grep -vP '^\-\- '|grep -ioaznP '^\s*set +\@.*=.*?;'`
        if [ -n "$sql_content" ];then
            has_errors+=1
            echo -e "error file: ${RED} ${FILE} ${COLOR_END}"
            echo -e "ERROR TYPE:  find set variable: ${RED} ${sql_content} ${COLOR_END}"
            echo -e "you can use ${GREEN}SELET @variable := value ${COLOR_END}"
        fi
    done
  done
done
if [ "$has_errors" != 0 ];then
    echo -e "find errors: ${RED} ${has_errors} ${COLOR_END}"
    echo -e "尝试使用命令:${GREEN}git reset --soft HEAD^${COLOR_END}撤销提交记录,"
    echo -e "如果同时提交多个,则用${GREEN}git log${COLOR_END}查询错误提交的commitid,"
    echo "代替上一个命令的HEAD^"
    exit 1
fi
exit 0
```
