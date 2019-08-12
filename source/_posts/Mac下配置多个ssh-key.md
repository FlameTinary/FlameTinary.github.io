---
title: Mac下配置多个ssh-key
date: 2019-08-12 15:02:02
tags:
- Mac
categories:
- iOS
---

> 在我们的工作经常会遇到公司的git仓库需要配置SSH-Key，而自己的github也需要配置SSH-Key，如果你不想使用同一个SSH-Key，那么可以生成多个SSH-Key，从而分别配置不同的仓库。

<!--more-->

打开终端，查看当前`~/.ssh`路径中存在的SSH-Key：
![](/Users/sheldon/Pictures/Xnip2019-05-28_15-54-48.jpg)

我的路径下已经存在了一个SSH-Key了，那么接下来再创建一个SSH-Key:

![](/Users/sheldon/Pictures/Xnip2019-05-28_15-57-34.jpg)

使用命令创建：
```bash
ssh-keygen -t rsa -C "这里填写你的邮箱地址"
```
在`Enter file in which to save the key (<文件路径>):`这里输入自定义的SSH-Key名称**（必须写，不然会默认覆盖原来的Key）**

例如：
```bash
Enter file in which to save the key (<文件路径>): id_rsa_github
```

然后输入密码（密码不可见）：
```bash
Enter passphrase (empty for no passphrase):
Enter same passphrase again:
```

当看到这样的字样，说明创建成功了：
![](/Users/sheldon/Pictures/Xnip2019-05-28_16-01-35.jpg)

将SSH-Key添加到ssh-agent中：
```bash
ssh-add id_rsa_github
```
输入密码:
```bash
Enter passphrase for id_rsa_github:
```

添加成功：
```bash
Identity added: id_rsa_github (<邮箱地址>)
```

查看公钥：
```bash
cat id_rsa_github.pub
```

将公钥配置到github的ssh-key中；最后验证是否联通：
```bash
ssh -T git@github.com
```

成功字样：
```bash
Hi <这里是github的用户名>! You've successfully authenticated, but GitHub does not provide shell access.
```