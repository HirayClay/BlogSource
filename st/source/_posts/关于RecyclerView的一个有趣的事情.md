---
title: 关于RecyclerView的一个有趣的事情
date: 2018-01-19 15:22:22
tags: -RecyclerView 
      -Android
---
对于RecylcerView ，基本上第一印象就是View重用，但是真的明白怎么重用的吗，最近在写自定义LayoutManager,由此对RecylerView、LayoutManager、ItemAnimator整个之间的关系都比较的熟悉。不过回到标题上来，这个有趣的事情和RV的回收有关。

比如页面上此时显示了前六条Item（第六个Item没有显示全），那么你肯定觉得不把第六条Item全部划进来，第七条就不会调用onCreateViewHolder进行创建；但事实是当我只要向上稍微滑出去一点就会创建第七个Item，这是不是和对RV的回收重用的印象有些矛盾？是的，按照常理，我根本都没有滑出第七个Item，你就创建了，好像不太对。

最后我看了下源码，其实原因比较简单，得先贴一下滑动发生时候调用填充逻辑的方法代码（部分代码）：
```java
 int fill(RecyclerView.Recycler recycler, LayoutState layoutState,
            RecyclerView.State state, boolean stopOnFocusable) {
        int remainingSpace = layoutState.mAvailable + layoutState.mExtra;
        LayoutChunkResult layoutChunkResult = mLayoutChunkResult;
        while ((layoutState.mInfinite || remainingSpace > 0) && layoutState.hasMore(state)) {
            layoutChunkResult.resetInternal();
            layoutChunk(recycler, state, layoutState, layoutChunkResult);
            if (layoutChunkResult.mFinished) {
                break;
            }
            layoutState.mOffset += layoutChunkResult.mConsumed * layoutState.mLayoutDirection;
            if (layoutState.mScrollingOffset != LayoutState.SCROLLING_OFFSET_NaN) {
                layoutState.mScrollingOffset += layoutChunkResult.mConsumed;
                if (layoutState.mAvailable < 0) {
                    layoutState.mScrollingOffset += layoutState.mAvailable;
                }
                recycleByLayoutState(recycler, layoutState);
            }
        }

        return start - layoutState.mAvailable;
    }
```

因为当滑动发生的时候，填充的postion是从第七个Item开始的，所以第七个Item被创建了，但是呢，立马被 recycleByLayoutState(recycler, layoutState)这个方法给回收掉了，并且缓存了起来，毕竟第七个Item在屏幕外，所以被回收了，而且这个while跑了这一次就退出了，到了第七个Item真的出来的时候就直接从缓存里面取出来用了

Note："scrap" View指的是仍然有效可以直接拿来重用的VH，只是暂时脱离了RV。名字让人很误解。