---
title: Github 生成密钥并上传工程
date: 2016-08-20 11:56:38
tags: git
categories: git
---
**前言**
本文主要是关于GitHub，生成密钥并上传工程的步骤以及出现的问题
<!--more-->
**配置SSH**
上传文件需要配置ssh key，不然无法上传。首先先检查一下本地是否已经存在ssh key,在Git Bash输入以下指令（任意位置点击鼠标右键），检查是否已经存在了SSH keys。
```
ls -al ~/.ssh
```
如果不存在就没有关系，如果存在的话，直接删除.ssh文件夹里面所有文件：

![1400844-5423105c40f22eb3.png](http://upload-images.jianshu.io/upload_images/2642181-c94a42fc160ff253.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


设置name和emai
git config --global user.name "<your name>"git config --global user.email "<your email>"

需要注意的是这里的name是随意的，邮箱是你的联系邮箱，与github上的邮箱没有什么联系（不过我都是同一个邮箱）。
**生成ssh 密钥**
输入以下指令（邮箱就是你注册Github时候的邮箱）后，回车：
```
ssh-keygen -t rsa -C "XXXXX@qq.com"
```
一路按回车键即可，如果设置了密码请记住。这一步在~/.ssh/下生成了两个文件id_rsa 和 id_rsa.pub
**获取Key**
```
$ cat ~/.ssh/id_rsa.pub
```

![1400844-24efe9fb6231936b.png](http://upload-images.jianshu.io/upload_images/2642181-c2a8ad657a2b2e03.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

然后title随便取个名字，key 就是上面我们拷贝的内容，好了，最后我们测试一下看是否配置成功。
输入以下命令：
```
ssh git@github.com
```
成功的话会显示以下的大致内容：
```
The authenticity of host 'github.com (192.30.252.128)' can't be established.RSA key fingerprint is 
16:27:ac:a5:76:28:2d:36:63:1b:56:4d:eb:df:a6:48.Are you sure you want to continue connecting (yes/no)? yesWarning: 
Permanently added 'github.com,192.30.252.128' (RSA) to the list of known hosts.Hi git-xuhao! You've successfully authenticated, 
but GitHub does not provide shell access.Connection to github.com closed.
```
在github上创建repository
本地仓库与远程建立连接
```
cd projectdir
git init
// 将所有文件add
git add .
git commit -m "first commit"// 添加远程仓库地址
git remote add origin https://github.com/silan-liu/buttonEdgeInset.git
git push -u origin master
```
在push的时候，可能会提示，先pull，则先执行git pull origin master。

然后拷贝key
**在Github上添加SSH密钥**
在[https://github.com/settings/keys](https://github.com/settings/keys)下 add new ssh key

**情景：**
在github上创建项目，然后本地git init
然后没有git pull -f --all
然后git add .  | git commit -am "init"
导致github上的版本里有readme文件和本地版本冲突，下面给出冲突原因：

[master][~/Downloads/ios] git push -u origin master

Username for 'https://github.com': shiren1118Password for 'https://shiren1118@github.com': To https://github.com/shiren1118/iOS_code_agile.git ! [rejected]        master -> master (non-fast-forward)error: failed to push some refs to 'https://github.com/shiren1118/iOS_code_agile.git'hint: Updates were rejected because the tip of your current branch is behindhint: its remote counterpart. Merge the remote changes (e.g. 'git pull')hint: before pushing again.hint: See the 'Note about fast-forwards' in 'git push --help' for details.查看大部分资料，只有这个有用
http://www.cnblogs.com/xwdreamer/archive/2012/05/29/2523958.html
**勾选强制覆盖已有的分支（可能会丢失改动），再点击上传，上传成功。**

只有这句是核心，所以，本人就略微想了一下
[master][~/Downloads/ios] git push -u origin master -f 

至此，搞定问题
