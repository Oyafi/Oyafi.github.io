---
title: 如何解决repo嵌套问题
date: 2026-04-02 15:11:43
Categories: 技术
tags: [git, github]
---

### 如何解决git 误上传导致的子仓库嵌套问题

在编写自己的项目时，外面有时候需要使用别人仓库中的内容，于是习惯性的git clone 了别人的repo ，然后 add commit ，然后 push，于是乎我们的repo里莫名就多了一个子repo。

这是由于我们并没有删除clone下来的项目中的git信息，导致git将下载内容识别为了一个submodule，于是造成了一系列麻烦的后果。

#### 怎么解决呢

首先回退本地提交信息

` git reset --soft HEAD^`

然后手动删除我们子仓库中的.git .github .gitignore文件

然后移除仓库的关联信息

`git rm --cached "子仓库名"`

最后重新commit，然后强制覆盖我们的远程分支

~~~shell
git commit -m "delete submodule"
git push -f origin main
~~~

这样在远程仓库中的子模块就会显示为一个正常的文件，而不是子repo了
