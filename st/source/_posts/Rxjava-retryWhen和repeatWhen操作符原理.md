---
title: Rxjava retryWhen和repeatWhen操作符原理
date: 2018-08-08 16:29:42
tags: 
    -Rxjava
    -Operator
---
### 契机
因为最近使用了mvvm，不再用mvp,并且大量使用RxJava 简化一些场景下的操作。以至发现了一个操作符retryWhen，搜了一些资料，几乎都是 一位叫DanLew的外国人写的一篇文章或者其译文。原文在这[ >> ](https://blog.danlew.net/2016/01/25/rxjavas-repeatwhen-and-retrywhen-explained/),思路清晰，知道了怎么用，一些关键的注意点，但就是没有分析具体原理和流程是怎样(但是看他提到的一些词，应该是搞明白了内部原理的)。痛定思痛————当然也是觉得这个操作符非常有意思，所以仔细研究一番。（说实话，我也是最近才觉得RxJava有些源码真的值得好好翻一翻）。本文基于rxjava 1.3.8。

### retryWhen和repeatWhen真的不一样吗
先看下retryWhen的方法：
    ```java
      public final Observable<T> retryWhen(final Func1<Observable<Throwable>,Observable> notificationHandler) {
        return OnSubscribeRedo.<T>retry(this, InternalObservableUtils.createRetryDematerializer(notificationHandler));
    }
    ```
    再看repeatWhen:
    ```java
     public final Observable<T> repeatWhen(final Func1<Observable< Void>, Observable> notificationHandler) {
        return OnSubscribeRedo.repeat(this, 
        InternalObservableUtils.createRepeatDematerializer(notificationHandler));
    }
    ```
 几乎可以认为是一样的，除了对传入的handler处理稍微不同以外。另外参数上有点小不一样，retryWhen的参数内的泛型是Observable<Throwable>,
 repeatWhen的参数泛型则是Observable<Void>。这和两者响应的事件不一样，retry是对错误响应，发生错误了该选择是否重试；repeat则是对完成事件响应，数据发射完之后是否重试，完成事件是没有数据的所以是Void。

不过我看到这个方法最大的两个疑惑是：为什么不是Func1<Throwable,Boolean>类型的参数，根据给的异常返回true false决定是否重试不是很合理吗？？然后马上反应过来，返回Observable会更灵活，如果重试的逻辑很复杂，单纯根据一个bool 的true or false来决定是否重试，是满足不了一些场景下的需要的。

 retryWhen方法上有一段注释：
 ```html
    Returns an Observable that emits the same values as the source observable with the exception of an {@code onError}. An {@code onError} notification from the source will result in the emission of a{@link Throwable} item to the Observable provided as an argument to the {@code notificationHandler}
    function. If that Observable calls {@code onComplete} or {@code onError} then {@code retry} will call {@code onCompleted} or {@code onError} on the child subscription. Otherwise, this Observable will resubscribe to the source Observable.
 ```
 具体含义就是，这个操作符会返回一个Observable(记作o1),o1会发射和源observable一样的数据（源observable可能会抛出异常）。当源Observable 发射错误事件时，会将这个错误传递给一个Observable(记作o2),而这个o2会作为参数传给notificationHandler。因为notificationHandler 返回的也是一个Observable(o3),如果o3 后续发射了complete或者error事件（其实就是调用了onComplete或者onError），那么会导致child subscription 也调用onComplete或者onError,结束整个流程，不然的话（也就是调用了onNext），那么将会重新订阅源Observable——————也就是再次激活源Oservable。

 翻译的有点啰嗦。简而言之就是，我用一个代理Subscriber去订阅源Obsevable，从源Observable获取数据，没有发生错误的情况下，就和一个普通正常的Observable的一样，数据发射完了就结束了。不同的是， 源头可能发生错误，抛出异常，针对这种情况，我们选择怎么处理。给我们的处理方式就是，我给你一个Observable<Throwable>,当源Observable发射错误事件的时候，下游想从源Observable重新尝试订阅（也就是retry的含义）。而repeatWhen则只是稍微不同，repeatWhen响应的是源Observable的complete事件，就是当数据发射完了，是否重新订阅，重复的从源Observable获取数据。

###  使用
```java
           
           retryCount = 0
            Observable.create(new Observable.OnSubscribe<Integer>() {
            public void call(Subscriber<? super Integer> subscriber) {
                subscriber.onNext(1 / 0);
                subscriber.onCompleted();
            }
        }).retryWhen(new Func1<Observable<? extends Throwable>, Observable<?>>() {
            public Observable<?> call(Observable<? extends Throwable> err) {
                return err
                        .flatMap(new Func1<Throwable, Observable<?>>() {
                            public Observable<?> call(Throwable throwable) {
                                System.out.println(throwable);
                                if (throwable instanceof ArithmeticException && retryCount < 3) {
                                    System.out.println("retry ++");
                                    retryCount++;
                                    return Observable.just("");
                                } else {
                                    System.out.println("no retry");
                                    return Observable.empty();
                                }
                            }
                        });

            }
        })
```
创建了一个必然抛出算术异常的Observable。重试的逻辑是超过三次就放弃重试。这里是直接Observable.just("")触发的重试，以及Observable.empty()结束整个流程

这样写是没问题的，但是既然是返回Observable，那么我直接 这样：
```java
    retryWhen(new Func1<Observable<? extends Throwable>, Observable<?>>() {
            public Observable<?> call(Observable<? extends Throwable> err) {
                                if (throwable instanceof ArithmeticException && retryCount < 3) {
                                    System.out.println("retry ++");
                                    retryCount++;
                                    return Observable.just("");
                                } else {
                                    System.out.println("no retry");
                                    return Observable.empty();
                                }
            }
        })
```

当然是不可以的，这样的话是返回一个和err无关的Observable，Observable.just("") 和Observable.empty()把事件发射完毕后，整个流程也就结束了，我们的订阅者也会取消订阅，不再接收消息。所以根本就是没有发挥这个操作符的效果。那么为什么？
看源码！
<!-- 那么问题来了，为什么这样写，就可以生效，就像官方说的那样，调用onNext就可以触发重试，onError或者onComplete就结束。 -->

### 源码分析

retryWhen方法实际返回的是这个
```java
  OnSubscribeRedo.<T>retry(this, InternalObservableUtils.createRetryDematerializer(notificationHandler));
```
 InternalObservableUtils.createRetryDematerializer(notificationHandler) 这句代码的实际返回的是
 一个 RetryNotificationDematerializer 作用是把Observable<Notification>转换成Observable<Throwable> ，然后传给我自己定义的notificationHandler

    继续看内部逻辑：
    

