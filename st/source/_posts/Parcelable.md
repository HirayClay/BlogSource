---
title: Parcelable
date: 2018-01-05 10:11:07
tags:
    - Android 
    - Parcelable
---
这里并不是要仔细说一遍Parcelable，而是看了一些Parcelable的国内博客，发现都是说怎么用。怎么用还用你教吗，，，官方文档就有例子，而是有个点，没有一篇博客说出来（也有可能我看的不仔细？）。
这个疑惑估计你也有过，在使用Parcelable 序列化和序列化的时候都是write** read**方法调用，但是发现没有如果有两个相同类型的值，比如有两个int要序列化，我们是不是要调用writeInt两次，然后反序列化的时候调用两次readInt，那么问题来了，反序列化的时候调用readInt怎么就知道是拿到的正确的值，而不会拿反了，毕竟两个int呢。
就这个不大不小的点，一直没看到，当然我也想过，write** readInt**操作的值的顺序必须一致，不然就会错
写了一小段代码跑了下，确实必须按照顺序来，因为底层就是个指针在挪动，挨个读取，比如取一个int的值，就会挪动4字节，所以顺序错了有时会拿到错的值，甚至拿不到正确的值（读到null直接crash）

```java
    var name: String = ""
    var nickName: String = ""
    var id: Int = 0

    constructor(parcel: Parcel) {
        id = parcel.readInt()
        nickName = parcel.readString()
        name = parcel.readString()
    }

    override fun writeToParcel(parcel: Parcel, flags: Int) {
        parcel.writeString(name)
        parcel.writeString(nickName)
        parcel.writeInt(id)
    }
```
然后直接崩了
![](https://raw.githubusercontent.com/HirayClay/draft/master/ParcelableCrash.png)

这里有篇歪果仁写的[博客](https://www.sitepoint.com/transfer-data-between-activities-with-android-parcelable/)，很详细

The End