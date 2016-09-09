---
title: Ubuntu 安装facebook的buck
date:  2016-08-26 14:37:29
tags: Ubuntu
categories: Ubuntu
---

记录自己在安装配置buck过程中出现的bug，写本文的主要目的是放了方便记录，如果文中哪些技术使用不当，希望能改指出;
**
Buck环境配置
**
有两种方式可以下载Buck，一种是通过brew（这个通过apt-get install brew ）安装，但是由于中国网络的原因，我并没有安装成功，所以我使用了第二种方式，通过源码安装;
需要下载[buck](https://github.com/facebook/buck)和[Watchman](https://github.com/facebook/watchman)源码
<!--more-->
克隆buck源码
```
git clone https://github.com/facebook/buck.git
```
克隆watchman源码，Facebook 开源的一个文件监控服务，用来监视文件并且记录文件的改动情况，当文件变更它可以触发一些操作，例如执行一些命令等等。安装watchman，是为了避免Buck每次都去解析 [build files](https://buckbuild.com/concept/build_file.html)，同时可以缓存其他一些东西，减少编译时间。
```
git clone https://github.com/facebook/watchman.git
```
下载速度相当之坑爹，由于下载这个源码的原因，耽误了我将近一天的时间;
有需要的同学可以去我的网盘中下载日期是2016/08/09 （这可是情人节啊，为了下载这个都没过好） 网盘地址[buck]:http://yunpan.cn/c6bZ2hD5bXKfD （提取码：71c6）和[watchman]: http://yunpan.cn/c6bZ6GxEjcwvi （提取码：68a5）;
在下载源码的同时可以先将编译源码所需要的**环境配置**好;

//编译watchman所需要
```
sudo apt-get install automake 
```
[Oracle JDK 7](http://www.oracle.com/technetwork/java/javase/downloads/jdk7-downloads-1880260.html)
[Apache Ant 1.8 (or newer)](http://ant.apache.org/)
[Python 2.6 or 2.7](https://www.python.org/downloads/)
[Git](http://git-scm.com/download)
C 编译器：[gcc](http://gcc.gnu.org/)或者[clang](http://clang.llvm.org/get_started.html)
[Android SDK](https://dl.google.com/dl/android/studio/ide-zips/2.0.0.20/android-studio-ide-143.2739321-linux.zip)
[Android NDK（r10e）](http://pan.baidu.com/s/1c2Now4g)

JDK,Android SDK 配置配置查看 我之前写的一片文章http://www.jianshu.com/p/9c035cdc1899
**安装NDK**
Android  NDK 这个一定要使用r10e，我之前使用的r11b但是由于在编译buck的过程中没有成功，提示没有arm-linux-androideabi-4.8，后来使用r10e，编译通过;
下载NDK后由于是bin格式，所以直接安装即可，安装之前不要忘记给权限，我的安装地址是/opt/Android/NDK/android-ndk-r10e，安装成功之后配置环境变量;
首先sudo gedit /etc/profile将环境变量添加到最后;
```
export NDK_HOME=/opt/Android/NDK/android-ndk-r10e
export PATH=$NDK_HOME:$PATH
```
**安装 ANT**
下载[Apache Ant 1.8 (or newer)](http://ant.apache.org/) http://ant.apache.org/，我使用的是最新版，apache-ant-1.9.7，解压到/opt/apache-ant-1.9.7
之后配置环境变量
```
export ANT_HOME=/opt/apache-ant-1.9.7
export PATH=$ANT_HOME/bin:$PATH  
```
通过ant -version 查看是否安装成功
**安装GIT**
```
sudo apt-get install git
```
**安装python**
```
sudo apt-get install python-dev
```
通过输入 python查看安装版本，由于我之前使用的是ubuntu自带的python，在编译watchman的时候提示，没有python.h 文件;忘记错误是什么了，反正是重新安装了一下最新的python之后没有问题了，
**GCC**
这个ubuntu里面自带，这里不再赘述了;
****
buck和watchman的源码终于下完了;
**开始编译watchman**
进入到watchman的目录之后
```
$ cd watchman 
$ ./autogen.sh
$ ./configure 
$ make 
$ sudo make install
```
在经历各种问题之后终于安装成功了;watchman -v 查看版本 我的是4.7.0

**开始编译buck**
```  
$ cd buck
$ ant
$ ./bin/buck --help
```
在编译过程中出现过由于NDK 版本不对导致编译失败，还有就是由于源码下载不完整导致编译失败;
以后会频繁使用到buck安装目录下面的bin/buck命令，所以为了方便在Terminal中直接执行buck命令，需要将该目录添加到环境变量中，cd到用户主目录，打开.bash_profile文件：
```
 sudo gedit /etc/profile
```

添加如下配置：
```
export PATH=/home/jiangcy/software/buck-master/bin:$PATH
```
接着执行如下命令使该配置立即生效：

```
 source  /etc/profile
```

**快速创建基于 Buck 构建的 Android 工程**

使用命令可以快速创建一个Android工程，该命令执行过程中会要求你补全如下两个参数的值：
```
touch .buckconfig && buck quickstart
```
```
Enter the directory where you would like to create the project: BuckTest
Thanks for installing Buck!

In this quickstart project, the file apps/myapp/BUCK defines the build rules. 

At this point, you should move into the project directory and try running:

    buck build //apps/myapp:app

or:

    buck build app

See .buckconfig for a full list of aliases.

If you have an Android device connected to your computer, you can also try:

    buck install app

This information is located in the file README.md if you need to access it
later.

```


进入到工程根目录，在Terminal中输入如下命令创建IntelliJ工程：
```
buck project --ide IntelliJ
```
创建成功，显示一下数据
```
jiangcy@jiangcy:~/jcybuck/BuckTest/BuckTest$ buck project --ide IntelliJ
Using watchman.
OK   //java/com/example/activity:activity#dummy_r_dot_java BUILT_LOCALLY buck-out/gen/java/com/example/activity/__activity#dummy_r_dot_java_dummyrdotjava_output__/activity#dummy_r_dot_java.jar
[-] PROCESSING BUCK FILES...FINISHED 0.1s [100%] 🐳  New buck daemon
[-] GENERATING PROJECT...FINISHED 1.5s
[-] DOWNLOADING... (0.00 B/S AVG, TOTAL: 0.00 B, 0 Artifacts)
[-] BUILDING...FINISHED 0.7s [100%] (2/2 JOBS, 2 UPDATED, 2 [100.0%] CACHE MISS)
```
之后进入工程目录查看Readme中的的使用命令，
```
Thanks for installing Buck!

In this quickstart project, the file apps/myapp/BUCK defines the build rules. 

At this point, you should move into the project directory and try running:

    buck build //apps/myapp:app

or:

    buck build app

See .buckconfig for a full list of aliases.

If you have an Android device connected to your computer, you can also try:

    buck install app

This information is located in the file README.md if you need to access it
later.
```
主要参考文献
[基于Facebook Buck改造Android构建系统之初体验](http://www.jianshu.com/p/1e990aac7836)
[使用buck构建你的Android App](http://www.blogjava.net/xiaomage234/archive/2014/10/22/418951.html)
