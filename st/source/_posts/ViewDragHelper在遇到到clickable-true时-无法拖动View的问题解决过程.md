---
title: ViewDragHelper在遇到到clickable=true时 无法拖动View的问题解决过程
date: 2018-08-28 15:51:11
tags:
    - Android 
    - ViewDragHelper
    - 拖动
---

ViewDragHelper 使用很简单，看到一篇文章说如果支持ViewDragHelper的ViewGroup的子View设置了clickable = true会导致无法拖动子View。但是解决办法是从ViewDragHelper源码出发覆写了，
当时觉得虽然也能解决问题，但是我觉得可以从事件分发的角度来解决。

于是我自定义了一个非常简单的Layout来验证这件事：
```java
public class NSLayout extends LinearLayout {

    private static final String TAG = "NSLayout";

    class DragCallBack extends ViewDragHelper.Callback {

        @Override
        public boolean tryCaptureView(@NonNull View child, int pointerId) {
            return true;
        }

        @Override
        public int clampViewPositionHorizontal(@NonNull View child, int left, int dx) {
            return Math.min(Math.max(0, left), getWidth() - child.getWidth());
        }

        @Override
        public int clampViewPositionVertical(@NonNull View child, int top, int dy) {
            return Math.min(Math.max(0, top), getHeight() - child.getHeight());
        }
        
    }

    private ViewDragHelper dragHelper;


    public NSLayout(Context context, @Nullable AttributeSet attrs) {
        super(context, attrs);
        dragHelper = ViewDragHelper.create(this, new DragCallBack());
    }


    @Override
    public boolean onInterceptTouchEvent(MotionEvent ev) {
        return dragHelper.shouldInterceptTouchEvent(ev);

    }

    @Override
    public boolean onTouchEvent(MotionEvent event) {
        dragHelper.processTouchEvent(event);
        return true;
    }
}
```
布局文件
```xml
     <com.viewdraghelper.NSLayout
        android:layout_width="match_parent"
        android:layout_height="match_parent">

        <View
            android:layout_width="80dp"
            android:layout_height="80dp"
            android:clickable="true"
            android:background="@color/colorPrimary"/>

    </com.viewdraghelper.NSLayout>
```
结果发现设置clickable= true确实是没法拖动这个View。原因是因为clickable = true会导致这个View在ACTION_DOWN事件发生的时候就把事件接手了，当然了，由于后续事件还是要先经过NSLayout的onInterceptTouchEvent也就是说还会走shouldInterceptTouchEvent。而前面说过可以通过覆写
ViewDragHelper.CallBack的getViewHorizontalDragRange和getViewVerticalDragRange任意一个方法返回大于0的数值就可以解决这个问题。这样能解决的道理很简单，当返回了大于0的数值后，最后在MOVE事件来临的时候会触发后续的逻辑将mDragState置为STATE_DRAGGING，导致最后shouldInterceptTouchEvent返回true，直接拦截了后续事件，View只能接收到一个Cancel事件结束。

那么从事件分发的角度来说，其实也可以解决，其实这个问题和RecyclerView的Item点击情况有点像，当DOWN事件发生，如果没有Move，直接UP就会点击Item,如果发生了Move就会滚动RecyclerView这中情形下的item 点击和这里的View设置了clickable = true接收了事件是一样的，RecyclerView的滚动和View被拖动是一样，可以一一对应起来。只不过RecyclerView可以处理好是自己内部处理得好。这里我们可以这样处理，当DOWN事件发生的时候，我们copy这个DOWN事件保存起来，如果后续有Move事件，那么说明是要拖动View，把之前保存的DOWN手动分发给ViewDragpHelper，走拖动逻辑。

修改之后的NSLayout代码只改变了一点：
```java
    private MotionEvent downEv;

    @Override
    public boolean onInterceptTouchEvent(MotionEvent ev) {

        if (ev.getAction() == MotionEvent.ACTION_DOWN) {
            downEv = MotionEvent.obtain(ev);
        } else if (ev.getAction() == MotionEvent.ACTION_MOVE) {
            if (downEv != null) {
                onTouchEvent(downEv);
                downEv = null;
            }
        }
        return dragHelper.shouldInterceptTouchEvent(ev);

    }
```
思路很简单，就是保存DOWN事件，后续发生Move，就认为是要拖动View，把之前保存的DOWN事件重放一遍，让ViewDragHelper认为是一次完整的事件流。
这样处理同样也可以解决问题。UltraPtr这个库里面就有很多地方像这样手动分发事件。