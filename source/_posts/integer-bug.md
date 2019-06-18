---
layout: _posts
categories: java
tags: [integer]
title: Integer之你所不知道的bug
date: 2019-06-16 01:54:10
---
## 写在前面的话
本来以为自己的博客没几个人看，但是群里的小伙伴却很认真的看完了，顺便还帮我找了几个语病，提了不少建议。这里我要感谢一下[@Rukawalee](https://github.com/Rukawalee),这位同学是我的审稿小王子，不辞辛劳的帮我审稿，调整语言和篇幅，修改一些语病，以达到最好的阅读体验(内心OS：好歹初中语文年级第一，怎会沦落为现在这样语病百出的地步，哭唧唧)。应小伙伴们的建议，我将这篇文章进行重写，添加了一些Integer的知识点，同时也增加了一些例子，让同学们能更加清晰，更加深入的了解Integer的秘密。当然，这个知识点并不仅仅局限于int和Integer，所有的基础类型和包装类型的知识点都是互通的，这里只是以int和Integer为例而已。
## bug缘起
今天日常在牛客群里和小伙伴们吹水聊天，我聊到代码规范的时候吐槽了一波，公司每个人的代码规范都不统一，int和integer有的人用equals有的人用==。然后建议统一用equals。于是贴了一波代码为了佐证一下我的建议吧。代码如下:
```java
int a = 1;
Integer b = null;
if(a == b) {
    System.out.println(true);
}else {
    System.out.println(false);
}
```
我说，如果写出这样的代码，直接就掉空指针异常坑里了，所以还是最好统一使用equals方法。然而，群里的小伙伴可能是为了捧哏故意说不知道有啥问题。嗯，于是我不要脸的好为人师了一把，既然大家给面子，那就装一波(手动滑稽)。这次，我就从源头上来剖析包装类和基本数据类型的区别以及bug出现的原因。
## int及其包装类
### Integer的起源
<font color=red>在Java中，因为基本类型并不是一个类，它是以值的形式存储在虚拟机内存中。因此它不能在Object之间直接引用。</font >比如说在Map中作为泛型使用或者像Object一样调用方法。如果我们想将基本类型当做类来处理，这个时候我们就需要使用到包装类Integer了。包装类是JDK1.0中提供的，他解决了基本数据类型和Object之间不能直接传递的缺陷。

### int的自动装箱和拆箱
在JDK1.5之前，int和Integer是不能直接比较的，它需要我们自己显式的调用intValue()进行比较。但是在JDK1.5之后提供了拆箱和装箱的功能之后，我们可以直接将其进行==比较。当然，它同时还提供了其他的便利，待会我会以代码进行一一解释。

#### 自动装箱
我们看下面一段代码：
```java
public class Test01 {

    public static void main(String[] args) {
        Integer i = 10;
        encasement(10);
    }

    private static void encasement(Object i) {
        if (i instanceof Integer) {
            System.out.println("int 被包装成了 Integer");
        } else {
            System.out.println("int 并没有被包装成Integer");
        }
    }
}
```
我们思考一下，上面的代码会输出什么呢，相信聪明的读者已经知道了答案，那就是:
```java
int 被包装成了 Integer
```
接下来，我们来解析上面的一段代码，因为引入了自动装箱机制，<font color=red>"Integer i = 10"实际上是调用了valueOf()方法，所以上面的方法等同于"Integer i = Integer.valueOf(10)"。注意，这个不等同于"Integer i = new Integer(10)"。</font>为什么强调这一点呢，我待会会讲到。

然后就是第二行代码了，我们前面说过，int不能在Object之间直接引用，为什么我的int能被encasement方法接收呢，原因也是内部做了装箱操作，在方法的传递中，我们的编译器悄悄的把int转换成了Integer，从输出结果也可以看出来了。
#### Integer的==到底比较的是什么
我们大家应该都知道，<font color=red>==除基本类型比较的是值之外，Object类型都是比较地址的。</font>Integer也不例外，但是细心的小伙伴可能会发现下面这个问题。
```java
public class Test02 {
    public static void main(String[] args) {
        Integer a = 10; // 实际为 Integer a = Integer.valueOf(10);
        Integer b = 10;
        Integer c = new Integer(10);
        Integer d = 200;
        Integer e = 200;
        System.out.println(a == b);
        System.out.println(b == c);
        System.out.println(d == e);
        System.out.println(d.equals(e));
    }
}
```
输出结果为：
```java
true
false
false
true
```
<font color=red>同学们注意了，以上均为Integer，实际上比较的是对象的引用(即地址)。</font>但是为什么第一个会输出true呢，按照常理来说，第一个输出结果应该为false呀。同学们别急，听我慢慢给你们解释。解释前我们先看一段源码。

```java
public final class Integer extends Number implements Comparable<Integer> {
    ...// 省略
    private final int value;

    public static Integer valueOf(int i) {
        if (i >= IntegerCache.low && i <= IntegerCache.high)
            return IntegerCache.cache[i + (-IntegerCache.low)];
        return new Integer(i);
    }
    
    public Integer(int value) {
        this.value = value;
    }
        
    private static class IntegerCache {
        static final int low = -128;
        static final int high;
        static final Integer cache[];

        static {
            // high value may be configured by property
            int h = 127;
            String integerCacheHighPropValue =
                sun.misc.VM.getSavedProperty("java.lang.Integer.IntegerCache.high");
            if (integerCacheHighPropValue != null) {
                try {
                    int i = parseInt(integerCacheHighPropValue);
                    i = Math.max(i, 127);
                    // Maximum array size is Integer.MAX_VALUE
                    h = Math.min(i, Integer.MAX_VALUE - (-low) -1);
                } catch( NumberFormatException nfe) {
                    // If the property cannot be parsed into an int, ignore it.
                }
            }
            high = h;

            cache = new Integer[(high - low) + 1];
            int j = low;
            for(int k = 0; k < cache.length; k++)
                cache[k] = new Integer(j++);

            // range [-128, 127] must be interned (JLS7 5.1.7)
            assert IntegerCache.high >= 127;
        }

        private IntegerCache() {}
    }
            
    public boolean equals(Object obj) {
        if (obj instanceof Integer) {
            return value == ((Integer)obj).intValue();
        }
        return false;
    }
    
        ...// 省略
}
```
我们在上面已经说到过对Integer直接赋值，编译器会自动优化成对valueOf()方法的调用。所以蹊跷必然出现在这个valueOf()方法。通过源码我们可以发现，<font color=red>Integer内部有一个IntegerCache的内部类，这个内部类里面维护了一个cache[]的Integer数组，数组的大小为256，范围是-128~127。而我们的valueOf()方法是判断int的值是否是在这个区间内。如果在这个区间内，我们返回的则是cache数组中的Integer对象，否则new一个新的Integer对象出来。</font>这下我们应该就明白了为什么第一个会输出true了，因为他们返回的都是同一个对象，当然地址就是一样的啦。<font color=red>第二个输出为false的原因是new关键字的含义就是从内存中申请分配空间，所以地址必然是不一样的</font>。至于第三个为什么false，那是因为超出了区间调用了new Integer()，所以地址也是不一样的。<font color=red>这也是为什么装箱不是调用的new，因为如果调用的是new，那么第一个输出只会是false了。</font>而我们的equals()方法则是通过上一节讲的装箱操作，将int转换为Integer之后通过比较类型再将两个对象的value值进行比较，所以只要值相等就会返回true，参考第四个输出。对了，IntegerCache的区间是可调的，至于具体操作，因为不在这篇博客讲解的范围内，大家可自行google或者百度。
#### 自动拆箱
还是先看一段代码：
```java
public class Test03 {

    public static void main(String[] args) {
        int a = 10;
        Integer b = new Integer(10);
        System.out.println(a == b);
        encasement(b);
    }

    private static void encasement(int i) {
        System.out.println(i);
    }
}
```
输出结果：
```java
true
10
```
我们来详解一下，我们之前说过<font color=red>除了基本数据类型，java对象==比较的都是地址，因此基本数据类型是无法和除包装类以外的Object使用==进行比较。同时也不能在Object之间直接引用。</font>那为什么上面代码可以通过编译并且输出true呢，那是因为int包装类型在和int进行比较的时候会隐式的调用intValue()方法，所以在"a == b"这一句代码中，其实真正的语法是"a == b.intValue()"。"encasement(i)"同理，在进行参数传递的时候调用了intValue()方法。因此我们可以使用int来接收它。
## 为什么会有bug
好吧，我们回到之前的问题中来，我相信大家都快忘了问题是什么，没关系，我把代码再贴一下。顺便请问一下同学们，下面的代码会出现什么bug呢？
```java
int a = 1;
Integer b = null;
if(a == b) {
    System.out.println(true);
}else {
    System.out.println(false);
}
```
仔细思考一下哦，思考完之后看看下面的结果和你猜想的是不是一样的。
```
Exception in thread "main" java.lang.NullPointerException
	at com.example.test.Test04.main(Test04.java:17)
```
为什么会抛出空指针异常呢，我相信聪明的小伙伴们已经知道了原因了，那就是因为"a == b"实际上是"a == b.intValue()",而我们的b为null，这就导致了空指针异常的发生。虽然我们直接写出来对比很直观，但是在写项目的时候如果直接使用==来比较int和Integer可能就会忘了Integer为null的情况，若是通过测试上线后发现问题，那就是个惨痛的教训了。

因此，我个人建议使用equals来比较int和Integer，因为在这之前我们必然要考虑到Integer等于null的情况，避免了跳进坑里面。同时，建议大家项目中的POJO类都使用包装类型而不是基本数据类型，因为有的时候需要用null来表示一些异常情况。比如说股市的涨幅情况，如果出了问题导致没有获取到这个数值，使用Integer我们能使用-来表示获取失败，而如果是int，经过序列化之后获取的则是0，那显然是一个错误的值。

然后，一个小伙伴说，他们项目组刚遇到这个问题，并且还放到公司的wiki上面去了。我相信通过上面的解释，大家应该都能知道为什么需要加三目表达式了吧。
![](https://i.loli.net/2019/06/17/5d076c3d17f7525889.png)
## 通过bug我们学习到了什么
其实，这个bug我们大家都能避免的。因为Integer的装箱和拆箱机制大家应该都是知道的，这个知识点就和基础类型的隐式装换一样烂熟于心了。可是为什么还是会掉进这个陷进里面去呢，我想还是太大意，没有去深入思考。我们都知道Integer和int比较会有拆箱操作，却没有认真的去思考java是如何拆箱的，知其然却不知其所以然，最终导致错误的发生。

其实Integer这个类是有很多可以学习到的知识点，所以我建议大家可以去看看这个的源码。最后，对于每一个知识点，我们都应该知其然并知其所以然，应该发散自己的思维，通过现象看本质。这样我们才能在工作中完美的避开那些隐蔽的坑。