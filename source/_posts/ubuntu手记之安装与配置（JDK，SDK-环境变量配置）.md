---
title: ubuntu手记之安装与配置（JDK，SDK 环境变量配置）
date: 2016-08-23 14:35:19
tags: Ubuntu
categories: Ubuntu
---
下面只是一个备忘：
分区方案：
我的电脑是4G 内存，500G硬盘;由于是linux小白一枚，完全是百度的，也不知道分得对不对请大神能够指点一二;
```
/ : 逻辑分区，加大！50G = 50 000MB.//ext4文件系统
boot: 逻辑分区，不少于200MB，给1G=1000MB吧.//ext4文件系统
swap空间： 逻辑分区，8 000MB (我有4G内存.).//ext4文件系统（这个也不不知道分得对不对）
home： 逻辑分区，300G = 100 000MB.//ext4文件系统
/opt    逻辑分区  100G //ext4文件系统
```
<!--more-->
我都放在了逻辑分区里面，防止主分区挂了，网上是这样说的，也不知道会不会影响运行速度，所有东西都没有了由于是一枚程序员，开发android ，所以想要把android studio和sdk放在opt上;所以单独分出来一个程序;

之后是安装jdk，在jdk官网下载jdk，之后创建/usr/lib/jvm下;
```
sudo mkdir /usr/lib/jvm
```
之后将jdk解压后拷贝到这个目录下，并重命名为java，以便记忆;
之后配置环境变量;
首先
```
sudo gedit /etc/profile
```
在文本的末尾加上
```
export JAVA_HOME=/usr/lib/jvm/java   
export JRE_HOME=${JAVA_HOME}/jre  
export CLASSPATH=.:${JAVA_HOME}/lib:${JRE_HOME}/lib  
export PATH=${JAVA_HOME}/bin:$PATH
```
之后
```
source /etc/profile
```
命令进行保存生效;可以先输入 echo $PATH 查看设置的path有没有问题，
如果没有问题，输入java -version可以打印出java版本;

jdk环境变量配置完之后配置Android  sdk，[Android SDK 中国下载地址:]http://www.android-studio.org/ 为什么在这里下载你们懂得;
之后将SDK 解压到/opt目录下/创建Android/SDK的文件夹，
之后配置SDK环境变量，包括adb，同样的输入sudo gedit /etc/profile进入文本，在末尾处添加;
```
export ANDROID_HOME=/opt/jiangcy/Android/SDK
export PATH=$ANDROID_HOME/tools:$ANDROID_HOME/platform-tools:$PATH
```
之后source /etc/profile生效，通过输入echo $PATH才看，路径是否正确;
在终端中输入android 会出现android sdk manager 可以通过android sdk中国下载地址中的sdk代理说明，设置代理，下载android sdk manager;

在使用过程中发现切换到root用户后，输入java ，android ，adb命令失效了，不知道是不是我配置的有问题后来百度找到解决办法;
通过 sudo gedit /root/.bashrc 打开文本，在末尾处添加 source /etc/profile 保存后执行;据说直接执行./root/.bashrc就ok了 ，但是我没有成功不知道为什么，提示 bash: ./root/.bashrc: 没有那个文件或目录，但是没有关系，重启之后也生效了
