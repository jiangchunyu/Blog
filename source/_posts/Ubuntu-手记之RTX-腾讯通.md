---
title: Ubuntu 手记之RTX (腾讯通)
date: 2016-08-27 14:36:32
tags: Ubuntu
categories: Ubuntu
---

由于小白一枚，主要参考一下几篇博客安装完成
[Ubuntu 16.04 wine rtx 2015安装过程]http://www.linuxdiyf.com/linux/21867.html
主要按照这个博客安装的rtx，其中出现的问题主要参考其他文章，我会在下面详细的写出
<!--more-->
[Ubuntu 16.04安装QQ解决方案](http://www.linuxdiyf.com/linux/22724.html)
1.安装最新版 wine
 详见 Ubuntu wine wiki（https://wiki.ubuntu.org.cn/Wine）。
ubuntu 官方有自带 wine ，但是推荐用 winehq 官方提供的最新版本 wine ，新版本解决很多以前旧版本的问题。
PPA地址：https://launchpad.net/~wine/+archive/ubuntu/wine-builds
````
sudo add-apt-repository ppa:wine/wine-builds
sudo apt-get update
sudo apt-get install wine-devel
```
 安装wine成功之后安装winetricks，
```
 sudo apt-get install winetricks
```
安装完成后安装依赖，
 为防止 32 位、64 位可能出现不兼容，执行命令的时候配置 WINEARCH 为 win32。
```
WINEARCH=win32 WINEPREFIX=~/.wine winetricks msxml3 gdiplus 
riched20 riched30 ie6 vcrun6 vcrun2005sp1 allfonts
```
安装 rtx2015

rtx 官方下载页面：http://rtx.tencent.com/rtx/download/index.shtml。
```
WINEARCH=win32 WINEPREFIX=~/.wine wine  /home/jiangcy/下载/rtxclient2015formal.exe
(下载的rtx详细地址)
```
安装完后，直接可用。字体、截图及发送、文件发送皆正常。不会出现离线后再上线就登不上的问题。
但是在安装过程中出现各种组件注册失败的现象;各种百度，之后找到解决办法，http://forum.ubuntu.org.cn/viewtopic.php?t=388115
这篇博客中提到，需要安装vcrun6sp6,
之后按照说明进行了
请按照下面的步骤进行尝试:

1. 备份 wineprefix 配置文件夹:
```
$ mv ~/.wine ~/.wine-backup
```
2. 安装 vcrun6sp6, 解决缺少 mfc42 的问题
```
$ winetricks -q vcrun6sp6
```
3. 重新安装 rtx（重新安装的过程中出现了32位不兼容的问题
WINEARCH set to win32 but '/home/jiangcy/.wine' is a 64-bit installation）
google之后找到解决方案;http://askubuntu.com/questions/136714/how-to-force-wine-into-acting-like-32-bit-windows-on-64-bit-ubuntu
首先  rm ~/.wine
之后gedit   .bashrc 大概第十行的地方添加 export WINEARCH=win32之后source .bashrc

终端中输入
```
WINEPREFIX=~/.wine32 wine setupprogram.exe
```
但是不知道 为啥好像没成功，之后我又安装了一次 vcrun6sp6
```
winetricks -q vcrun6sp6
```
之后重新安装rtx没有报错，但是还是出现一次 "注册失败" ,应该是一个bug [1], 我还不清楚有什么影响, 除此之外应该基本正常.安装成功了 ，但是由于是在家里还没有测试能不能正常登录

![2016-08-07 20-52-45屏幕截图.png](http://upload-images.jianshu.io/upload_images/2642181-2ba254206656a854.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

QQ国际版 winQQ 安装教程 http://www.ubuntukylin.com/ukylin/forum.php?mod=viewthread&tid=7688&extra=page%3D1
**由于Wine QQ一直没更新版本导致目前版本报版本过低无法使用，暂时先上UK官网的国际版Wine QQ，虽然功能没那么新，但稳定能用：**
**下载：**
下载地址：[http://www.ubuntukylin.com/application/show.php?lang=cn&id=279](http://www.ubuntukylin.com/application/show.php?lang=cn&id=279)或者下载地址：[https://pan.baidu.com/s/1bApH7o](https://pan.baidu.com/s/1bApH7o)下载后解压得到wine-qqintl文件夹，里面有三个deb包：fonts-wqy-microhei_0.2.0-beta-2_all.deb、ttf-wqy-microhei_0.2.0-beta-2_all.deb、wine-qqintl_0.1.3-2_i386.deb**安装：**
1、在wine-qqintl目录下打开终端输入：sudo dpkg -i fonts-wqy-microhei_0.2.0-beta-2_all.deb ttf-wqy-microhei_0.2.0-beta-2_all.deb wine-qqintl_0.1.3-2_i386.deb2、如果报依赖错误，输入：sudo apt-get install -f3、自动解决依赖后再执行步骤1另外需要注意的是：在登录的时候，如果输入密码时提示密码错误，请使用旁边的软件盘进行鼠标点击输入就能成功了。 


![2016-08-07 22-25-18屏幕截图.png](http://upload-images.jianshu.io/upload_images/2642181-04ce41e92b479ca3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

