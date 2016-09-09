---
title: RxJava实现事件总线 Rxbus代替eventbus 减少库的使用
date:  2016-09-01 14:07:30
tags: Android
categories: Android
---
什么是Eventbus
EventBus定义：是一个发布 / 订阅的事件总线。 这么说应该包含4个成分：发布者，订阅者，事件，总线。 那么这四者的关系是什么呢？ 很明显：订阅者订阅事件到总线，发送者发布事件。
总结一下就是：我订阅你，你遇到事情了，发送事件，或者理解为更新动态，我就收到消息。
常用的地方
<!--more-->
Eventbus和Rxbus常用于组件间信息的交换与通知，避免采用广播以及使用一大堆接口来实现。
使用的地方以本次项目来举例： 一个商城界面，包含一个RecyclerView和LinearLayout，LinearLayout中是一个购物篮信息，也就是美团那种。当点击RecyclerView中的按钮时，商品被添加，LinearLayout中的商品总价应该发生变化。而这时候就到了使用Eventbus或者Rxbus的时候了。 为了增加商品总价，常见的方法有这几种： 1. 在创建adapter的时候将LinearLayout的对象一并传入，以此可以更改LinearLayout中的TextView 2. 设置广播事件。添加商品-》发送广播-》处理广播 3. 设置接口。添加商品-》触发接口 4. 使用观察者模式。也就是Eventbus以及Rxbus实现的功能。
以上部分抄自一个网友，有的时候知道怎么回事就是写不出来；好坑,不多说了直接上代码
```
public class RxBus {
    private static volatile RxBus defaultInstance;
    // 主题
    private final Subject bus;
    // PublishSubject只会把在订阅发生的时间点之后来自原始Observable的数据发射给观察者
    public RxBus() {
        bus = new SerializedSubject<>(PublishSubject.create());
    }
    // 单例RxBus
    public static RxBus getDefault() {
        RxBus rxBus = defaultInstance;
        if (defaultInstance == null) {
            synchronized (RxBus.class) {
                rxBus = defaultInstance;
                if (defaultInstance == null) {
                    rxBus = new RxBus();
                    defaultInstance = rxBus;
                }
            }
        }
        return rxBus;
    }
    // 提供了一个新的事件
    public void post (Object o) {
        bus.onNext(o);
    }
    // 根据传递的 eventType 类型返回特定类型(eventType)的 被观察者
    public <T> Observable<T> toObserverable (Class<T> eventType) {
        return bus.ofType(eventType);
    }
}


```

自定义event 实现数据拆分
```
public class RxEvent {
    public int reciveType;
    public int eventType;
    public String eventAction;
    public  Object event;

    public RxEvent() {
    }

    /**
     * RxBus 事件
     * @param reciveType 接收者类型
     * @param eventType 事件类型
     * @param eventAction 事件Action
     * @param event       时间
     */
    public RxEvent(int reciveType, int eventType, String eventAction, Object event) {
        this.reciveType = reciveType;
        this.eventType = eventType;
        this.eventAction = eventAction;
        this.event = event;
    }



    public Object getEvent() {
        return event;
    }

}

```
在项目中的使用,注意要在activity或者是fragment的start中注册Subscription观察者事件，并且在onDestroy中将解除注销事件，在android中使用过程中可以结合Rxandroid一起使用，
```
Subscription rxMainBus = RxBus.getDefault().toObserverable(RxEvent.class)
                .filter(rxEvent -> {
		//此处可以通过Rxjava的filter过滤函数对数据进行过滤，从而得到自己想要的数据
                    if ((rxEvent.reciveType == IStatics.DATA_BROADCAST || 
                  rxEvent.reciveType ==IStatics.DATA_ALL) && (rxEvent.eventType == 
                  IStatics.EVENT_MAIN || rxEvent.eventType ==IStatics.EVENT_ALL_THREAD)) {
                        return true;
                    }
                    return false;
                })
              //.observeOn(Schedulers.computation())//可以设置为子线程中接收数据
                .observeOn(AndroidSchedulers.mainThread())//设置为主线程接收数据
                .subscribe(new Action1<RxEvent>() {
                               @Override
                               public void call(RxEvent rxEvent) {
                                   switch (rxEvent.eventAction) {
                                      //此处可以根据事件类型分析数据，一对应不同的操作

                                   }
                               }
                           },
                        new Action1<Throwable>() {
                            @Override
                            public void call(Throwable throwable) {
                                // TODO: 处理异常
                            }
                        });

```

特别注意的是一定要在onDestroy中解除事件的注销，以保证不出现内存泄漏
```
if (rxMainBus != null && !rxMainBus.isUnsubscribed()) {    rxMainBus.unsubscribe();}
```
