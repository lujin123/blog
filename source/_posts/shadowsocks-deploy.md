---
title: Docker部署shadowsocks
date: 2019-05-18 00:14:46
tags: [docker, shadowsocks]
---
简单记录下使用`docker`部署`shadowsocks`的步骤，顺道写一个自动脚本，加快部署，方便下次在`ip`被墙之后快速的部署科学上网工具

## 安装docker
这一步按照[`docker`](https://docs.docker.com/install/linux/docker-ce/ubuntu/)官网提示一步步走即可，反正很简单，简单列一下`ubuntu`中的步骤：

```
> apt-get remove docker docker-engine docker.io containerd runc
> apt-get update
> apt-get install \
    apt-transport-https \
    ca-certificates \
    curl \
    gnupg-agent \
    software-properties-common
> curl -fsSL https://download.docker.com/linux/ubuntu/gpg | apt-key add -
> apt-key fingerprint 0EBFCD88
> add-apt-repository \
   "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
   $(lsb_release -cs) \
   stable"
> apt-get update
> apt-get install docker-ce docker-ce-cli containerd.io
```
如果提示权限不足，可以切换成`root`再试

## 部署shadowsocks
安装完成`docker`后即可创建`shadowsocks`环境，步骤如下：
```sh
> docker pull oddrationale/docker-shadowsocks
> docker run -d -p 12345:12345 oddrationale/docker-shadowsocks -s 0.0.0.0 -p 12345 -k password -m aes-256-cfb
```
这样基本就可以了，解释下命令的参数含义：

1. `-p`: 表示映射的端口
2. `-s`: 表示当前服务器的`host`，一般用`0.0.0.0`即可
3. `-k`: 表示密码，设置喜欢的即可
4. `-m`: 表示数据加密所用的协议，`shadowsocks`支持的都可以写