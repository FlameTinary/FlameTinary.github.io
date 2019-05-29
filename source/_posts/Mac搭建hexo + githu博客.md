---
title: Mac搭建hexo + github博客
date: 2015-09-01 10:30:54
categories:
- Hexo
tags:
- Hexo
- Mac

---

# 环境配置
## 1. node.js
用来生成静态页面.Mac中自带了node.js.[下载地址](https://nodejs.org/en/)
## 2.Git
用来将本地Hexo内容提交到Github上。Xcode自带Git，这里不再赘述。如果没有Xcode可以参考[Hexo官网](https://hexo.io/docs/)上的安装方法。
<!--more-->
# 搭建步骤
## 一般搭建方法
> 待续...

## 优化搭建方法(全部存放于gitHub中,包括Hexo源文件和生成的静态页面文件)
> 概述:
Hexo部署到GitHub上的文件，是.md（你的博文）转化之后的.html（静态网页）。因此，当你重装电脑或者想在不同电脑上修改博客时，就不可能了。如果我们将Hexo生成的网站文件存放到GitHub上进行管理的。这样，不仅解决了上述的问题，还可以通过git的版本控制追踪你的博文的修改过程。
> 我们需要有两个分支，一个用来存放Hexo网站的文件，一个用来发布网站。


### 搭建流程
1. 创建仓库，`Flametinary.github.io`；
2. 创建两个分支：`master` 与 `hexo`；
3. 设置`hexo`为默认分支（我们只需要手动管理这个分支上的Hexo网站文件）；
4. 使用`git clone git@github.com:FlameTinary/FlameTinary.github.io.git`拷贝仓库；
5. 在本地`Flametinary.github.io`文件夹下通过`Git bash`依次执行`npm install -g hexo-cli
`、`hexo init`、`npm install` 和 `npm install hexo-deployer-git`（此时当前分支应显示为`hexo`）;
6. 修改`_config.yml`中的`deploy`参数，分支应为`master`；
7. 依次执行`git add .`、`git commit -m “…”`、`git push origin hexo`提交网站相关的文件；
8. 执行`hexo generate -d`生成网站并部署到GitHub上。



# 相关问题解决
1. 在将hexo源文件提交到git的hexo目录中出现`modified:   themes/next`的问题.

    **原因:** 由于next主题是存放于GitHub中的,因此在将next主题clone下来的时候,同时携带了一些GitHub的相关文件,是由这些脏数据引起的.
    
    **解决办法:** 删除相关文件即可,进入存放hexo的文件夹->themes->next,其中有`git`/`GitHub`/`gitattribute`,三个文件,删除即可.
    
2. git 分支可以commit成功,但是push不到origin hexo.
**原因:** 本地的hexo分支没有关联到origin上的hexo分支.
**解决办法:**使用`git branch --set-upstream hexo origin/hexo`.

3. 如果在修改hexo文件,比如修改next主题后,`hexo s`显示了修改后的效果,但是 `hexo generate`/`hexo deploy`后却在GitHub page上没有效果,可以先 `hexo clean`一下,然后在执行部署`hexo g -d`.
4. 在`hexo deploy`后显示error:`Deployer not found: git`,尝试执行`npm install hexo-deployer-git --save`可修复.
5. 执行`hexo s`命令后，页面访问显示`Cannot GET/`, 重新执行`npm install`.
6. 执行`hexo deploy`,打开gitHub page,显示页面404 not found.
**原因:**主要是Jekyll升级所致 
**解决办法:**
    1. .deploy_git 目录, 添加 .nojekyll 空文件
    2. source目录, 添加.nojekyll 空文件
    3. 修改 Hexo 上层_config.yml配置文件, 添加`include: - .nojekyll`
    4. 重新部署推送: `hexo d -g`


