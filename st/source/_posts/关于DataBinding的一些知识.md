---
title: 关于DataBinding的一些知识
date: 2018-04-11 10:29:01
tags:
    - DataBinding
    - Android
---
Android的DataBinding出来很久了，但是因为我一直用的mvp，出来的时候就简单的 开启了enable =true这句代码弄了下搞了个简单的layout布局就没管了（好吧，貌似只能算单向绑定=_=）。然后最近想深入的看下DataBinding，比如自定义的控件怎么实现双向绑定,一步一步来吧，其实东西真的不多，不过很强大,基本都是注解。

###简单的绑定
先建立一个Worker类：
```java
    public class Worker {

    public int workerId;
    public String name;
    public int wage;
    public String photoUrl;
    public int photoId;

    public Worker() {
    }


    public Worker(int workerId, String name, int wage, int photoId) {
        this.workerId = workerId;
        this.name = name;
        this.wage = wage;
        this.photoId = photoId;
    }
}
```
这是一个简单的布局文件(data_binding_layout.xml)，在TextView上显示Worker的名字：
```xml
    <?xml version="1.0" encoding="utf-8"?>
<layout xmlns:android="http://schemas.android.com/apk/res/android">


    <data>

        <variable
            name="worker"
            type="com.hiray.mvvm.mvvm.model.Worker" />

    </data>

    <TextView
        android:layout_width="120dp"
        android:layout_height="wrap_content"
        android:text="@{worker.name}" />  
</layout>
```
会根据布局文件名字生成一个ViewDataBinding对象————DataBindingLayoutBinding,在代码中设置一下：
```java
       public class DataBindingActivity extends AppCompatActivity {
            @Override
            protected void onCreate(Bundle savedInstanceState) {
                super.onCreate(savedInstanceState);
                DataBindingLayoutBinding binding = DataBindingUtil.setContentView(this, R.layout.data_binding_layout);
                binding.setWorker(new Worker(1205, "Mike", 12000, R.mipmap.boy);
            }
        }
```
以上是非常简单的一种绑定，只是单向的数据绑定，数据映射到UI上
关于这样的单向绑定有好几个注解

### BindingMethods、BindingMethod
比如我想给ImageView加上自定义的属性下载图片
```xml
        <?xml version="1.0" encoding="utf-8"?>
<layout xmlns:android="http://schemas.android.com/apk/res/android">


    <data>

        <variable
            name="worker"
            type="com.hiray.mvvm.mvvm.model.Worker" />

    </data>

    <ImageView
        app:imageDrawable="@{worker.photoId}"
        android:layout_width="50dp"
        android:layout_height="50dp"/>  
</layout>
```
这里"app:imageDrawable"这个属性是无法被识别的，会编译出错，我们要通过某种方式告诉系统该怎么做。我们建立一个类 ViewBindingAdapter，使用BindingMethods和BindingMethod注解：
```java
@BindingMethods(
        @BindingMethod(
                type = ImageView.class,
                attribute = "app:imageDrawable",
                method = "setImageResource"
        )
)
public class ImageAttr {

     
}
```
其中type是你要绑定的类，attribute是你自己定的,method是ImageView中的方法，也就是告诉ImageView，在xml中碰到app:imageDrawable属性的时候调用ImageView的setImageResource方法。
但是如果我此时不想传入图片id，而是传入一个id的String字符，那怎么办呢，因为ImageView并没有接收String参数来设置图片的方法，那么我们要利用BindConversion转换一下

### BindingConversion

```java

public class ImageAttr {

    @BindingConversion
    public static int convertStringToResId(String idString){
        return Integer.parseInt(idString);
    }
     
}
```

但是有些时候即使转换了，也没有对应的setter方法可以使用，比如你设置drawableLeft这种属性的时候，是没有setDrawableLeft方法的，只有setCompoundDrawable,那么这时候就可以使用 BindingAdapter这个注解了

### BindingAdapter 
顾名思义是绑定适配器，如果像设置的属性没有直接的方法，需要转换一下，那么就用到这个注解，比如设置drawableLeft,是没有setDrawableLeft方法的，必须调用view的setCompoundDrawables
```java

public class ImageAttr {

    @BindingAdapter("app:drawableLeft")
    public static void bindDrawableLeft(TextView view, Drawable leftDrawable) {
        view.setCompoundDrawables(leftDrawable, null, null, null);
    }
     
}
```

### InverseMethod
在双向绑定中，需要对值进行转换
比如我们有个checkbox，如果model中的一个String类型的值是“Alice”就让checkbox选上，反之不勾选，这里就要用到InverseMethod注解
新建一个Converter类，写了两个静态方法：
```java
    public class Converter {

    @InverseMethod("convertStringToBool")
    public static String convertBoolToString(boolean b) {
        if (b)
            return "Alice";
        else return "Unknown";
    }

    public static boolean convertStringToBool(String name) {
        return "Alice".equals(name);
    }
}
```

布局文件：
```xml
    <?xml version="1.0" encoding="utf-8"?>
<layout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto">

    <data>
        <variable
            name="person"
            type="com.hiray.mvvm.mvvm.model.Person"/>
        <import type="com.hiray.mvvm.mvvm.attr.Converter"/>
    </data>

        <android.support.v7.widget.AppCompatCheckBox
            android:checked="@={Converter.convertStringToBool(person.personName)}"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content" />

</layout>
```
### 给自定义控件支持双向绑定
这里有个自定义的BlinkView，不停的闪烁，由一个bool 类型的值 blink控制是否闪烁，伪代码如下：
```java
    public class BlinkView extends View {

    private Paint paint;
    boolean blink = true;
    ......

    public void setBlink(boolean blink) {
        this.blink = blink;
        invalidate();
    }

    public boolean getBlink(){
        return blink;
    }
    public void toggle() {
        setBlink(!blink);
        if (onBlinkChangeListener != null)
            onBlinkChangeListener.onBlinkChange(blink);
    }

    public interface OnBlinkChangeListener {
        void onBlinkChange(boolean blink);
    }

    private OnBlinkChangeListener onBlinkChangeListener;

    public void setOnBlinkChangeListener(OnBlinkChangeListener onBlinkChangeListener) {
        this.onBlinkChangeListener = onBlinkChangeListener;
    }
}
```
layout文件中：
```xml
<?xml version="1.0" encoding="utf-8"?>
<layout xmlns:android="http://schemas.android.com/apk/res/android">

    <data>

        <variable
            name="data"
            type="com.hiray.mvvm.mvvm.model.DataHolder" />


    </data>
        <com.hiray.mvvm.mvvm.widget.BlinkView
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            app:blink="@={data.blink}"
            app:blinkColor="@color/colorAccent" />

</layout>
```
这样写显然不会有任何效果，建立一个名为BlinkViewAdapter的类，使用InverseBindingMethods注解告诉Android在遇到app:blink属性的时候怎么把数据映射到UI上，然后又要告诉AndroidUI怎么映射到数据上，代码如下：
```java
    @InverseBindingMethods(
        @InverseBindingMethod(type = BlinkView.class,
                attribute = "blink")
)
public class BlinkViewAdapter {

//    @InverseBindingAdapter(attribute = "app:blink")
//    public static boolean isBlink(BlinkView view) {
//        return view.getBlink();
//    }

    @BindingAdapter(value = {"app:blinkChanged", "app:blinkAttrChanged"}, requireAll = false)
    public static void setListener(BlinkView view, BlinkView.OnBlinkChangeListener listener,
                                   final InverseBindingListener attrChange) {

        if (attrChange == null && listener != null)
            view.setOnBlinkChangeListener(listener);
        else view.setOnBlinkChangeListener(new BlinkView.OnBlinkChangeListener() {
            @Override
            public void onBlinkChange(boolean blink) {
                attrChange.onChange();
            }
        });

    }

}
```
其中InverseBindingMethod注解有三个参数：type表示你要绑定的类，attribute就是你要进行绑定的属性（就是写在xml上的属性），method默认按照属性名字去找有没有 isXX 或者getXX方法，不然你就写上method名字，如果没有直接的method名字，可以使用InverseBindingAdapter注解（代码中注释的部分），这是告诉系统在UI发生变化的时候调用什么方法获取UI信息这里Android默认是按照属性名字去找有没有 xxAttrChanged的（如果你自己没有定义的话），当然这个event可以自己定义，比如你定义成"abcdefg&%##$",那么上面setListener方法的BindingAdapter注解的"app:blinkAttrChanged"也得改成这个。
另外setListener有三个参数，第一个是控件BlinkView自己，第二个是BlinkView.OnBlinkChangeListener 和"app:blinkChanged"对应；第三个是InverseBindingListener是必须的，这个参数的实现在编译期就生成了，就是通知系统UI变化了，要更新ui信息到数据上（更新的方法就是前面的第一部分），可以看下生成的attrChange:
```java
     private android.databinding.InverseBindingListener mboundView3blinkAttrChange = new android.databinding.InverseBindingListener() {
        @Override
        public void onChange() {
            // Inverse of data.blink
            //         is data.setBlink((boolean) callbackArg_0)
            boolean callbackArg_0 = mboundView3.getBlink();
            // localize variables for thread safety
            // data.blink
            boolean dataBlink = false;
            // data != null
            boolean dataJavaLangObjectNull = false;
            // data
            com.hiray.mvvm.mvvm.model.DataHolder data = mData;



            dataJavaLangObjectNull = (data) != (null);
            if (dataJavaLangObjectNull) {




                data.setBlink(((boolean) (callbackArg_0)));
            }
        }
    };
```
看到这里其实就明白了，这里生成的东西都是根据前面的注解来的，收到刷新提示就会调用方法获取UI信息，并且更新数据模型中的值，由此完成了整个的双向的绑定