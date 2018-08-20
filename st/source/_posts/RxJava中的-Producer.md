---
title: RxJava中的 Producer
date: 2018-08-15 15:04:18
tags: 
    - RxJava
    - Producer 
    - 背压
---
本文基于 rxjava 1.3.8

### 背压
Producer本身是作为沟通上下游的一个接口，只有一个方法：
```java
void request(long n);
```
如果传入的n是Long.MAX_VALUE，表明是放弃背压，上游会有多少就生产多少。比如range()操作符内部的RangeProducer，如果遇到n = Long.MAX_VALUE,直接用fastPath方法，一口气把事件全部发射出去，反之是请求多少生产多少。

但是平时我们调用range操作符，好像都没有考虑过什么背压，都是这样：
```java
            Observable.range(1, 2)
                .subscribe(new Subscriber<Integer>() {
                    @Override
                    public void onCompleted() {
                        
                    }

                    @Override
                    public void onError(Throwable e) {

                    }

                    @Override
                    public void onNext(Integer integer) {
                            System.out.println(integer)
                    }
                });
```
也同样是一口气把数据全部给我了，似乎也没有调用request(Long.MAX_VALUE)

### Range操作符内部
进入到range操作符，实际内部只有这么一句：
```java
    public void call(final Subscriber<? super Integer> childSubscriber) {
        childSubscriber.setProducer(new RangeProducer(childSubscriber, startIndex, endIndex));
    }
```

RangeProducer这个Producer实际上就是一个fastPath和slowPath，分别对应没有背压和有背压的情况

那么实际childSubscriber是下游的subscriber，也就是我们调用subscriber的时候new 出来的那个,那么问题来了，到这里都没有什么问题，为何range操作符还是会一口气把数据全部发射出去呢？这里setProducer我们漏掉了。
看一下setProducer做了什么。setProducer方法是subscriber的方法，这里很好理解，下游的subscriber需要数据属于消费者，消费者需要通知生产者，所以给消费者一个方法设置合适（不同场景下有不同的生产者）的生产者。

```java
        public void setProducer(Producer p) {
        long toRequest;
        boolean passToSubscriber = false;
        synchronized (this) {
            toRequest = requested;
            producer = p;
            if (subscriber != null) {
                // middle operator ... we pass through unless a request has been made
                if (toRequest == NOT_SET) {
                    // we pass through to the next producer as nothing has been requested
                    passToSubscriber = true;
                }
            }
        }
        // do after releasing lock
        if (passToSubscriber) {
            subscriber.setProducer(producer);
        } else {
            // we execute the request with whatever has been requested (or Long.MAX_VALUE)
            if (toRequest == NOT_SET) {
                producer.request(Long.MAX_VALUE);
            } else {
                producer.request(toRequest);
            }
        }
    }
```
官方的注释其实已经是对这段代码的最简洁的解释了：如果Subscriber是通过Subscriber(Subscriber)或者Subscriber(Subscriber, boolean)方法设置了subscriber，那么就会对这个subscriber调用setProducer，反之如果没有设置subscriber并且已经有请求到来，那么就会直接调用producer.request(n)，其中n是已经累积的请求。

当然官方解释遗漏一个情况，那就是当没有设置subscriber，而且也没有请求到来的时候，那么就会调用producer.request(Long.MAX_VALUE),也就是让上游有多少就生产多少。

当然为了更容易明白这个方法的作用，总结一下：如果Subscriber自身设置了内部Subcriber那么会把这个producer设置给这个内部Subscriber;不然就请求上游（request(n))开始生产积累的数据,没有累积请求那就一口气全部生产完。有些时候我们要深刻理解前面半句，因为setProducer可能会形成很长的调用链:)。

回到最开始的具体的例子range(1,2).subscribe(subscriber),实际range操作内部那个childSubscriber 是我们new的那个subscriber（准确说是包装了之后的SafeSubscriber） 调用setProducer设置了RangeProducer。
由于我们new 的Subscriber是没有设置内置的subscriber的，那么最后实际会走到 “producer.request(Long.MAX_VALUE);”那句来，导致的结果就是RangeProducer 调用fastPath不考虑背压一口气全部生产完数据。

### 多个操作符的情况
这里我们有一个自定义的操作符，目的是过滤掉奇数，只要偶数：
```java
EvenFilter implements Observable.Operator<Integer, Integer> {
        public Subscriber<? super Integer> call(final Subscriber<? super Integer> child) {

            return new Subscriber<Integer>(child) {

                public void onNext(Integer t) {
                    if ((t & 1) == 0) {
                        child.onNext(t);
                    }
                }


                public void onError(Throwable e) {
                    child.onError(e);
                }


                public void onCompleted() {
                    child.onCompleted();
                }
            };
        }
    }
```
操作符本身很简单，在onNext那里简单过滤了一下，看上去很完美。我们再写个例子：

```java
     Observable.range(1, 2)
                .lift(new EvenFilter())
                .subscribe(new Subscriber<Integer>() {
                    @Override
                    public void onCompleted() {

                    }

                    @Override
                    public void onError(Throwable e) {

                    }

                    @Override
                    public void onNext(Integer integer) {
                        System.out.println(integer);
                    }
                });
```
逻辑也很简单，最后打印出了 2，符合预期。我们再加上一个操作符take()
```java
     Observable.range(1, 2)
                .lift(new EvenFilter())
                .take(1).subscribe(new Action1<Integer>() {
                    @Override
                    public void call(Integer integer) {
                        System.out.println(integer);
                    }
                })
```
然后就出问题了，什么都没有打印。有点反常，因为加上了take(1),逻辑上是没有改变的，应该会输出一样的结果。 一直到lift(new EvenFilter())操作都是可以正常往下游发射数据的，那么为什么只是加入了take会变得奇怪导致没有数据打印。很明显问题出在take操作符上。

分析，必须给他分析————range操作符 childSubscriber.setProducer那句实际上childSubscriber是我们下游EventFilter call方法返回的Subscriber，由于EvenFilter 返回的这个Subscriber设置了child这个Subscriber，所以实际上还会调用这个child的setProducer
把RangeProuducer设置给这个child。而这个child实际又是下游take操作符内部返回的Subscriber ，take操作符返回的这个Subscriber的setProducer方法是这样的：
```java
     public void setProducer(final Producer producer) {
                child.setProducer(new Producer() {

                    // keeps track of requests up to maximum of `limit`
                    final AtomicLong requested = new AtomicLong(0);

                    @Override
                    public void request(long n) {
                        if (n > 0 && !completed) {
                            // because requests may happen concurrently use a CAS loop to
                            // ensure we only request as much as needed, no more no less
                            while (true) {
                                long r = requested.get();
                                long c = Math.min(n, limit - r);
                                if (c == 0) {
                                    break;
                                } else if (requested.compareAndSet(r, r + c)) {
                                    producer.request(c);
                                    break;
                                }
                            }
                        }
                    }
                });
            }
```
这里实际上是调用了下游的child Subscriber(就是我们最后new的Subscriber)setProducer，根据前面的总结，最后会调用这里这个匿名Producer的request方法，并且n = Long.MAX_VALUE,最后会走到producer.request(c)这句，而且c = limit,啊哈！！！！，因为我们是take(1) 所以limit = 1，上游的RangeProducer 生产一个数据之后就没了，没了！导致2不会被发射出来！

做一整个问题就出在take操作符这里，因为take认为自己只要一个数据，所以只向上游请求了一个数据，这其实非常符合take的逻辑，然而它的上游，可以认为此时take的上游是我们的EvenFilter，EvenFilter自身数据来自range，而不是对range发来的数据做缓存，然后根据下游的请求来发射偶数2。所以这里与其说是take的错误，不如说是我们的Operator实现不够完美。那么我们这里其实有三种解决办法，一种是简单的修改EvenFilter
- request(1)
```java
    public void onNext(Integer t) {
                    if ((t & 1) == 0) {
                        child.onNext(t);
                    }else request(1)
                }
```
这样达到的效果是，如果符合要求，发往下游，不然请求下一个数据，这样所有的数据都会被发射出来

- 使用filter操作符
或者是使用rxjava提供的操作符，因为我们的EvenFilter实在有点多次一举，只是简单的过滤偶数，可以使用filter操作符：
```java
    Observable.range(1, 2)
                .filter(new Func1<Integer, Boolean>() {
                    @Override
                    public Boolean call(Integer integer) {
                        return (integer & 1) == 0;
                    }
                })
                .take(1)
```
- 使用其余的构造方法返回Subscriber
由于我们的EvenFilter返回的Subscriber使用的是Subscriber(subscriber)构造方法，所以使得 会调用child.setProducer，这里我们使用空的构造方法：


```java
 return new Subscriber<Integer>(/*child*/) {

                public void onNext(Integer t) {
                    System.out.println("inner: " + t);
                    if ((t & 1) == 0) {
                        child.onNext(t);
                    }
                }


                public void onError(Throwable e) {
                    child.onError(e);
                }


                public void onCompleted() {
                    child.onCompleted();
                }
            };
```

为什么这样可以呢，因为这样就不会调用child的setProducer,而是返回的这个Subscriber的setProducer，根据前面总结的，由于这个Subscriber没有内置的Subscriber,最后会导致调用RangeProducer的 request(Long.MAX_VALUE),一口气生产全部的数据

其实这个例子暗含一个教训，我们在实现自己的操作符的时候尽量不要去使用Subscriber(subscriber)这个构造方法返回Subscriber给上游，因为导致的问题是我们操作符自己被跨越了，下游和上游单独在联系，跨越了操作符自己，导致有些问题不能按照上游到下游一条连贯的线去思考，容易出现一些反常识的bug。