---
title: RxNetty 在android上的使用之TCP长连接
date: 2016-08-29 14:14:14
tags: Android
categories: Android
---
在上个项目开始我们开始使用tcp异步通信机制来实现所需要的功能，使用异步的方式主要的好处能够不阻塞，以便能在接收数据的时候更加流畅，我们选用了netty异步通讯框架来实现这个功能，之前我写了一篇关于netty实现异步通讯长连接的文章，但是在使用中我发现，有的时候会莫名其妙的报连接断开的现象，而且代码逻辑也不是特别好，之后我在github上发现rxnetty这个库不错（主要是最近在学习rx……的使用，所以在使用netty的时候就想会不会有一个使用rxjava实现的netty框架呢？于是在github上发现真的有 哈哈哈）；这个框架可以更根据设置读取数据的时间就可以自己尝试重连操作；本文中使用的代码完全是在上一篇netty实现长链接: http://www.jianshu.com/p/2dfecc719cd5 的基础上修改简化的，

<!--more-->

```
package com.jcy.nettyserver;

import android.os.Handler;
import android.os.Message;
import android.util.Log;

import com.alibaba.fastjson.JSON;
import com.jcy.data.ReceiveData;

import java.net.InetSocketAddress;
import java.net.SocketAddress;
import java.util.concurrent.TimeUnit;

import io.netty.channel.ChannelOption;
import io.netty.channel.ChannelPipeline;
import io.netty.handler.codec.LineBasedFrameDecoder;
import io.netty.handler.codec.string.LineEncoder;
import io.netty.handler.codec.string.LineSeparator;
import io.netty.handler.codec.string.StringDecoder;
import io.netty.handler.codec.string.StringEncoder;
import io.netty.util.CharsetUtil;
import io.reactivex.netty.channel.Connection;
import io.reactivex.netty.protocol.tcp.client.TcpClient;
import rx.Observable;
import rx.functions.Action1;

/**
 * @className: RxNettyManager
 * @desc:
 * @author: Jiangcy
 * @datetime: 2016/8/2
 */
public enum RxNettyManager {
    instance;
    //多长时间为请求后，发送心跳
    private Connection<String, String> mConnection;
    private String serverIP;
    private int port;
    private static final String TAG = "NettyManager";
    private String HEAT_STRING;//心跳数据
    private String LOGIN_STRING;//登陆数据
    private boolean isOnLine = false;
    private String heartAction;
    private int spacingTime = 5;
    private NettyListener mListener;


    RxNettyManager() {
    }


    /**
     * 初始化RxNetty异步通信库
     * @param serverip      服务器IP
     * @param port          通讯端口号
     * @param login         登陆数据
     * @param heartStr      心跳数据
     * @param heartAction  心跳数据Action
     * @param spacingTime  心跳间隔
     */
    public void init(String serverip, int port,String login, String heartStr, String heartAction, int spacingTime) {
        this.serverIP=serverip;
        this.port=port;
        LOGIN_STRING = login;
        HEAT_STRING = heartStr;
        this.heartAction = heartAction;
        this.spacingTime=spacingTime;
    }


    private Handler mHandler = new Handler(){
        @Override
        public void handleMessage(Message msg) {
            super.handleMessage(msg);
            mHandler.removeMessages(0);
            send(HEAT_STRING);
        }
    };

    /**
     * 客户端连接服务器
     */
    public void connectServer() {
        if (LOGIN_STRING.equals("") || LOGIN_STRING == null) {
            new RuntimeException("LOGIN_STRING isEmpty please init first");
            return;
        }
        if (HEAT_STRING.equals("") || HEAT_STRING == null) {
            new RuntimeException("HEAT_STRING isEmpty  please init first");
            return;
        }
        SocketAddress socketAddress = new InetSocketAddress(serverIP, port);
        Log.e(TAG, " rxNettyClientConnect  socketAddress : " + socketAddress.toString());
        TcpClient.newClient(socketAddress)
                .<String, String>pipelineConfigurator(new Action1<ChannelPipeline>() {
                    @Override
                    public void call(ChannelPipeline pipeline) {
                        //这部分设置一定要和服务器上设置相同，要不然会出现无法连接的现象
                        // Decoders
                        pipeline.addLast("frameDecoder", new LineBasedFrameDecoder(1024 * 1024 *
                                1024));
                        pipeline.addLast("stringDecoder", new StringDecoder(CharsetUtil.UTF_8));
                        // Encoder
                        pipeline.addLast("stringEncoder", new StringEncoder(CharsetUtil.UTF_8));
                        pipeline.addLast("lineEncoder", new LineEncoder(LineSeparator.UNIX,
                                CharsetUtil.UTF_8));
                    }
                })
                .channelOption(ChannelOption.SO_KEEPALIVE, true)
//设置读取超时时间
                .readTimeOut(spacingTime+3, TimeUnit.SECONDS)
                .createConnectionRequest()
                .subscribe(newConnection -> {
                    mConnection = newConnection;
                    mConnection.getInput().subscribe(message -> {
                        Log.e(TAG, "receive : " + message);
                        String action = JSON.parseObject(message, ReceiveData.class).action;
                        //根据心跳进行心跳保活发送数据
                        if (heartAction.equals(action))
                            mHandler.sendEmptyMessageDelayed(0,spacingTime*1000);
                        if (mListener != null) mListener.reciveData(action, message);
                    }, e -> reconnect(e));
                }, e -> reconnect(e), () -> {
                    if (mListener != null) mListener.onLine();
                    isOnLine = true;
                    Log.e(TAG, "connect success");
                    send(LOGIN_STRING);
                });

    }

    /**
     * 断开自动重新连接
     */
    private void reconnect(Throwable e) {
        mHandler.removeCallbacksAndMessages(null);
        //延迟spacingTime秒后进行重连
        Observable.timer(spacingTime, TimeUnit.SECONDS).subscribe(l -> {
            if (mConnection != null) mConnection.closeNow();
            Log.e(TAG, "reconnect");
            if (isOnLine) {
                if (mListener != null) mListener.offLine();
            }
            isOnLine = false;
            connectServer();
        });
    }


    public void setListener(NettyListener listener) {
        mListener = listener;
    }

    /**
     * 发送数据
     *
     * @param s
     */
    public boolean send(String s) {
        if(mConnection==null)return false;
        Log.w(TAG, "send : " + s);
//在发送的时候一定要将subscribe实现，开始在使用write函数的时候由于rxjava知识不牢，导致只是实现
//writeString函数没有实现观察者，导致根本没有发送，所以之前一直是发送失败；
        mConnection.writeString(Observable.just(s)).subscribe(v -> {
//暂时还不知道是干啥用的
        }, e -> {
//发送失败，出现异常
        }, () -> {
//发送成功
        });
        return true;
    }

    /**
     * 停止服务
     */
    public void stopServer() {
        mHandler.removeCallbacksAndMessages(null);
        if (mConnection != null) {
            mConnection.closeNow();
            mConnection=null;
        }
    }

}
```
使用rxnetty只需要简单的几句话就能解决netty tcp长连接的实现过程，由于没有handler所以使用一个方法再其他程序中使用能够更加的方便；还有就是在使用可以通过使Lambda 表达式来简化代码，使代码的可读性能够增加，还有就是在使用长链接的时候推荐使用rxAndroid，使用这个方法最主要的好处就是子线程可以很容易的将数据发送到主线程来更新界面；
关于Lambda 在android中的使用
主要是在android studio的project的build中
 ```
dependencies {   
 classpath 'com.android.tools.build:gradle:2.1.2'    
classpath 'me.tatarka:gradle-retrolambda:3.2.5'  
}
```
并且在使用的moudle中声明
```
apply plugin: 'com.android.application'
//加入plugin声明
apply plugin: 'me.tatarka.retrolambda'
android {
    compileSdkVersion 24
    buildToolsVersion "24.0.0"

    defaultConfig {
        applicationId "com.jcy.rxnetty"
        minSdkVersion 15
        targetSdkVersion 24
        versionCode 1
        versionName "1.0"
    }
    buildTypes {
        release {
            minifyEnabled false
            proguardFiles getDefaultProguardFile('proguard-android.txt'), 'proguard-rules.pro'
        }
    }
    //加入compileOptions,这会让IDE使用用JAVA8语法解析
    compileOptions {
        sourceCompatibility JavaVersion.VERSION_1_8
        targetCompatibility JavaVersion.VERSION_1_8
    }
}
```

以下关于导入rxnetty包出现问题的解决方案来自以为网友写的博客
把RxNetty的tcp包加入到依赖，直接这样编译会有两个问题，第一个问题是jar重复：
```
com.android.build.api.transform.TransformException: com.android.builder.packaging.DuplicateFileException: Duplicate files 
copied in APK THIRD-PARTYFile1: 
C:\Users\XXX.gradle\caches\modules-2\files-2.1\org.openjdk.jmh\jmh-
core\1.11.2\f4f8cd9874f5cdbc272b715a381c57e65f67ddf2\jmh-core-1.11.2.jarFile2: 
C:\Users\XXX.gradle\caches\modules-2\files-2.1\org.openjdk.jmh\jmh-generator-
annprocess\1.11.2\72d854bf76ba5e59596d4c887a6de48e7003bee2\
jmh-generator-annprocess-1.11.2.jar



dependencies {
 ... 
compile('io.reactivex:rxnetty-tcp:0.5.2-RC1') 
{ exclude group: 'org.openjdk.jmh' }
 ...
}
```
另一个问题是引用的netty包中META-INF/下的部分文件重复。 

```
packagingOptions {
    ...
    exclude 'META-INF/INDEX.LIST'
    exclude 'META-INF/BenchmarkList'
    exclude 'META-INF/io.netty.versions.properties'
    exclude 'META-INF/CompilerHints'
    ...
  }
```
本文中还是使用了两个库文件一个是rxjava和fastjson
 ```
   compile 'io.reactivex:rxjava:1.0.14'

    compile 'com.alibaba:fastjson:1.2.15'

```
