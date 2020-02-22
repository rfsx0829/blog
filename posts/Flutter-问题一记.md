---
title: Flutter 问题一记
tags:
  - Flutter
  - Android
  - 小问题
date: 2019-04-06 17:44:32
categories: Flutter
abbrlink: 406
---
# Flutter 环境配置的小问题

因为之前重置过 Mac 一次，导致我的 Flutter 开发环境没有了，又得重新配置环境，而这个过程，真的是让我很头大，我觉得这甚至可以写进博客，这里可以稍微记录下大体的思路

## 下载 Flutter 并添加 bin 到 PATH

这一步倒是没什么问题，轻车熟路的，一下子就搞定了

## 配置 Android SDK

这是最烦的，在这里耗费了不知道多少时间和好心情

### 下载

第一个问题，下载，本来没什么难度，但是。。。

<!-- more -->

在 Flutter 的官方文档上有这么一个 Note

![Note](/blog/pics/1554481707317-e938bb0a-666c-48fb-970e-eb7592538ce5.png)

翻译一下：Flutter **依赖一个 Android Studio 的完整安装** 来支持它对 Android 平台的依赖，然而，你可以在很多编辑器里写你的 Flutter 应用

官网说需要一个完整的安装。。但我记得之前好像没有装 Android Studio 也能用啊，这次怎么回事

那。。到底是下载完整的 Android Studio 呢还是只下载 Command Line Tool 呢

我选择了后者

### Flutter doctor

Flutter 不认 Android SDK，始终说缺一个 platforms 文件夹，才知道原来这是可以通过 sdkmanager 安装的，于是 执行 sdkmanager 试试，得到了报错。。。网上一查。。。是Java的问题

### Java 版本冲突

网上查了很久才知道，原来 sdkmanager 只支持 JDK 10 及以下的，但是我之前。。。刚装了 JDK 12。。

于是，卸载，装 JDK 8

ps： 为什么是 JDK 8 不是 JDK 10 ？

ps： 因为 JDK8 可以直接支持 sdkmanager，9 和 10 还要去加环境变量

```bash
export JAVA_OPTS='-XX:+IgnoreUnrecognizedVMOptions --add-modules java.se.ee'
```

### Java 解决方案

1. JDK 8 及以下，可直接使用 sdkmanager 安装 Android SDK
1. JDK 9 和 JDK 10，需添加环境变量让 sdkmanager 能找到那个模块，然后再正常使用
1. JDK 11 和 JDK 12，添加环境变量无作用，因为。。那个包被移除了。。只能装低版本。。。

### 环境变量

最后的，其实挺简单，通过 sdkmanager 安装好 platforms 和 build-tools 之后，添加一个 ANDROID_HOME  就完事了，再执行 flutter doctor 看看，应该就可以识别 SDK 了

ps：可能还会有证书问题，这个 flutter 给了提示，按提示来可以手动确认证书

## 手机

电脑环境倒是配置好了，手机又出问题了，之前在玩 Island，但似乎它的权限有点过大了，居然把 Android 调试给劫持了，把这个软件里面的一个功能关掉才能正常连接电脑。。。

然后几分钟后，手机疯狂死机重启。。。

然后就重装了 Pixel Experience CAF

## 小结

深夜了，环境终于全部搞完，我的三天假期就只剩下两天来做开发，一天都被配这个环境给浪费掉了，从心说一句，垃圾 Java ！自己都不向下兼容，你开发**呢
