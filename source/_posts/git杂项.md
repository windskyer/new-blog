---
title: git杂项
categories: Linux
sage: false
date: 2019-04-10 15:29:46
tags: git
---

<amp-auto-ads type="adsense" data-ad-client="ca-pub-5216394795966395"></amp-auto-ads>

# gitignore文件的获取与配置

`可以在 GitHub 官网上搜索 gitignore , 有项目维护着各种语言开发所使用的gitingore文件`

<!-- more -->

# gitignore 文件

|示例|解释|
|---|---|
|# 此为注释|表示注释,  将被忽略|
|*.a|* 代表所有, 即忽略所有 .a结尾的文件|
|/todo|仅仅忽略 RootSRC(项目根目录)/todo, 即忽略指定的文件或文件夹(不包括子目录)|
|todo|忽略 todo/ 目录下的所有文件, 包括子目录|
|doc/*.txt|会忽略 doc/ 目录内的所有.txt文件, 但不包括子目录的|

`.gitignore只能忽略那些还没有被track的文件, 如果某些文件已经被纳入了版本管理中, 则修改.gitignore是无效的, 解决方法是先把本地缓存删除(改变成未track状态), 然后再提交`

```bash
git rm -r --cached .
git add . -A
git commit -m 'update .gitignore'
```