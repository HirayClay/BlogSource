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

当然是不可以的，这样的话是返回一个和err无关的Observable，Observable.just("") 和Observable.empty()把事件发射完毕后，整个流程也就结束了，我们的订阅者也会取消订阅，不再接收消息。所以根本就是没有发挥这个操作符的效果。那么为什么我不使用参数err 这个Observable就会无效？


### 源码分析
为了搞清楚为什么我们不使用传入的err这个Observable，就会导致retryWhen操作符失效，还是从方法本身入手

retryWhen方法实际返回的是这个
```java
  OnSubscribeRedo.<T>retry(this, InternalObservableUtils.createRetryDematerializer(notificationHandler));
```
 InternalObservableUtils.createRetryDematerializer(notificationHandler) 这句代码的实际返回的是
 一个 RetryNotificationDematerializer 作用是把Observable<Notification>转换成Observable<Throwable> ，然后传给我们自己定义的notificationHandler作为参数 然后返回一个新的Observable

==========go on 2108.8.10(今天又看了下，又有新的发现，之前的分析漏了点东西，不过也更加发觉 1.x版本的一些东西确实写的复杂了，结构不清晰)

继续看内部逻辑（省略一些过于细节的细节[不然语言又是罗哩罗嗦]，需要对源码比较熟悉，最好先看一遍）：
    
    ```java
   final Subject<Notification<?>, Notification<?>> terminals = BehaviorSubject.<Notification<?>>create().toSerialized();
   final Subscriber<Notification<?>> dummySubscriber = Subscribers.empty();
        // subscribe immediately so the last emission will be replayed to the next
        // subscriber (which is the one we care about)
   terminals.subscribe(dummySubscriber);
    ```
被 这三句话坑了很久，还让我看了BehaviorSubject的源码很久，发现这几句就是废话，有没有都行。看他的注释，意思是为了后面我们关心的subscriber能够获取到最近的那个事件，先订阅再说。（实际上最近的那个事件根本不存在）


```java
final Action0 subscribeToSource = new Action0() {
            @Override
            public void call() {
                Subscriber<T> terminalDelegatingSubscriber = new Subscriber<T>() {
                    boolean done;

                    @Override
                    public void onCompleted() {
                        if (!done) {
                            done = true;
                            unsubscribe();
                            terminals.onNext(Notification.createOnCompleted());
                        }
                    }

                    @Override
                    public void onError(Throwable e) {
                        if (!done) {
                            done = true;
                            unsubscribe();
                            terminals.onNext(Notification.createOnError(e));
                        }
                    }

                    @Override
                    public void onNext(T v) {
                        if (!done) {
                            child.onNext(v);
                            decrementConsumerCapacity();
                            arbiter.produced(1);
                        }
                    }
                };
            }
        };
```
精简了n多代码。这个subscribeToSource Action0的意思就是 新建了一个terminalDelegatingSubscriber订阅源Observable，就是一个代理订阅者，主要的目的是关注源Observable发出的complet 和error事件。为什么是关注complete和error呢，因为这前面说过retryWhen和repeatWhen本质上的逻辑是一样的，只不过retryWhen关注的是
error,repeatWhen关注是complete。complete 和error都被包装成了Notification，然后发射出去。

继续：
```java
final Observable<?> restarts = controlHandlerFunction.call(
                terminals.lift(new Operator<Notification<?>, Notification<?>>() {
                    @Override
                    public Subscriber<? super Notification<?>> call(final Subscriber<? super Notification<?>> filteredTerminals) {
                        return new Subscriber<Notification<?>>(filteredTerminals) {
                            @Override
                            public void onCompleted() {
                                filteredTerminals.onCompleted();
                            }

                            @Override
                            public void onError(Throwable e) {
                                filteredTerminals.onError(e);
                            }

                            @Override
                            public void onNext(Notification<?> t) {
                                if (t.isOnCompleted() && stopOnComplete) {
                                    filteredTerminals.onCompleted();
                                } else if (t.isOnError() && stopOnError) {
                                    filteredTerminals.onError(t.getThrowable());
                                } else {
                                    filteredTerminals.onNext(t);
                                }
                            }

                            @Override
                            public void setProducer(Producer producer) {
                                producer.request(Long.MAX_VALUE);
                            }
                        };
                    }
                }));
```

这里稍微复杂一点，从里往外看，里面是terminals.lift操作了一下，主要逻辑在返回的新的Subscribeder的onNext这里，也就是对前面发过来的
Notificaiton拦截判断处理一下。一共有三种情况（其实就是决定retry和repeat的地方）：

 1) t.isOnCompleted() && stopOnComplete 条件如果满足 对应的是retryWhen 结束不会重试
 2）t.isOnError() && stopOnError 条件满足 对应的是repeatWhen 结束，不会重复
 3）这种情况下我们针对retryWhen看，如果收到的是一个error事件，很明显前面两个条件都不会满足，直接来到这里，调用onNext,传递给下游

我们把这个terminal.lift之后得到的Observable记作 o1。之后这个o1会作为参数传递给我们的controlHandler
 。前面说过这个controlHandler不是我们自己定义的那个notificationHandler，而是经过包装之后的RetryNotificationDematerializer。所以controlHandler.call(o1)这句代码展开就是：
 ```java
    notificationHandler.call(o1.map(ERROR_EXTRACTOR))
 ```
 进一步展开：
 ```
    notificationHanlder.call(o1.map(new Func1<Notification<?>, Throwable> {
        @Override
        public Throwable call(Notification<?> t) {
            return t.getThrowable();
        }
    }))
 ```

所以我们定义的notificationHandler里的Observable<Throwable>就是这么来的。加上我们自己添加的逻辑之后返回一个名为restarts的Observable

最后，schedule了一个匿名的任务Action0:
```java
     worker.schedule(new Action0() {
            @Override
            public void call() {
                restarts.unsafeSubscribe(new Subscriber<Object>(child) {

                    @Override
                    public void onNext(Object t) {
                        if (!child.isUnsubscribed()) {
                           
                            if (consumerCapacity.get() > 0) {
                                worker.schedule(subscribeToSource);
                            } else {
                                resumeBoundary.compareAndSet(false, true);
                            }
                        }
                    }
                });
            }
        });
```
注意这个任务让restarts进行订阅（为了方便起见，我们把这个匿名的Subscriber记为s1），然后在收到next事件的时候，执行前面的subscribeToSource任务，也就是向源Observable发起订阅的任务。
那么这里的bug出现了，该怎么触发subscribeToSource这个任务呢？？？？？？？ 我们可以从后往前推，要触发这个subscribeToSource必须上游
调用onNext吐出事件来，而这里的上游就是restarts,也就是terminals terminals要吐出事件来，必须依赖源Observable吐出事件, 这里就形成了一个互相依赖的困境！！<!-- restarts是什么？是我们自己对传入的err那个Observable做变换以后返回的，所以如果我们做的变换逻辑里面如果发出了事件，那么才会导致这里 s1的onNext进行调用 -->
termials本身既是Observable也是一个Observer,所以terminals.lift 的目的是对自己发出的事件进行拦截 ，但是自己一开始并没有发出事件，然后把这个lift之后的ob传给我们定义的notificationHandler，所以——————最开始的事件还是必须由我们发出来！！

其实是child.setProducer那一行，会导致调用request(Long.MAX_VALUE)，导致subscribeToSource被调用了

