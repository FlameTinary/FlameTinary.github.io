---
title: linux 配置远程登录服务器
date: 2020-01-21 21:51:58
tags:
- hide
categories:
- hide
---

## 创建服务器

创建谷歌GCP服务器

## 在服务器配置sshd_config

### 登录服务器

打开终端, 输入

```bash
ssh root@<这里是你服务器的ip地址>
```

输入密码, 就完成了服务器的登录

### 使用vim打开`sshd_config`文件

在服务器中打开sshd_config文件

```bash
vi /etc/ssh/sshd_config
```

修改`sshd_config`文件中的几个地方

```bash
RSAAuthentication yes     # RSA认证
PubkeyAuthentication yes  # 公钥认证
AuthorizedKeysFile .ssh/authorized_keys  # 公钥认证文件路径
```

## 本地生成ssh-keygen

回到本地, 使用`ssh-keygen`生成公钥和私钥

## 将生成的public key放到服务器

进入`/.ssh`目录, 执行命令

```bash
cat -n ~/.ssh/id_rsa.pub>>root@<这里是你服务器的ip地址>:~/.ssh/authorized_keys
```

期间会输入服务器的密码.

## 本地创建ssh的config文件, 并配置host

在本机的目录`~/.ssh`下创建config文件

```bash
touch config
```

然后使用vim打开进行编辑

```bash
#ssh模版
Host            hostname
HostName        xxx.xxx.xxx.xxx
Port            22
User            root # 用户名
ServerAliveInterval 60 # 每隔60秒 发送KeepAlive请求，保证不会因为超时空闲断开
IdentityFile    ~/.ssh/id_rsa #私钥地址
```

## 使用配置好的Host登录远端服务器

使用命令

```bash
ssh <刚才配置的HostName>
```

就完成了登录
