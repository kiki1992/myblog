---
title: SourceTree使用问题--Mac文件拥有者设置
date: 2017-12-13 14:10:32
summary: 本篇记录了小白在使用SourceTree时遇到的一个问题。
tags: [Git,日常记录]
---
# Mac文件拥有者属性设置引发的SourceTree使用问题
小白最近开始用公司给配的mac，以前在Win系统上一直用的TortoiseGit管理Git，可惜人家没有Mac版本。怎么办，上网查查。貌似Mac上SourceTree这款软件用的人挺多，实际用过之后确实感觉方便。可没曾想，刚用没几天就遇到了点问题，提交拉取通通报错，还好英文不算差，照着报错信息算是找到了问题所在。怕以后遇到同样的问题又忘了，这里稍微记录下。

先贴张图看看都报啥错了
![sourceTree报错信息](/myblog/images/sourceTreeErr.png)
主要看这句话：insufficient permission for adding an object to repository database .git/objects
貌似.git目录下的文件权限不够，通过命令行看了下文件属性，发现objects这一项的所有者为root，而我当前登陆的是kiki这个用户，难怪权限不够了。
![.git目录下的文件权限-w200](/myblog/images/macOwn.png)
于是用下面的命令修改拥有者属性，果然就正常了。
```
chown -R kiki object
```
事后想了想，应该是以前用root用户在命令行操作过git导致的这个问题，看来以后在切换用户操作时一定要多多考虑会不会对文件系统造成影响。

参考链接：[chown命令用法](https://www.cnblogs.com/peida/archive/2012/12/04/2800684.html)

