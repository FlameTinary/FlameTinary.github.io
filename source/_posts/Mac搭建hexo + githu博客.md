---
title: Mac搭建hexo + githu博客
date: 2015-09-01 10:30:54
tags:
---

# 相关问题解决
1. 在将hexo源文件提交到git的hexo目录中出现`modified:   themes/next`的问题.

    **原因:** 由于next主题是存放于GitHub中的,因此在将next主题clone下来的时候,同时携带了一些GitHub的相关文件,是由这些脏数据引起的.
    
    **解决办法:** 删除相关文件即可,进入存放hexo的文件夹->themes->next,其中有`git`/`GitHub`/`gitattribute`,三个文件,删除即可.
    
2. git 分支可以commit成功,但是push不到origin hexo.
**原因:** 本地的hexo分支没有关联到origin上的hexo分支.
**解决办法:**使用`git branch --set-upstream hexo origin/hexo`.

3. 如果在修改hexo文件,比如修改next主题后,`hexo s`显示了修改后的效果,但是 `hexo generate`/`hexo deploy`后却在GitHub page上没有效果,可以先 `hexo clean`一下,然后在执行部署`hexo g -d`.

