---
title: CentOS7+Python3+Django
date: 2019-01-21 18:58:53
tags:
- 服务器
- Python3
- Django
- CentOS
categories:
- 服务器
photo: https://timgsa.baidu.com/timg?image&quality=80&size=b9999_10000&sec=1548078575611&di=e34e1e31642abf8ae1b2f114c50fcf82&imgtype=0&src=http%3A%2F%2Fi1.hdslb.com%2Fbfs%2Farchive%2Fc7500e86fe398f38111ed264dc559c47d498e12e.jpg
---
# CentOS 7中布置Python3和Django

Centos7中默认安装了python2.7.5，因为一些命令要用它，比如：yum；它使用的是python2.7.5。当我们在命令行里输入

```bash
python -V
```

就可以看到版本为2.7.5。

<!--more-->

安装各种依赖包：

```bash
yum -y groupinstall "Development tools"
yum -y install zlib-devel bzip2-devel openssl-devel ncurses-devel sqlite-devel readline-devel tk-devel gdbm-devel db4-devel libpcap-devel xz-devel libffi-devel gcc automake autoconf libtool make wget
```

## 安装python3

因为我们要安装python3版本，所以python要指向python3才行，目前还没有安装python3，先备份，备份之前先安装相关包，用于下载编译python3

```bash
yum install zlib-devel bzip2-devel openssl-devel ncurses-devel sqlite-devel readline-devel tk-devel gcc make
```

这几个包必须得安装，否则安装python3时可能会出现各种错误。
运行下面两个命令，进行备份

```bash
cd /usr/bin
mv python python.bak
```

下载python3.6.3

```bash
wget https://www.python.org/ftp/python/3.6.3/Python-3.6.3.tar.xz
```

解压

```bash
tar -xvJf  Python-3.6.3.tar.xz
```

进入解压好的python3.6.3文件夹

```bash
cd Python-3.6.3
```

编译安装

```bash
./configure prefix=/usr/local/python3
make
make install
```

安装完毕，/usr/local/目录下就会有python3了。

## 实现Python3和Python2的共存

添加Python3的软链

```bash
ln -s /usr/local/python3/bin/python3 /usr/bin/python
```

这个时候在执行命令python -V和python2 -V，就能看到python3和python2的版本了。

添加pip3的软链

```bash
ln -s /usr/local/python3/bin/pip3 /usr/bin/pip
```

因为执行yum需要Python2的版本，所以我们还有修改yum的配置，执行：

```bash
vi /usr/bin/yum
```

把第一行`#! /usr/bin/python`修改为`#! /usr/bin/python2`

同理

```bash
vi /usr/libexec/urlgrabber-ext-down
```

文件里面的`#! /usr/bin/python`也要修改为`#! /usr/bin/python2`

## 安装Django项目中需要的python相关包

安装python相关包需要用到python中的pip命令

```bash
pip install Django
pip install PyMySQL
pip install Scrapy
pip install beautifulsoup4
pip install bs4
pip install lxml
pip install numpy
pip install requests
pip install simplejson
pip install urllib3
```

此时，python所需要的包已经都安装好了。

## 安装MySQL

下载MySQL源安装包

```bash
wget http://dev.mysql.com/get/mysql57-community-release-el7-8.noarch.rpm
```

安装MySQL源

```bash
yum localinstall mysql57-community-release-el7-8.noarch.rpm
yum install mysql-devel
```

安装MySQL

```bash
yum install mysql-community-server
```

启动MySQL服务

```bash
systemctl start mysqld
```

查看MySQL状态

```bash
systemctl status mysqld
```

开机启动

```bash
systemctl enable mysqld
```

修改root本地登录密码

```bash
grep 'temporary password' /var/log/mysqld.log
mysql -uroot -p
set password for 'root'@'localhost'=password('!2Qw32sd');
```

注意：mysql5.7默认安装了密码安全检查插件（validate_password），默认密码检查策略要求密码必须包含：大小写字母、数字和特殊符号，并且长度不能少于8位。否则会提示ERROR 1819 (HY000): Your password does not satisfy the current policy requirements错误

配置默认编码为utf8

修改`/etc/my.cnf`配置文件，在[mysqld]下添加编码配置，如下所示：

```bash
[mysqld]
character_set_server=utf8
init_connect='SET NAMES utf8'
```

## 导入django和mysql数据库

## 导入django项目

由于我项目放在码云上面，然后CentOS又自带git，我的数据库文件也比较小，所以也放在django项目中了，用git下载下来:

```bash
sudo su root
mkdir project
cd project
git clone https://gitee.com/dafeige/django-restframework-demo.git
```

此时，我的数据库文件路径是：`project/django-restframework-demo/tutorial/test_python.sql`，由于需要将此sql文件导入到mysql数据中，需要给此文件加权限:

```bash
chmod 777 project/django-restframework-demo/tutorial/test_python.sql
```

## 导入sql数据库文件

进入数据库

```bash
mysql -u root -p
```

导入sql文件

```bash
create database test_python;
use test_python;
source project/django-restframework-demo/tutorial/test_python.sql;
```

## 部署django工程

进入到工程中

```bash
sudo su root
cd project/django-restframework-demo/tutorial
python manage.py runserver 0.0.0.0:80 &
```

最后面的”&”，这符号表示在后台运行该进程。这里的IP地址如果用公网IP

会运行不了，而用0.0.0.0则外网和127.0.0.1都能够访问。