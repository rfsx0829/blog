---
title: Golang 程序在 Docker 容器中对外发送 https 请求的一个问题及解决方案
date: 2020-03-06 18:49:24
categories: Golang
tags:
  - Golang
  - 小问题
  - Docker
  - https
  - openssl
abbrlink: 3060
---

最近没事，打算把博客的背景图片弄成像[必应](https://cn.bing.com)一样，每天都是很好看的图。简单查了一下，有一些别人开发的公开的 API 可以使用，直接获取必应的每日图片，但我还是打算自己写一个，于是又开始了折腾。

具体代码可以在 [little-tools/bingImg](https://github.com/rfsx0829/little-tools/tree/master/bingImg) 仓库找到。内容也都很简单。到这里都没有遇到什么问题。

在把程序部署到服务器的时候，我更喜欢静态编译 Golang 程序，然后做成 Docker 的镜像，这样会方便一些。但是有些时日没碰 Go，以前敲得挺熟悉的命令居然忘记了，现在就把这个记在博客里，免得以后再忘记还到处去找。

```bash
$ CGO_ENABLED=0 GOOS=linux go build -ldflags '-extldflags "-static"'
```

然后把程序上传到服务器，写一个简单的 Dockerfile，build 成镜像，然后启动一个容器。

然后问题出现了，容器能正常启动，但是下载不到图片。查看日志

![](/blog/pics/20200306001.png)

Google 了好久终于找到了解决方案，这里记录一下。

如果 Docker 容器是从 scratch 这个空镜像构建的，并且访问 https 的站就会有上述的问题，解决方案就是把需要访问的网站的 CA 证书复制到容器的 /etc/ssl/certs/ 这个目录下。

所以在 Dockerfile 里多加一行

> ...
> ADD ./bing.com.crt /etc/ssl/certs/
> ...

然后重新 build 镜像，再启动容器就可以了。

还有个问题，要怎么去取得一个网站的 CA 证书呢？可以直接用 Chrome 访问，然后点地址栏的那个锁，导出证书就可以啦。

或者也有命令行方式，在这里给出

```bash
$ timeout 5 openssl s_client -connect cn.bing.com:443 > bing.com.crt
$ # cn.bing.com 替换要获取证书的网站噢 bing.com.crt 换成其他文件名都可
```

这样就可以把必应的证书保存到 bing.com.crt 文件中啦。

所以现在这个简单的小服务也可以对外提供服务了。如果有需要的话，可以直接使用。

[自制必应每日图片API](https://dev.bugzeng.com/api/v1/bingimg)

不过目前只有 1920*1080 分辨率的，未来获取会扩展，逃)

