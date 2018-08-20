---
title: ProducerArbiter
date: 2018-08-16 14:32:41
tags:
---

### Producer并不是一层不变
前面说过Subscriber可以通过setProducer设置Producer，而且这个方法也是支持并发调用的，意味着我们的Producer是可能不断变化的————换个说法————我们的数据源可能会改变。关于Producer的逻辑看这个{% post_link RxJava中的-Producer %}。但是那篇文章没有提到的是，如果我们的Producer在中途改变了，会发生什么情况。举个例子(这个例子实际来自RxJava开发者的一篇进阶[博文](https://blog.piasy.com/AdvancedRxJava/2016/07/02/operator-concurrency-primitives-7/))：

给定两个Observable,希望观察第一个Observable，当一个Observable结束之后，观察第二个Observable，第二个Observable结束了，那么才会结束,自定义了一个TheObserve操作符
```java
    public static final class ThenObserve<T> implements Observable.Operator<T, T> {
        final Observable<? extends T> other;
        public ThenObserve(Observable<? extends T> other) {
            this.other = other;
        }

        @Override
        public Subscriber<? super T> call(Subscriber<? super T> child) {
            Subscriber<T> parent = new Subscriber<T>(child, false) {
                @Override
                public void onNext(T t) {
                    child.onNext(t);
                }
                @Override
                public void onError(Throwable e) {
                    child.onError(e);
                }
                @Override
                public void onCompleted() {
                    other.unsafeSubscribe(child);
                }
            };
            child.add(parent);
            return parent;
        }
    }

    Observable<Integer> source = Observable
                .range(1, 10)
                .lift(new ThenObserve<>(Observable.range(11, 90)));

    TestSubscriber<Integer> ts = new TestSubscriber<>();
    ts.requestMore(20);

    source.subscribe(ts);

    ts.getOnNextEvents().forEach(System.out::println);
```
结果输出了1到30一共30个数字，并不像预想的先输出1-10 然后输出剩下的11-20 一共20个数字。问题出在Subscriber上。

我们知道Subscriber可以向上游请求数据，如果没有设置Producer，内部有个requested计数器会将这个请求先保存起来，待到调用了setProducer的时候会把请求传递到上游。
而问题在于这个计数器只负责累计计数，并不会在请求已经到达的时候，减去已经完成的请求。那么在这个例子导致的问题就是第一个range(1,10)发出了10个数后，紧接着数据源变成了range(11,90),此时这个range(11,90)依然得到这个 数值为20个requested ，所以依然发射了 20个数字，所以最后导致一共产生了30个数据。那么应对这种数据源发生变化的场景我们需要用到ProducerArbiter。
<!-- @import "[TOC]" {cmd="toc" depthFrom=1 depthTo=6 orderedList=false} -->



先修改这个例子，让ProducerArbiter正确的发挥作用，首先，ProducerArbiter的作用是作为中间枢纽，充当上下游的Producer，Producer发生改变对于上下游是透明的。

修改后：
```java
    public static final class ThenObserve<T> implements Observable.Operator<T, T> {
        final Observable<? extends T> other;

        public ThenObserve(Observable<? extends T> other) {
            this.other = other;
        }

        @Override
        public Subscriber<? super T> call(Subscriber<? super T> child) {
            ProducerArbiter arbiter = new ProducerArbiter();

            Subscriber<T> parent = new Subscriber<T>() {
                @Override
                public void onNext(T t) {
                    child.onNext(t);
                    arbiter.produced(1);
                }

                @Override
                public void onError(Throwable e) {
                    arbiter.setProducer(null);
                    child.onError(e);
                }

                @Override
                public void onCompleted() {
                    arbiter.setProducer(null);
                    //产生新的subscriber用来接收producer的改变
                    DelegatingSubscriber<T> subscriber = new DelegatingSubscriber<>(child, arbiter);
                    child.add(subscriber);
                    other.unsafeSubscribe(subscriber);
                }

                @Override
                public void setProducer(Producer p) {
                    arbiter.setProducer(p);
                }
            };
            child.add(parent);
            child.setProducer(arbiter);
            return parent;
        }
    }


    static class DelegatingSubscriber<T> extends Subscriber<T>{
        Subscriber<? super  T> actual;
        ProducerArbiter arbiter;


        public DelegatingSubscriber(Subscriber<? super  T> downstream,ProducerArbiter arbiter){
            this.actual = downstream;
            this.arbiter = arbiter;
        }
        @Override
        public void onCompleted() {
            arbiter.setProducer(null);
            this.actual.onCompleted();
        }

        @Override
        public void onError(Throwable e) {
            arbiter.setProducer(null);
            this.actual.onError(e);
        }

        @Override
        public void onNext(T o) {
            this.actual.onNext(o);
            arbiter.produced(1);
        }

        @Override
        public void setProducer(Producer p) {
           arbiter.setProducer(p);
        }
    }
```
从最后的实现看，child Subscriber是消费者，需要从第一个range Observable和other Observable两处获取数据，由于数据源是不稳定的，所以child实际上只设置了ProducerArbiter一个Producer，实际Producer的切换交给了ProducerArbiter，这样数据源切换对于child 来说是透明的。
这里具体这做了三件事情：给child 设置ProducerArbiter；ProducerArbiter在数据源发生改变时切换到其他Producer；数据产生时调用producer(n)更新计数。

### ProducerArbiter 实现原理
如果是我们来实现ProducerArbiter以此满足上面 例子中的需要，其实我们要做的就是每次请求到来的时候增加计数，当有事件被消耗的时候减少计数，同时支持串行访问，从这样的角度我们再去看ProducerArbiter的源码，可能就会比较容易理解。一共有四个方法：

- ```java public void request(long n) ```
- ```java public void setProducer(Producer newProducer) ```
- ```java  public void produced(long n) ```
- ```java  public void emitLoop() ```


 #### request方法：
```java
public void request(long n) {
        if (n < 0) {
            throw new IllegalArgumentException("n >= 0 required");
        }
        if (n == 0) {
            return;
        }
        synchronized (this) { 
            if (emitting) {
                missedRequested += n; //1
                return;
            }
            emitting = true;
        }
        boolean skipFinal = false; //2
        try {
            long r = requested;
            long u = r + n;
            if (u < 0) {
                u = Long.MAX_VALUE;
            }
            requested = u;

            Producer p = currentProducer;
            if (p != null) {
                p.request(n);
            }

            emitLoop();//3
            skipFinal = true;
        } finally {
            if (!skipFinal) {
                synchronized (this) {
                    emitting = false;
                }
            }
        }
    }
```
1) 用missedRequested保存没有处理的请求，因为并发访问的原因，此时可能有其他线程在调用，所以先保存起来。

2) skipFinal 实际上的作用是决定是否跳过最后的finally代码块（当然实际并不能跳过，finally代码块的作用只是为了预防emitLoop方法发生错误而没有安全的将emitting 置为false，导致一直阻塞后续的请求）。

3) emitLoop的作用实际就是真正调用实际的Producer 请求数据的过程。代码能够走到这一步说明emitting = false,此时没有来自其他线程的数据请求，执行emitLoop漏循环处理已经积累的请求。

#### setProducer方法

```java
    public void setProducer(Producer newProducer) {
        synchronized (this) {
            if (emitting) {//1
                missedProducer = newProducer == null ? NULL_PRODUCER : newProducer;
                return;
            }
            emitting = true;
        }
        boolean skipFinal = false;
        try {
            currentProducer = newProducer;
            if (newProducer != null) {//2
                newProducer.request(requested);
            }

            emitLoop();
            skipFinal = true;
        } finally {
            if (!skipFinal) {
                synchronized (this) {
                    emitting = false;
                }
            }
        }
    }
```

#### emitLoop方法
```java 
public void emitLoop() {
        for (;;) {
            long localRequested;
            long localProduced;
            Producer localProducer;
            synchronized (this) {
                localRequested = missedRequested;//1
                localProduced = missedProduced;
                localProducer = missedProducer;
                if (localRequested == 0L
                        && localProduced == 0L
                        && localProducer == null) {//2
                    emitting = false;
                    return;
                }
                missedRequested = 0L;//3
                missedProduced = 0L;
                missedProducer = null;
            }

            long r = requested;

            if (r != Long.MAX_VALUE) {
                long u = r + localRequested;
                if (u < 0 || u == Long.MAX_VALUE) {
                    r = Long.MAX_VALUE;
                    requested = r;
                } else {
                    long v = u - localProduced;
                    if (v < 0) {
                        throw new IllegalStateException("more produced than requested");
                    }
                    r = v;
                    requested = v;
                }
            }
            if (localProducer != null) {
                if (localProducer == NULL_PRODUCER) {
                    currentProducer = null;
                } else {
                    currentProducer = localProducer;
                    localProducer.request(r);
                }
            } else {
                Producer p = currentProducer;
                if (p != null && localRequested != 0L) {
                    p.request(localRequested);
                }
            }
        }
    }
```
？？emitLoop 实际使用underlying producer在发射数据？？ProducerArbiter实际是完善了Subscriber只能累加已经产生的请求，而不能减去已经生产的请求而导致同一个请求，在生产者变化的情况下，请求会被执行两次，而ProducerArbiter做了加法减法计数两项工作。