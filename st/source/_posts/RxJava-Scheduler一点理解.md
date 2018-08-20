---
title: RxJava Scheduler一点理解
date: 2018-08-15 17:32:59
tags:
    - RxJava
    - Scheduler
    - 线程调度
---

### subscribeOn
通过subscribeOn 和 ObserveOn 两个方法rxjava可以灵活的指定任务执行的线程和指定收到事件的线程
直接看源码：
```java
        public final Observable<T> subscribeOn(Scheduler scheduler, boolean requestOn) {
        if (this instanceof ScalarSynchronousObservable) {
            return ((ScalarSynchronousObservable<T>)this).scalarScheduleOn(scheduler);
        }
        return unsafeCreate(new OperatorSubscribeOn<T>(this, scheduler, requestOn));
    }
```
使用了lift操作，看一下OperatorSubscribeOn这个操作符,重点是call方法：

```java
   @Override
    public void call(final Subscriber<? super T> subscriber) {
        final Worker inner = scheduler.createWorker();

        SubscribeOnSubscriber<T> parent = new SubscribeOnSubscriber<T>(subscriber, requestOn, inner, source);
        subscriber.add(parent);
        subscriber.add(inner);

        inner.schedule(parent);
    }
```
这个SubscribeOnSubscriber类型的parent实际是个Action0,然后inner.schedule(parent)直接让这个任务在指定的线程执行了，
然后事件到来的时候，简单调用这个call方法参数传来的Subscriber 调用一下onNext onError onComplete就完了。嗯，其实也很简单，一下就懂了，就是把source Observable的订阅放到一个指定的Scheduler中执行，然后事件也会在所在的Scheduler中发出来。其实可以得出一个结论：无论subscribeOn调用多少次，调度都只会在第一次调用subscribeOn指定的线程中执行。比如一个Observable 调用了subscribeOn(Schedulers.io).subscribeOn(Schedulers.compution()),其实这样做的效果可以拆分看，第一次调用subscribeOn把调度指定在io线程，那么后面的调度就是想把  '把调度指定在io' 的调度指定在computation线程，换个说法，我有一个操作是要指定调度在io线程 ，只不过我把这个操作放在了computation线程去执行而已，有种脱了什么放什么的多余+_+(实在找不出什么恰当的比喻了)。

我们以一个小栗子来看一下整个流程：
<pre><code>
 val observable = Observable.create(Observable.OnSubscribe<Int> { subscriber ->
            Log.i("source===>", Thread.currentThread().name)
            for (i in 0..0) {
                subscriber.onNext(i)
            }

            subscriber.onCompleted()
        })
 val map = observable
                .observeOn(Schedulers.computation())
                .subscribeOn(Schedulers.newThread())
                .map(Func1<Int, String> { integer ->
                    Log.i("map===>", Thread.currentThread().name)
                    integer!!.toString()
                })


 map.observeOn(Schedulers.newThread())
                .subscribe(Action1<String> { s -> Log.i("onNext===>", Thread.currentThread().name) })
</code></pre>

首先创建一个简单的发射一个数字的Observable，然后调用map操作转换成string 然后打印出来。注意subscribeOn 和ObserveOn的位置，我们是先observeOn 然后subscribeOn
打印结果：

<pre>
    <code>
    source===>: RxNewThreadScheduler-2
    map===>: RxComputationScheduler-1
    onNext===>: RxNewThreadScheduler-1
    </code>
</pre>

可以看到最后的onNext 调用并没有像预想的那样 发生在observeOn指定的computation 线程中，而是subscribeOn指定的创建的新线程中。
其实结合前面的subscribeOn的源码分析可以知道，调用subscribeOn之后的所有操作其实都会在subscribeOn 指定的线程中，这也是为什么map 和subscribe 两个操作都发生在RxNewThreadScheduler的原因。

### observeOn
ObserveOn和SubscribeOn不太一样，subscribeOn方法是放在哪儿都可以调用多次也只有第一次调用的效果。ObserveOn也可以多次调用，但是每次都会生效，要理解清楚还得看代码，直接进入OperatorObserveOn操作符的call方法
<pre><code>
       @Override
    public Subscriber<? super T> call(Subscriber<? super T> child) {
        if (scheduler instanceof ImmediateScheduler) {
            // avoid overhead, execute directly
            return child;
        } else if (scheduler instanceof TrampolineScheduler) {
            // avoid overhead, execute directly
            return child;
        } else {
            ObserveOnSubscriber<T> parent = new ObserveOnSubscriber<T>(scheduler, child, delayError, bufferSize);
            parent.init();
            return parent;
        }
    }

</code></pre>
不同的是 observeOn 的call方法是有返回值的，对于很多call方法有返回值的操作符，其实都可以认作是代理模式。包装了下游的subscriber，生成新的subscriber,然后让这个新的subscriber订阅上游observable,自己内部先处理，然后转发给下游的subscriber,达到代理的目的。我们再看一下这个ObserveOnSubscriber:

1)
<pre><code>
         @Override
        public void onNext(final T t) {
            if (isUnsubscribed() || finished) {
                return;
            }
            if (!queue.offer(NotificationLite.next(t))) {
                onError(new MissingBackpressureException());
                return;
            }
            schedule();
        }
</code></pre>

当接收到上游发来的事件时，调用onNext,然后先存入队列，存入成功，则会执行schedule方法进行调度，schedule方法了解一下：

2)
<pre><code>
            protected void schedule() {
            if (counter.getAndIncrement() == 0) {
                recursiveScheduler.schedule(this);
            }
        }
</code></pre>
非常简单的一句，如果当前没有任务（发射事件）调度，那么开始，并且把计数器加一，这里实际是对多线程的考虑，同一时刻只能有一个线程进行调度 。结合上面的onNext方法一起看就是——————如果有任务，先放入队列，放不进去就调用onError，不然就调度，而调度的话又必须满足当前没有其他线程在调度。调度任务会执行ObserveOnSubscriber的call方法,这样就实现了线程切换。call方法内部就是让传进来的child subscriber接收上游的事件，到这里我们可以得出结论ObserveOn 其实只对ObserveOn调用之后的操作生效。举个例子(kotlin编写)：
<pre><code>
 val ob = Observable.create(Observable.OnSubscribe<Int> { t ->
            t.onNext(1)
            t.onCompleted()
        })
 ob.observeOn(Schedulers.io())
                .map { it->it.toString() }
                .observeOn(Schedulers.computation())
                .map { it-> it.toCharArray() }
                .subscribe {  }
</code></pre>
上游简单发射一个Int数字,第一次调用ObserveOn 那么对于这第一个ObserveOn的操作符而言call方法传入的subscriber是下游map 生成的MapSubscriber,所以第一个map的操作发生在io线程，当同理第二个map 也会发生在computation线程。其实到这里可以总结出来ObserveOn方法的作用其实就是将之后的操作调度ObserveOn指定的线程中执行。

3)
call 方法实现：
<pre><code>
     @Override
        public void call() {
            long missed = 1L;//能进入到call这个方法，说明进入前，说明只有一次调度（就是本次调度）
            long currentEmission = emitted;
            final Queue<Object> q = this.queue;
            final Subscriber<? super T> localChild = this.child;
            for (;;) {
                long requestAmount = requested.get();//获取下游的请求数量

                while (requestAmount != currentEmission) { //直到把下游的请求都发射完为止
                    boolean done = finished;
                    Object v = q.poll();
                    boolean empty = v == null;

                    if (checkTerminated(done, empty, localChild, q)) { //是否已经结束
                        return;
                    }

                    if (empty) {
                        break;
                    }

                    localChild.onNext(NotificationLite.<T>getValue(v));

                    currentEmission++;
                    if (currentEmission == limit) {
                        requestAmount = BackpressureUtils.produced(requested, currentEmission);
                        request(currentEmission);
                        currentEmission = 0L;
                    }
                }

                if (requestAmount == currentEmission) {
                    if (checkTerminated(finished, q.isEmpty(), localChild, q)) {
                        return;
                    }
                }

                emitted = currentEmission;
                missed = counter.addAndGet(-missed);//如果不是0，其他线程可能也在请求，导致新的多个调度任务，那么还得继续处理，记录调度任务数量，进入下次循环，直到任务全部处理完为止
                if (missed == 0L) {
                    break;
                }
            }
        }
</code></pre>
大致逻辑梳理一下：
- 首先声明了一个missed  = 1L记录需要调度的数量以及一个currentMission记录已经发射的事件数量；
- 一个for循环嵌套了一个while循环：
    1. while循环的作用就是发射事件，发射事件之前检查是否已经结束，结束的原因可能是已经结束了或者发生错误
    2. 每发射一个事件就计数