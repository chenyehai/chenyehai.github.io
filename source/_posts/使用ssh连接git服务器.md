---
title: 使用ssh连接git服务器
date: 2018-09-23 14:07:40
tags: git
abstract: 本文讲述使用ssh协议连接git服务器的配置方法。使用ssh连接可以免去clone、push、pull等操作时输入账号密码，使用方便。
posturl: git-ssh
---
### 配置git-ssh主要步骤如下：
1. 安装git（选择自己喜欢的安装方法即可），配置git的user、email等基本操作。为了步骤的完整，这里也写一下配置user、email的方法。
```bash
$ git config --global user.name "username"
$ git config --global user.email "example@email.com"
```

2. 生成密钥文件。
```bash
$ ssh-keygen -t rsa -C "example@email.com"
```
回车执行该命令，可以一路回车下去密钥文件生成完毕。 `-t`参数表示密钥类型， -C参数是注释（comment），所以这个理论上可以随便写（有文章说这个是git的用户邮箱，其实没有限制）

3. 查看（可以执行`cat ~/.ssh/id_rsa.pub`）并拷贝公钥文件内容到git服务器，即上一步生成的pub后缀文件（默认id_rsa.pub）。
4. 执行测试是否成功.
```bash
$ ssh -T git@github.com
```
看到如下提示内容：

> The authenticity of host 'github.com (207.97.227.239)' can't be established.
> 
> RSA key fingerprint is SHA256:jok3FH7q5LJ6qvU8iPNehBgXRw51ErE77S0Dn+Vg/Ik.
> 
> RSA key fingerprint is MD5:98:ab:2b:30:60:00:82:86:bb:81:db:87:22:c4:2f:b1.
>
> Are you sure you want to continue connecting (yes/no)?

输入yes，回车即可。此时正常则会出现成功的提示。

5. 可以不输密码直接进行`clone` 项目了。

### 使用`https`协议`clone`的项目，修改为ssh协议
如果有项目之前用https输入密码`clone`的，现在仍然需要输入账号密码。因此，需要手动修改项目的`git`配置。打开项目的`.git`文件夹下的config文件，找到remote节点的内容如下：
> ［remote "origin"］
> 
> 　　　　url = <font color='#0f0'>https://</font>github.com<font color='#0f0'>/</font>username/project.git
> 
> 　　　　fetch = +refs/heads/*:refs/remotes/origin/*

修改为
> ［remote "origin"］
> 
> 　　　　url = <font color='#f00'>git@</font>github.com<font color='#f00'>:</font>username/project.git
> 
> 　　　　fetch = +refs/heads/*:refs/remotes/origin/*

主要是修改协议名，以及域名后面的第一个斜杆改为冒号，已经用颜色区别。还可以直接从git服务器页面上拷贝那个ssh的url过来，快捷不会出错。

### 其他方法
如果你使用的git服务器不支持ssh协议连接，http协议连接也有方法免去每次连接都输入密码。方法就是使用`git`的凭证存储功能。
执行如下命令进行配置
```bash
$ git config --global credential.helper cache
```
最后一个参数`cache`表示是缓存模式，该模式会将凭证存放在内存中一段时间。 密码永远不会被存储在磁盘中，并且在15分钟后从内存中清除。也可以在后面加上缓存时间参数`--timeout seconds`指定缓存seconds秒。
除了`cache`模式以外，还有`store`模式，永久存储，将上述命令改为如下命令即可：
```bash
$ git config --global credential.helper store
```
** 注意： store模式的密码是明文保存的 **