---
title: ProducerArbiter
date: 2018-08-16 14:11:56
tags: 
    - RxJava 
    - Producer
    - ProducerArbiter
---


### Producer并不是一层不变
前面说过Subscriber可以通过setProducer设置Producer，而且这个方法也是支持并发调用的，意味着我们的Producer是可能不断变化的————换个说法————我们的数据源可能会改变。关于Producer的逻辑看这个{% post_link RxJava中的-Producer %}。但是那篇文章没有提到的是，如果我们的Producer在中途改变了，会发生什么情况。举个例子(这个例子实际来自RxJava开发者的一篇进阶[博文](https://blog.piasy.com/AdvancedRxJava/2016/07/02/operator-concurrency-primitives-7/))。这个系列的文章真的非常不错，能够看到很多我们不会注意到的问题，也可以窥见早期RxJava实现上的一些影子，对于进一步理解Rx帮助很大。

给定两个Observable,希望观察第一个Observable，当一个Observable结束之后，观察第二个Observable，第二个Observable结束了，那么才会结束,自定义了一个TheObserve操作符：
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
结果输出了1到30一共30个数字，并不像预想的先输出1-10 然后输出剩下的11-20 一共20个数字。
我们知道Subscriber可以向上游请求数据，如果没有设置Producer，内部有个requested计数器会将这个请求先保存起来，待到调用了setProducer的时候会把请求传递到上游。
而问题在于这个计数器只负责累计计数，并不会在请求已经到达的时候，减去已经完成的请求。那么在这个例子导致的问题就是第一个range(1,10)发出了10个数后，紧接着数据源变成了range(11,90),此时这个range(11,90)依然得到这个 数值为20的requested计数器 ，所以依然发射了 20个数字，所以最后导致一共产生了30个数据。那么应对这种数据源发生变化的场景我们需要用到ProducerArbiter。
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
                p.request(n);//3
            }

            emitLoop();//4
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

3) 这里必须详细解释为什么是请求n，而不是requested。因为下游发出数据请求的时候，这些请求积累是发生在每个Producer上的，此刻的n实际是下游在向currentProducer请求，而requested是之前积累的，可能是之前的missedProducer上积累的请求，更加详细的逻辑会在emitLoop中。

4) emitLoop的作用实际就是真正调用实际的Producer 请求数据的过程。代码能够走到这一步说明emitting = false,此时没有来自其他线程的数据请求，执行emitLoop漏循环处理已经积累的请求。

#### setProducer方法

```java
    public void setProducer(Producer newProducer) {
        synchronized (this) {
            if (emitting) {
                missedProducer = newProducer == null ? NULL_PRODUCER : newProducer;//1
                return;
            }
            emitting = true;
        }
        boolean skipFinal = false;
        try {
            currentProducer = newProducer;
            if (newProducer != null) {
                newProducer.request(requested);//2
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
1) 如果传入的newProducer是null则会把missedProducer设置成NULL_PRODUCER，可以说是一个标记，在emitLoop中作为判断条件使用。
2) 请求了requested个，貌似和前面request方法的<strong>//3</strong>处有点不同，这是因为如果切换了Producer，那么之前的Producer没有生产的数据，再也不会有机会产生，所以这些
积累下的数据就应该全部交给新的Prodcuer来产生。

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

            if (r != Long.MAX_VALUE) {//1
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
            if (localProducer != null) {//2
                if (localProducer == NULL_PRODUCER) {
                    currentProducer = null;
                } else {
                    currentProducer = localProducer;
                    localProducer.request(r);
                }
            } else {//3
                Producer p = currentProducer;
                if (p != null && localRequested != 0L) {
                    p.request(localRequested);
                }
            }
        }
    }
```
emitLoop方法比较长，直接看真的容易看不懂，我也是最开始卡在这里很久，后来发现最好是用单一Producer的角度分析，从简化的角度理解，如果此时一直都只有一个Producer,不存在Producer切换，那么可以认为此时的ProducerArbiter实际是一个普通的Producer，这里的emitLoop方法实际上处理的只是missedRequested，实际上和RangeProducer(以RangeProducer为例)的背压处理情况下的slowPath方法极度相似————不停的处理积累的请求直到把积累的请求完全处理完。

此时再来看emitLoop，其实只是多了Producer切换的情况，其实如果我们把Producer也看作是一种数据————没错！我们把Producer也看成是request一样，好比某处下游正在请求数据一样，只不过这个请求是请求切换Producer。对应请求数据，我们是寻找合适的Producer把请求发出去，对应Producer我们则应该是寻找合适的数据去发射。 此时我们再来看emitLoop的逻辑，退出漏循环的条件是所有的请求都处理完毕(准确说是：积累的请求处理完、该切换的Producer切换完毕以及累计处理完的事件计数完毕)，这里很容易理解。

1) 整个if语句块实际是进行已经消费的事件进行减法计数，减去已经消费的事件数

2) 如果localProducer（实际是missedProducer）不为空，表示存在Producer切换，不过还得继续判断（因为切换的时候可能是传的一个为null的Producer）,如果localProducer == NULL_PRODUCER实际上就是把当前的Producer设置成null了也就是不设置数据源，不然的话 切换数据源，并且把累计的请求处理完。


至此整个ProducerArbiter分析完毕，但是ProducerArbiter存在的问题是，没有保证事件是串行发射的，举个例子，此时前面的request正在被Producer A执行，正在发射事件给subscriber，但是紧接着切换了Producer B,此时Producer B也开始给Subscriber发射事件，这样导致的结果就是事件发射是并行的，必然会发生问题，这也显然不是我们想要的。其实我们这时候就需要ProducerObserverArbiter，看名字就是知道它还是个观察者。
