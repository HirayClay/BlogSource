---
title: Rxjava retryWhen和repeatWhen操作符原理
date: 2018-08-08 16:29:42
tags: 
    -Rxjava
    -Operator
---
### 契机
因为最近使用了mvvm，不再用mvp,并且大量使用RxJava 简化一些场景下的操作。以至发现了一个操作符retryWhen，搜了一些资料，几乎都是 一位叫DanLew的外国人写的一篇文章或者其译文。原文在这[ >> ](https://blog.danlew.net/2016/01/25/rxjavas-repeatwhen-and-retrywhen-explained/),思路清晰，知道了怎么用，一些关键的注意点，但就是没有分析具体原理和流程是怎样。痛定思痛————当然也是觉得这个操作符非常有意思，所以仔细研究一番。（说实话，我也是最近才觉得RxJava有些源码真的值得好好翻一翻）。本文基于rxjava 1.3.8。

### retryWhen和repeatWhen真的不一样吗
先看下retryWhen的方法：
    ```java
      public final Observable<T> retryWhen(final Func1<? super Observable<? extends Throwable>, ? extends Observable<?>> notificationHandler) {
        return OnSubscribeRedo.<T>retry(this, InternalObservableUtils.createRetryDematerializer(notificationHandler));
    }
    ```
其实我看到这个方法最大的两个疑惑是：为什么不是Func1<Throwable,Boolean>类型的参数，根据给的异常返回true false决定是否重试不是很合理吗？？
 retryWhen方法上有一段注释：
 ```html
    Returns an Observable that emits the same values as the source observable with the exception of an {@code onError}. An {@code onError} notification from the source will result in the emission of a{@link Throwable} item to the Observable provided as an argument to the {@code notificationHandler}
    function. If that Observable calls {@code onComplete} or {@code onError} then {@code retry} will call {@code onCompleted} or {@code onError} on the child subscription. Otherwise, this Observable will resubscribe to the source Observable.
 ```
 具体含义就是，这个操作符会返回一个Observable(记作o1),o1会发射和源observable一样的数据（源observable可能会抛出异常）。当源Observable 发射错误事件时，会将这个错误传递给一个Observable(记作o2),而这个o2会作为参数传给notificationHandler。因为notificationHandler 返回的也是一个Observable(o3),如果o3 后续给其订阅者发射了complete或者error事件（其实就是调用了onComplete或者onError），那么会导致child subscription 也调用onComplete或者onError,结束整个流程，不然的话（也就是调用了onNext），那么将会重新订阅源Observable——————也就是再次激活源Oservable。

 翻译的有点啰嗦。简而言之就是，我用一个类似代理的东西去订阅源Obsevable，从源Observable获取数据，没有发生错误的情况下，就和一个普通正常的Observable的一样，数据发射完了就结束了。不同的是， 源头可能发生错误，抛出异常，针对这种情况，我们选择怎么处理。给我们的处理方式就是，我给你一个Observable<Throwable> ,当源Observable发射错误事件的时候，下游想从源Observable重新尝试订阅（也就是retry的含义）。而repeatWhen则只是稍微不同，repeatWhen响应的是源Observable的complete事件，就是当数据发射完了，是否重新订阅，重复的从源Observable获取数据。
