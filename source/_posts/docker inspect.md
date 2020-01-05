---
title: docker inspect命令
tags:
    - docker
    - docker 命令
abstract: 本文比较简单，就说说docker inspect在使用中的小技巧，以提高inspect时的效率
posturl: docker-inspect
date: 2020-01-05 18:18:59
---

## 描述与简介

`docker inspect`是docker客户端的原生命令，用于查看docker对象的底层基础信息。包括容器的id、创建时间、运行状态、启动参数、目录挂载、网路配置等等。另外，该命令也可以用来查看docker镜像的信息。
官方描述如下：
> Return low-level information on Docker objects

## 语法
语法如下：

> docker inspect [OPTIONS] NAME|ID [NAME|ID...]

### OPTIONS选项

下表摘自官网
<table>
<thead>
<tr>
    <td>Name, shorthand</td>
    <td>Default</td>
    <td>Description</td>
  </tr>
</thead>
<tbody>
<tr>
    <td><code class="highlighter-rouge">--format , -f</code></td>
    <td></td>
    <td>Format the output using the given Go template</td>
  </tr>
    
  <tr>
    <td><code class="highlighter-rouge">--size , -s</code></td>
    <td></td>
    <td>Display total file sizes if the type is container</td>
  </tr>

  <tr>
    <td><code class="highlighter-rouge">--type</code></td>
    <td></td>
    <td>Return JSON for specified type</td>
  </tr>

 <!-- end for option -->
</tbody>
</table>

如上表，`--type`用于指定docker对象类型，如：container, image。在容器与镜像同名时可以使用，使用频率较低。比如，当你机器上一个容器名为redis， 一个镜像为redis:latest，则可以使用下面的命令查看镜像信息。不使用type参数，则返回容器信息：
```bash
# 查看redis:latest镜像信息
docker inspect --type=image redis

# 查看redis容器信息
docker inspect redis
```

`--size`用于查看容器的文件大小，加上该参数，输出的结果中会包含`SizeRootFs`和`SizeRw`（目前我还不是很确定这两个值的含义，望知情者告知）。

以上两个参数都是用得比较少的，`--format`实用性最大，使用频率也比较高。从表格描述可知，传入的参数值应该是go语言的模板。它很强大，可以做很多go函数的操作，由于我的go语言还没有入门，所以这里就不说太多耍杂技的了，以免翻车，下面说一下常用的。


## 实践

在实践中，我们往往只需要查看其中部分信息，比如目录挂载信息、网络信息。而直接输入`docker inspect container`时，会输出容器的所有信息，
就显得比较臃肿，我们在命令行中翻页还不方便。 此时，`--format`的实用性就体现出来了。实践中的常用操作如下

### 查看目录挂载信息
输入如下命令, 则会输出容器的Mounts信息，可以看到容器中各个目录在宿主机的具体挂载位置。
```bash
docker inspect --format="{{json .Mounts}}"  container
```

参数中的json是go语言的方法名，后面是取Mounts的值做json化处理。去掉json也是可以的。
如果觉得这样输入还是不太好看，可以对json再作进一步处理，如使用python的json模块或者jq美化输出。命令如下：
```bash
#使用python的json模块美化

docker inspect --format="{{json .Mounts}}" container | python -m json.tool

#使用jq美化

docker inspect --format="{{json .Mounts}}" container | jq

```

### 查看容器网络信息

查看网络信息可以使用下面命令：

```bash
#查看完整网络信息

docker inspect --format="{{json .NetworkSettings}}" container | jq

#查看网络端口映射

docker inspect --format="{{json .NetworkSettings.Ports}}" container | jq

# 查看容器的网络ip、网关等信息

docker inspect --format="{{json .NetworkSettings.Networks}}" container | jq

```

## 延伸学习
如果感兴趣，还可以充分利用这个`--format`参数，因为它是go的模板语法，差不多是可以写go的代码。例如上述的命令，`json`就是go的方法名
所以可以结合其他的go方法（如`range`，`split`）来耍杂技，本文就不班门弄斧了。

## 参考资料
[docker官方文档](https://docs.docker.com/engine/reference/commandline/inspect/)