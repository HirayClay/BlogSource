---
title: 利用模板写Span
date: 2017-09-05 10:45:56
tags: 
    - SpannableString
    - TextView

---
在之前的项目中，PM特别喜欢把一些文字做颜色或者大小上的区分，所以经常会用到Span，没有什么好的封装想法，只能老老实实的用原始的api，显得非常的笨，但是又没有什么办法，没想到什么好的封装策略，只是觉得这样写真的好难看啊。但是一般需要做特殊处理的文字其实都是后台返回的某些字段，是有特别含义的，比如“距离审核还有6天结束”中的‘6’其实就是后台会单独返回给你的。我们App这边拼接好整句话然后显示出来。当时在做这样的项目的时候也找过类似的开源库，但是觉得总觉得哪里不对，也懒得用，还是用的原始的套路，先数一数‘6’在字符串中的起始结束下标，然后设置Span。直到最近为了深入了解gradle，去看了下groovy，看到“Template engines”的时候突然想起之前的Span，于是有了一个大胆的想法。

### 关于 Groovy的Template
Groovy可以动态生成字符串，比如模板是这样的'${name} is ${age} years old! '
绑定关系是这样的：[name:"Alice",age:"18"],那么生成的文字就是"Alice is 18 years old!"。你可能要问了，这和你说的Span有什么关系？？？当然有，前面我们说了，我们的需要设置Span的文字其实都是有含义的，我们用原始的api那样数出下标然后设置Span非常的无脑，根本没有体现出这个字段的含义，但是现在如果我们用groovy的方式，定义自己的模板那么"距离审核还有6天结束"的模板是不是就是"距离审核还有${day}天结束"，这样表达起来是不是更有内涵些，然后你又要问我，确实有内涵了，但这和Span又有什么关系呢？？好吧，也没什么关系，就是要有内涵一点，所以借用Groovy的思想重新封装对Span的处理。你可能还要问，不是已经有类似的库了吗，干嘛还要封装一个，比如Spanny。那好，我们看看Spanny怎么做的，
```java
    Spanny spanny = new Spanny("距离审核还有")
                .append("6", new ForegroundColorSpan(Color.RED))
                .append("结束！");
    textView.setText(spanny);
```


对比我们定义好一个模板 "距离审核还有${day}天结束"，比较一下就看出不同了。Spanny的做法是希望需要什么样的Span就自己拼一个，虽然配合链式调用挺舒服的，其实语义上就很乱，个人觉得还是Groovy这样的模板实在，毕竟当需要处理Span的时候，结构都是死的，所以用模板定义好结构是没有问题的，特别是当要处理的文字比较多的时候，这样拼接我觉得不太好，用定义的模板一眼看过去就非常的清晰明了。



### 需要解决的问题
虽然引入了template的思想来动态生成字符串，同时又需要对key替换后的文字做对应的处理，那么要解决的问题有以下三个：
1 如何解析模板字符串
2 如何替换key并生成结果字符串
3 如何解决以上两个问题

关于第一个问题，看了下groovy解析模板的代码，自己做了一下修改
最后解析模板的代码是这样的：
```java
List<MarkInfo> parseAndMark(Reader reader, Map<String, String> binding) {
        if (!reader.markSupported())
            reader = new BufferedReader(reader);
        List<MarkInfo> markers = new ArrayList<>();
        MarkInfo mark;
        StringWriter writer = new StringWriter(50);
        while (true) {
            int c;
            try {
                if ((c = reader.read()) != -1) {
                    if (c == '$') {
                        reader.mark(1);
                        c = reader.read();
                        if (c == '{') {
                            String key = findKey(reader);
                            //only true for text
                            if (!key.isEmpty()) {
                                String value = binding.get(key);
                                //for text
                                if (value != null) {
                                    int start = writer.getBuffer().length();
                                    int end = start + value.length();
                                    markers.add(new MarkInfo(key, value, start, end));
                                    writer.write(value);
                                } else {//for image

                                    int start = writer.getBuffer().length();
                                    int end = start + key.length();
                                    markers.add(new MarkInfo(key, value, start, end));
                                    writer.write(key);
                                }
                            } else {
                                writer.write("${");
                                //key not found
                                writer.write(key);
                            }

                        } else {
                            writer.write('$');
                            reader.reset();
                        }
                    } else {
                        writer.write(c);
                    }
                } else break;
            } catch (IOException e) {
                e.printStackTrace();
            }
        }

        return markers;
    }


    String findKey(Reader reader) {
        StringWriter stringBuilder = new StringWriter(10);
        int c;
        try {
            while ((c = reader.read()) != -1) {
                if (c == '}')
                    break;
                else stringBuilder.write(c);
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
        return stringBuilder.toString();
    }
```
这个方法的作用就是记录模板中所有key的起始结束位置。比如原始模板是："${name} is ${age} years old! "数据映射是[name:"Alice",age:"18"],解析之后就变成"Alice is 18 years old! "。并且'Alice' '18'两个数据在字符串中的位置被记录在了MarkrInfo中

我们来简单分析一下代码，一个简单的while循环，每次读取一个字符，每当读到'$'字符时认为可能是key要出现了，所以先在此处标记一下紧接着读取下一个字符，如果读到下一个字符是'{'则认为key出现了，调用findKey方法读取'{'和'}'之间的key值，如果为空则认为没有key，仅仅是读到了一个普通的"${}"，并且写入writer保存起来，如果key不为空认为读取到有效的key，记录key对应的value在字符串中的位置等信息，并且将value写入writer保存起来；如果'$'后面读到的不是'}'则认为只是读到了一个单独的'$'字符，虚惊一场，写入writer保存起来，并且把reader 重置，回到刚才标记的地方，也就是'$'的位置；如果读取的是普通的字符，直接写入writer.另外要说的就是ImageSpan的处理，由于有些字符最后是要替换成图片的，所以在binding中是没有其对应value的，所以当读取的key在binding中如果没有value，就认为这个key是要被替换成图片的，所以直接用key代替value，直接把key写入writer保存起来。

解析这一步完成以后我们其实得到了一个List<MarkerInfo>，记录了key被替换成value后的value在结果字符串中的位置信息，以及原始的key等信息。有了这些重要信息，就可以根据下标施加对应的Span了，以及一些点击事件的监听了。