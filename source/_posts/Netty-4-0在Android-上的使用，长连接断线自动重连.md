---
title: Netty 4.0在Android 上的使用，长连接断线自动重连
date: 2016-08-28 14:19:19
tags: Android
categories: Android
---
最近使用Netty 异步通讯框架 ，在使用的过程中发现如果当网络断开的时候会出现无法检测的现象；

影响长链接断开的原因主要有三种：1.服务停止，2.本地网线断开，3.公网或者局域网中交换机断开；

在使用的过程中发现在服务停止或者本地网络断开的时候netty的@ChannelHandler中的channelInactive会被调用,但是如果要是公网或者局域网交换机直接网络断开是不能立刻收到channelInactive的回调；所以我设计的是通过IdleStateHandler函数进行回调；在每次收到心跳数据之后写一个延迟发送的函数，延迟心跳时间发送心跳
<!--more-->
算了还是上代码吧，实在是写不下去了；

NettyClientBootstrap android客户端启动类
```
//netty 客户端入口程序
public class NettyClientManager{
    private String host; //ip地址
    private int port; //端口号
    private EventLoopGroup group;//EventLoop线程组
    private Bootstrap b;
    private Channel ch;
    private ScheduledExecutorService executorService;
    // 隔N秒后重连
    private static final int RE_CONN_WAIT_SECONDS = 5;
    //多长时间为请求后，发送心跳
    private static final int WRITE_WAIT_SECONDS = 7;
    // 是否停止
    private boolean isStop = false;
    private final String TAG = "NettyClientBootstrap";
//连接状态变化通知接口
    private ITCPStateListener mStateListener;
//handler实现类
    private NettyClientHandler mNettyClientHandler;
    private boolean isOnline =false;

    public NettyClientBootstrap(String host, int port) {
        this.host = host;
        this.port = port;
        group = new NioEventLoopGroup();
        b = new Bootstrap();
        mNettyClientHandler = new NettyClientHandler(mListener);
        b.group(group).channel(NioSocketChannel.class).handler(new ChannelInitializer<SocketChannel>() {
            @Override
            protected void initChannel(SocketChannel ch) throws Exception {
            /**
            *此处设置应该与服务器设置相同
            *
            **/
                ChannelPipeline pipeline = ch.pipeline();
                // Decoders
                pipeline.addLast("frameDecoder", new LineBasedFrameDecoder(1024 * 1024 *
                        1024));
                pipeline.addLast("stringDecoder", new StringDecoder(CharsetUtil.UTF_8));
                // Encoder
                pipeline.addLast("stringEncoder", new StringEncoder(CharsetUtil.UTF_8));
                pipeline.addLast("lineEncoder", new LineEncoder(LineSeparator.UNIX,
                        CharsetUtil.UTF_8));  
                //设置 IdleStateHandler  函数 可以在userEventTriggered函数中获取读写超时以及总超时
                pipeline.addLast("ping", new IdleStateHandler(WRITE_WAIT_SECONDS,WRITE_WAIT_SECONDS, WRITE_WAIT_SECONDS, TimeUnit.SECONDS));
                // 客户端的逻辑
                pipeline.addLast("handler", mNettyClientHandler);
            }
        });
    }

    //开连接服务
    public void onStart() {
        new Thread() {
            @Override
            public void run() {
                connServer();
                super.run();
            }
        }.start();

    }
    //停止服务
    public void onStop() {
        isStop = true;
        if (ch != null && ch.isOpen()) {
            ch.close();
        }
        if (executorService != null) {
            executorService.shutdown();
        }
    }

//内部连接函数
    private void connServer() {
        Log.e(TAG, "connServer  ServerIP =  " + IStatic.ServerIP + "   ;tcpPort = " + IStatic.tcpPort);
        isStop = false;
        if (executorService != null) {
            executorService.shutdown();
        }
        //以固定延迟（时间）来反复进行重连，用于开始没有连接成功的情况
        executorService = Executors.newScheduledThreadPool(1);
        executorService.scheduleWithFixedDelay(new Runnable() {

            boolean isConnSucc = true;

            @Override
            public void run() {
                try {
                    //连接服务器
                    if (ch != null && ch.isOpen()) {
                        Log.e(TAG, " ch != null && ch.isOpen() ");
                        ch.close();
                    }
                    ch = b.connect(host, port).sync().channel();
                    // 此方法会阻塞
//                    ch.closeFuture().sync();
                    System.out.println("connect server finish");
                    Log.e(TAG, "connect server finish");
                } catch (Exception e) {
                    e.printStackTrace();
                    Log.e(TAG, e.toString());
                    isConnSucc = false;
                } finally {
                    System.out.println("executorService.shutdown  before");
                    Log.e(TAG, "executorService.shutdown  before isConnSucc = " + isConnSucc);

                    if (isConnSucc) {
                        if (executorService != null) {
                            executorService.shutdown();
                        }
                    }
                    System.out.println("executorService.shutdown after");
                    Log.e(TAG, "connect server finish isConnSucc = " + isConnSucc);
                }
            }
        }, RE_CONN_WAIT_SECONDS, RE_CONN_WAIT_SECONDS, TimeUnit.SECONDS);
    }

//根据NettyClientHandler 中回调得到的状态对长链接进行重练操作
    private ITCPStateListener mListener = new ITCPStateListener() {
        @Override
        public void online() {
        //由于在handler中判断的有一点问题，所以在此处重新判断一下断线后第一次上线的时候回调给main函数，以便能够提示用户连接成功
            if(!isOnline)
            {
                if (mStateListener != null)
                    mStateListener.online();
                new Thread(){
                    @Override
                    public void run() {
                        super.run();
//连接成功后将executorService 连接池关闭，以保证能够在重连的时候能够建立连接成功
                        if(!executorService.isShutdown())
                        {
                            executorService.shutdown();
                            Log.e(TAG,"executorService is  not Shutdown");
                        }else {
                           Log.e(TAG,"executorService is   Shutdown");
                        }
                    }
                }.start();
            }
            isOnline=true;

        }

        @Override
        public void offline() {
            if(isOnline)
            {
                if (mStateListener != null)
                    mStateListener.offline();
            }
            isOnline=false;

            if (!isStop) {
                new Thread() {
                    @Override
                    public void run() {
                        super.run();
                        try {
                            sleep(5 * 1000);
                            /*
                            * 重连
                            */
                            connServer();
                        } catch (InterruptedException e) {
                            e.printStackTrace();
                        }
                    }
                }.start();
            }

        }
    };

//发送数据
    public void sendMsg(String msg)
    {
        mNettyClientHandler.sendMsg(msg);
    }

//设置连接状态监听
    public void setStateListener(ITCPStateListener stateListener) {
        mStateListener = stateListener;
    }

//设置返回数据监听
    public void setInfoListener(ITCPInfoListener infoListener) {
        mNettyClientHandler.setInfoListener(infoListener);
    }
}
```

netty客户端的入口程序完成后接下来我们来完成更为主要的handler实现程序
```
@ChannelHandler.Sharable
public class NettyClientHandler extends SimpleChannelInboundHandler<String> {

    private ITCPStateListener mStateListener;
    private final int HEART_FRESH_TIME = 5000;//设置心跳间隔时间
    private boolean isonline = false;
    private String HEART_FRESH = "";
    private final String TAG = "NettyClientHandler";
    private ChannelHandlerContext mctx;
    private ITCPInfoListener mInfoListener;
    private SFresh mFresh = null;

    public NettyClientHandler(ITCPStateListener StateListener) {

        mStateListener = StateListener;
    }

    @Override
    public void userEventTriggered(ChannelHandlerContext ctx, Object evt) throws Exception {
        if (evt instanceof IdleStateEvent) {
            IdleStateEvent event = (IdleStateEvent) evt;
            if (event.state() == IdleState.READER_IDLE) {
                //读超时
                MyLogger.e(TAG, "===服务端=== （Reader_IDLE 读超时）");
                ctx.channel().close();
            } else if (event.state() == IdleState.WRITER_IDLE) {
                //写超时
                MyLogger.e(TAG, "===服务端=== （Reader_IDLE 写超时）");
                ctx.channel().close();
            } else if (event.state() == IdleState.ALL_IDLE) {
                //总超时
                MyLogger.e(TAG, "===服务端=== （ALL_IDLE 总超时）");
                ctx.channel().close();

            }
        }
    }

//通过handler机制能够保证只是接收到数据只是发送一次心跳数据，在试验中发现，在网络不好的条件下，
//反复重连的时候会产生多个心跳那么将会出现同一秒内可能发生多次心跳，所以在接收到发送心跳的指令
//之后首先移除以前的历史数据，保证只是发送一次
      private Handler  mHanler =  new Handler(){
        @Override 
        public void handleMessage(Message msg) { 
      
          mHanler .removeMessage(0);

           //此处用来发送心跳数据
 
        } 
};
    @Override
    protected void channelRead0(ChannelHandlerContext ctx, String message) throws Exception {
//测试中发现channelActive这个函数有的时候并不代表真的是连接成功了，那么我选择在收到数据的时候认为是连成功
        if (!isonline) {
            mStateListener.online();
            isonline = true;
        }
         String action = JSON.parseObject(message, JsonBean.class).action;
        if(action.equls("fresh")){
            //此处是我们的心跳数据，你们可以根据你们自己的方式进行修改
            mHanler .removeMessageDelay(0,5000);
//接收到心跳数据延迟5s后发送数据，这样能够保证如果没有收到心跳数据的时候将
//不会发送心跳那么在超时读取超时的时候会被回调，还有就是如果在发送过程中网络断开那么会在写入的超时函数中被回调
        }
        //接收到的数据，对数据进行处理，我们项目中主要是使用json数据解析，我使用的是fastjson 感觉速度很快，而且几乎不出错
    }


//连接成功
    @Override
    public void channelActive(final ChannelHandlerContext ctx) throws Exception {
        System.out.println("Client active ");
        MyLogger.e(TAG, "Client active ");
        this.mctx = ctx;
        super.channelActive(ctx);
        setHeartInfo(1);
//延迟2s后发送首次心跳数据或者登陆数据
        Observable.timer(2, TimeUnit.SECONDS)
                .subscribe(l->{
                    sendHeartData();
                },e->{
                    MyLogger.e(TAG, "===发送心跳异常=== e = "+e.toString());
                    ctx.channel().close();
                });
    }

    //断开连接函数，客户端主动close或者服务器断开连接的时候会回调此函数
    @Override
    public void channelInactive(ChannelHandlerContext ctx) throws Exception {
        System.out.println("Client close ");
        MyLogger.e(TAG, "Client close ");
        super.channelInactive(ctx);
        isonline = false;
        mctx.close();
        this.mctx = null;
        mHeartTimer.cancel();
        if (mStateListener != null)
            mStateListener.offline();
    }
   //发送数据
    public boolean sendMsg(String message) {
        if (mctx != null) {
            try {
                MyLogger.e(TAG, "sendMsg = " + mctx.isRemoved());
                MyLogger.e(TAG, "sendMsg = " + message);
                mctx.channel().writeAndFlush(message);
                return true;
            } catch (Exception e) {
                return false;
            }
        }
        return false;
    }


//发送心跳数据
    private void sendHeartData() {
        try {
            MyLogger.e(TAG, " HeartTask ctx = " + mctx);
            MyLogger.e(TAG, " HeartTask HEART_FRESH = " + HEART_FRESH);
            if (mctx != null) mctx.channel().writeAndFlush(HEART_FRESH);
            if (mFresh.getLogin() == 1) {
                setHeartInfo(0);
            }
        } catch (Exception e) {
            MyLogger.e(TAG, " sendHeartData e = " + e.toString());
        }

    }



//设置心跳数据
    private void setHeartInfo(int first) {
        if (mFresh == null) {
            mFresh = new SFresh();
            mFresh.setIp(IStatic.IP);
            mFresh.setMac(IStatic.MAC);
        }
        mFresh.setLogin(first);
        HEART_FRESH = JSON.toJSONString(mFresh);
    }

//设置接收到的数据监听函数
    public void setInfoListener(ITCPInfoListener infoListener) {
        mInfoListener = infoListener;
    }
}

```



首次写简书，也不知道如何写主要就是贴上代码，让大家能够帮助检查一下，以望能够有大神能够读到帮助改正其中可能会在的bug以及不足之处；还有就是强烈推荐大家使用rxjava 异步 机制，写出来的代码真的是很好看
