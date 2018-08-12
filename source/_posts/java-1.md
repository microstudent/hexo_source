---
title: 学习笔记（一）之Java强引用、弱引用、虚引用、软引用
date: 2016-03-04 21:48:28
tags: java
---

## 前言
最近一直在为自己的Java知识充电，看了一些好书，决定把自己读到的觉得有用的知识拿出来分享一下，于是有了这个系列。
<!-- more -->

----------


## 什么是强引用
其实强引用在我们写Java代码非常常用，例如下面这一句语句就创建了一个强引用：
    
    Map<String,String> map = new Hashmap();
    
上面这个语句的作用是，创建一个Map对象，并且将其强引用至map上，这个引用是如此的强，以至于如果你不主动将map的引用设为null，垃圾回收器将不会主动回收这个对象！这个特性可以保证你正在使用的对象不会突然被回收了。

但是这样的引用是否能够完全满足我们的需求呢？

想象用以下的代码来实现一个栈：

    public class Stack{
        private Object[] elements;
        private int size = 0;
        private static final int DEFAULT_INIT_COUNT = 10;
        
        public Stack(){
            elements = new Object[DEFAULT_INIT_COUNT];
        }
        
        public void push(Object e){
            ensureCapacity();
            elements[size++] = e;
        }
        
        public Object pop(){
            if(size == 0)
                throw new EmptyStackException();
            return elements[--size];
        }
        
        /**
        *在当前数组存放的数量到达上限时自动拓展长度，以确保存放更多变量.
        **/
        private void ensureCapacity(){
            if(elements.length == size)
                elements = Array.copyOf(elements,2*size +1);
        }
    }
    
这段代码有什么问题？
其实里面隐藏着一个内存泄露的问题！
可以看到我们在pop出变量的时候是直接在size上进行自减来实现的，但是此时被弹出的对象还被elements这个数组引用着，对于垃圾回收器来说，他不确定我们是否还会在以后会继续用到这个引用，因此即使使用该栈的程序不再使用这些对象，垃圾回收器也不会主动回收这些对象。随着使用时间的不断增加，内存使用量将会不断增长！

其实修复的方法很简单，我们只需要在pop方法内return前加上一句：

    elements[size] = null;//断开强引用
    
垃圾回收器就能正常回收这个对象了。
这种工作对程序员来说实在是有些繁琐，那么有什么更好的方法呢？
这时候Reference对象应该登场了。

### Reference
java.lang.ref类库内包含了一组类，为垃圾回收提供了更大的灵活性，如果存在可能会耗尽内存的大对象时，这些类就非常有用。他们分别是SoftReference(软引用)、WeakReference(弱引用)、PhantomReference(虚引用)。与强引用不同的是，他们可以提供给垃圾回收器一定的依据，让垃圾回收器自行决定是否回收这些对象。

如果你想要持续持有某个对象的引用，希望以后还能访问到该对象，但是也希望垃圾回收器能及时回收他，那么就应该使用Reference对象了。

SoftReference(软引用)、WeakReference(弱引用)、PhantomReference(虚引用)由强到弱排列，对应着不同的『可及性』。
在堆中，对象有强可及对象、软可及对象、弱可及对象、虚可及对象和不可到达对象。应用的强弱顺序是强、软、弱、和虚。对于对象是属于哪种可及的对象，由他的最强的引用决定。
例如：

    String abc=new String("abc");  //1   
    SoftReference<String> abcSoftRef=new SoftReference<String>(abc);  //2   
    WeakReference<String> abcWeakRef = new WeakReference<String>(abc); //3   
    abc=null; //4   
    abcSoftRef.clear();//5 

在上述语句中，第一行创建了内容为"abc"的对象，并且建立了强引用。在第二行建立了软引用，但是此时仍然是强引用在起作用。第三行建立了弱引用，同样是强引用在起作用。直到第四行，消除了强引用，此时是软引用在起作用，最后一句消去了软引用，此时才是弱引用在起作用。
因此在使用Reference对象时一定不能有强引用指向该对象。

#### 软引用(SoftReference)
一般认为软引用是用以实现内存敏感的高速缓存。作为一个仅次于强引用的引用类型，它将一直持续到JVM内存不足时才会被清除，以防止OutOfMemoryError的错误。

#### 弱引用(WeakReference)
他是比软引用更低一级的引用，当gc扫描到一个弱引用引用到的对象时，无论是否存在内存不足的现象，垃圾回收器都会主动回收这个对象。但是由于垃圾回收器是一个优先级很低的线程，因此不一定会很快发现那些只具有弱引用的对象。

#### 虚引用(PhantomReference)
虚引用顾名思义，就是不存在引用，对虚引用执行get方法获得的始终是null，它的唯一作用就是当其指向的对象被回收之后，自己被加入到引用队列，用作记录该引用指向的对象已被销毁。
当一个对象被虚引用时，该对象被回收时将被加入到引用队列，以记录他被回收了，此刻将无法再次复活这个对象，而弱引用在被真正回收之前就会被加入到引用队列中，可以通过一个不规范的析构方法重新复活，从而不能确保对象完全被回收。
使用虚引用，上述情况将引刃而解，当一个虚引用加入到引用队列时，你绝对没有办法得到一个销毁了的对象。因为这时候，对象已经从内存中销毁了。因为虚引用不能被用作让其指向的对象重生，所以其对象会在垃圾回收的第一个周期就将被清理掉。

#### 引用队列(Reference Queue)
一旦弱引用对象开始返回null，该弱引用指向的对象就被标记成了垃圾。而这个弱引用对象（非其指向的对象）就没有什么用了。通常这时候需要进行一些清理工作。比如WeakHashMap会在这时候移除没用的条目来避免保存无限制增长的没有意义的弱引用。
引用队列可以很容易地实现跟踪不需要的引用。当你在构造WeakReference时传入一个ReferenceQueue对象，当该引用指向的对象被标记为垃圾的时候，这个引用对象会自动地加入到引用队列里面。接下来，你就可以在固定的周期，处理传入的引用队列，比如做一些清理工作来处理这些没有用的引用对象。