---
title: 自行处理Fling导致RecyclerView 滑动点击事件无效
date: 2018-04-02 09:00:44
tags:
    - RecyclerView 
    - Fling ItemClick
---
之前写过StackLayoutManager,一个自定义的LayoutManager，最近有同学说滑动之后item 点击无效，发现是滑动之后第一次点击无效，再次点击才能触发点击事件。第一反应觉得很诧异，要么就不触发，怎么还要点击两次才能触发的。带着疑问我调试了一下RecyclerView的onInterceptTouchEvent方法。结果是fling一次后点击item ，onInterceptTouchEvent方法返回了true，也就是事件被拦截了，就是导致Item无法点击的原因，拦截的条件是mScrollState == STATE_DRAGGING。但是明显现在应该处于STATE_IDLE状态，fling之后手指已经离开屏幕了。所以继续追踪，发现RecyclerView的fling事件内部自己有处理，而且fling完之后，会将mScrollState重置为STATE_IDLE，但是因为StackLayoutManager是使用的setOnFlingListener方式，导致没有重置状态，所以之后的第一次点击mScrollState仍然处于STATE_DRAGGING状态，所以被拦截了。但是我们是第二次点击又是可以的，所以肯定是第一次点击的某个地方将mScrollState重置为STATE_IDLE了，找了下，发现RecyclerView的onTouchEvent方法有这么一句
```java
                if (!((xvel != 0 || yvel != 0) && fling((int) xvel, (int) yvel))) {
                    setScrollState(SCROLL_STATE_IDLE);
                }
```
知道了前因后果之后，我们要做的就是自己处理fling之后应该将mScrollState重置为idle状态，但是RecylerView改变状态的方法并不对外暴露，所以最后我用了反射。