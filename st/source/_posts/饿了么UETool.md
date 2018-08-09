---
title: 饿了么UETool
date: 2018-06-05 15:35:05
tags:
--- 饿了么 UETool UI


饿了么最近开源一个界面调试工具————UETool,官方介绍是
UETool 是一个各方人员（设计师、程序员、测试）都可以使用的调试工具。它可以作用于任何显示在屏幕上的 view，比如 Activity/Fragment/Dialog/PopupWindow 等等。

目前 UETool 提供以下功能：

- 移动屏幕上的任意 view，如果重复选中一个 view，将会选中其父 view
- 查看/修改常用控件的属性，比如修改 TextView 的文本内容、文本大小、文本颜色等等
- 如果你的项目里正在使用 Fresco 的 DraweeView 来呈现图片，那么 UETool 将会提供更多的属性比如图片 URI、默认占位图、圆角大小等等
- 你可以很轻松的定制任何 view 的属性，比如你想查看一些额外的业务参数
- 有的时候 UETool 为你选中的 view 并不是你想要的，你可以选择打开 ValidView，然后选中你需要的 View
- 显示两个 view 的相对位置关系
- 显示网格栅栏，方便查看控件是否对齐

当然要知道实现的原理，先把项目克隆下来一个一个分析

### 如何做到页面的控件选取

并没有给每个View设置点击，不过当你开启捕捉控件的时候，点击View就可以查看View的所有属性。第一想法应该是在进入页面的时候可能已经把所有View的位置信息保存下来了，点击的时候获取点击的位置，然后一个个匹配而已。从代码里找的答案确实也差不多————记录View是在点击“捕捉控件”时，而不是进入页面的时候，其他猜想是对的。
点击“捕捉控件”会启动一个TransparentActivity页面，顾名思义是个个透明的页面，有几句代码：
```java
  EditAttrLayout editAttrLayout = new EditAttrLayout(this);
                editAttrLayout.setOnDragListener(new EditAttrLayout.OnDragListener() {
                    @Override
                    public void showOffset(String offsetContent) {
                        board.updateInfo(offsetContent);
                    }
                });
                vContainer.addView(editAttrLayout);
```
上面几句代码意思是在这个透明页面上加入了一个EditAttrLayout的ViewGroup
进入EditAttrLayout，发现是继承CollectViewsLayout，重写了onDraw 和onTouchEvent方法，onDraw的作用就是给选取的View画上了边框，onTouchEvent里面主要是一个IMode类型的变量mode在处理手MOVE和UP事件。IMODE有两种实现，一个是ShowMode,也是mode默认的MODE，另外一个是MoveMode，明显是处理View移动的(点击捕捉控件，点击控件后在弹出窗口的move选项勾选就切换成MoveMode，控件就会跟随手指移动)。核心的逻辑还是要进入CollectViewsLayout里看,CollectViewsLayout顾名思义就是为了收集保存View。在onAttachedToWindow 方法遍历了所有的view，并且保存在elements里面，并且每个View被包装成了一个Element
```java
 class Element{
    private View view;
    private Rect originRect = new Rect();
    private Rect rect = new Rect();
    private int[] location = new int[2];
    private Element parentElement;
 }
```
在遍历的过程中，遇到要过滤的控件或者是控件透明的 都会结束掉此次遍历（本身是透明的，也没必要遍历子view了）

### 查看属性弹窗
当开启捕捉控件，捕捉到控件后，会弹出Dialog显示控件的诸多属性。在前面说到的EditAttrLayout涉及到手势的判断，手指抬起的时候show了这个dialog，进入到dialog里面，实际上就是根据捕捉的view包装的element 提取出了所有属性。
```java
    public void notifyDataSetChanged(Element element) {
            items.clear();
            for (String attrsProvider : UETool.getInstance().getAttrsProvider()) {
                try {
                    IAttrs attrs = (IAttrs) Class.forName(attrsProvider).newInstance();
                    items.addAll(attrs.getAttrs(element));
                } catch (Exception e) {
                    e.printStackTrace();
                }
            }
            notifyDataSetChanged();
        }
```