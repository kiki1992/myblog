---
title: SourceTree使用问题--Mac文件拥有者设置
date: 2017-12-13 14:10:32
summary: 本篇记录了小白在使用SourceTree时遇到的一个问题。
tags: [Git,日常记录]
---
# Mac文件拥有者属性设置引发的SourceTree使用问题
小白最近开始用公司给配的mac，以前在Win系统上一直用的TortoiseGit管理Git，可惜人家没有mac版本。怎么办，百度呗。于是便用上了SourceTree这款软件，别说，还挺好用，图形界面操作真是方便。可没曾想，刚用没几天就遇到了点问题，提交拉取通通报错，没办法，解决呗。于是便有了这篇博客，主要怕以后又忘了呗，稍微记录下。

先贴张图看看都报啥错了
![sourceTree报错信息](/myblog/images/sourceTreeErr.png)
主要看这句话：insufficient permission for adding an object to repository database .git/objects
貌似.git目录下的文件权限不够，通过命令行看看文件权限
![.git目录下的文件权限](/images/macOwn.png)

