---
title: MotionLayout
date: 2018-08-20 09:57:59
tags:
    - MotionLayout
    - 动画
---


### 传统动画的问题
以前的动画————比如View动画和属性动画我们其实无法真正的很好的控制动画，因为动画的时间是给定的，而不是根据用户的操作去表现给定的动画。
简单来说，传统的动画一旦运行起来我们就无法控制，我们能做的只是何时开始，何时结束，或者何时取消，对于已经运行中的动画我们无法控制。那么有些情况我们就没法做的更好，比如用户在滑动列表时，其他控件如何响应用户的滑动操作并做出对应的动画————当然我们并不是做不到，而是不太优雅，如果能够配置并定义好这些东西，那么再好不过。MotionLayout做的就是这些。

### MotionLayout实现折叠动画
以前为了实现一个不那么复杂的滑动折叠效果，需要CoordinatorLayout CollapsingToolBar的帮助才能做到。而且布局文件里面的内容显得有些过于冗长。下面使用MotionLayout实现下面这个效果。

<img src="https://raw.githubusercontent.com/HirayClay/draft/master/motionglayout_rotation.gif" width="200" height="400px"/>

布局十分简单：
```xml
<android.support.design.widget.CoordinatorLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:id="@+id/content"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:fitsSystemWindows="false"
    android:background="@color/contentBackground">

    <android.support.design.widget.AppBarLayout
        android:id="@+id/app_bar"
        android:layout_width="match_parent"
        android:layout_height="@dimen/app_bar_height"
        android:theme="@style/AppTheme.AppBarOverlay">

        <include layout="@layout/motion_09_coordinatorlayout_header"/>

    </android.support.design.widget.AppBarLayout>

    <include layout="@layout/content_scrolling" />

</android.support.design.widget.CoordinatorLayout>
````

motion_09_coordinatorlayout_header:
```xml
<com.google.androidstudio.motionlayoutexample.utils.CollapsibleToolbar xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    android:id="@+id/motionLayout"
    app:layoutDescription="@xml/scene_09"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:minHeight="50dp"
    android:fitsSystemWindows="false"
    app:layout_scrollFlags="scroll|enterAlways|snap|exitUntilCollapsed">

    <ImageView
        android:id="@+id/background"
        android:layout_width="match_parent"
        android:layout_height="200dp"
        android:background="@color/colorAccent"
        android:scaleType="centerCrop"
        android:src="@drawable/monterey"/>

    <TextView
        android:id="@+id/label"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:transformPivotX="0dp"
        android:transformPivotY="0dp"
        android:text="Monterey"
        android:textColor="#FFF"
        android:textSize="32dp" />

</com.google.androidstudio.motionlayoutexample.utils.CollapsibleToolbar>
```


没有任何的嵌套，只是简单利用了ConstraintLayout(MotiongLayout继承自ConstraintLayout)的特性把控件放到合适的位置。可能会奇怪，并没有看到任何与动画有关的东西，难道是用Java代码在控制吗？并不是，动画的入口实则在layoutDescription属性上，这个属性决定了MotiongLayout和ConstraintLayout的不同，这个属性的值指向一个xml文件：

```xml
    <MotionScene
    xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:motion="http://schemas.android.com/apk/res-auto">

    <Transition
        motion:constraintSetStart="@+id/start"
        motion:constraintSetEnd="@+id/end"
        motion:duration="1000"
        motion:interpolator="linear">
        <OnSwipe
            motion:touchAnchorId="@+id/background"
            motion:touchAnchorSide="bottom"
            motion:dragDirection="dragUp" />

        <ConstraintSet android:id="@+id/end">
            <Constraint
                android:id="@id/background"
                android:layout_width="match_parent"
                android:layout_height="match_parent"
                android:alpha="0.2"
                motion:layout_constraintBottom_toBottomOf="parent"/>
            <Constraint
                android:id="@id/label"
                android:layout_width="wrap_content"
                android:layout_height="wrap_content"
                android:layout_marginStart="8dp"
                android:layout_marginBottom="8dp"
                android:rotation="0.0"
                motion:layout_constraintBottom_toBottomOf="@+id/background"
                motion:layout_constraintStart_toStartOf="parent" />
        </ConstraintSet>

        <ConstraintSet android:id="@+id/start">
            <Constraint
                android:id="@+id/background"
                android:layout_width="match_parent"
                android:layout_height="match_parent"
                android:alpha="1.0"
                motion:layout_constraintBottom_toBottomOf="parent"/>
            <Constraint
                android:id="@+id/label"
                android:layout_width="wrap_content"
                android:layout_height="wrap_content"
                android:rotation="-90.0"
                motion:layout_constraintBottom_toBottomOf="@+id/background"
                motion:layout_constraintStart_toStartOf="parent"/>
        </ConstraintSet>
    </Transition>

</MotionScene>
```
代码其实不多，最外边是MotiongScene，包含了所有动画需要的元素。Transition是:

| Attributes | Description |
| :------ | ------: |
| android:id | The id of the Transition |
|constraintSetStart| ConstraintSet to be used as the start constraints or a layout file to get the constraint from
|constraintSetEnd | ConstraintSet to be used as the end constraints or a layout file to get the constraint from
|interpolator |
|duration|
|staggered | float: a quick way to stagger the objects moving|
|&lt;OnSwipe&gt; | Adds support for touch handling (optional) |
|&lt;OnClick&gt;| Adds support for triggering transition (optional)
|&lt;KeyFrameSet&gt;|Describes a set of Key object which modify the animation between constraint sets.|