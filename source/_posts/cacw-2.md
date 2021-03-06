---
title: CACW开发笔记（二）：自定义View，实现载入动画！——前篇
date: 2015-11-12 20:30:55
tags: cacw
---

# 为什么要自定义View
前几天，当我在写app的登录界面的时候，用到了ProgressDialog这个dialog，虽然android自带了一个ProgressDialog，可是这货在不同平台上的表现不太一样，这对于有一点强迫症的我来说，不太能够接受。加上看了[materialup](http://www.materialup.com/)上各种漂亮的动画，于是决定用自定义View的方式来写一个Progress进度条。
<!-- more -->
这个自定义View现在已经上传到Github上了，有兴趣的同学欢迎fork，并改进这个动画。

[Github - ColorfulAnimView](https://github.com/microstudent/ColorfulAnimView)
# 最终效果

![](https://farm2.staticflickr.com/1535/23580321973_1fc797c2d4_o.gif)

# 开发笔记
要自定义一个View，最起码有两个方法需要重写，onMeasure(),以及onDraw()。

#### 1.View的测量
在XML文件里面我们通常可以很方便的用match\_parent、wrap\_content或者直接输入大小来定义一个控件的大小，其实这些特性都是在onMeasure()方法里面定义的。

Android系统提供了一个类——MeasureSpec，来帮助我们测量View。这是一个32位的int值，其中，高两位是测量的模式，低30位是测量的大小。使用这种方式是因为这样可以提高效率。

测量的模式有以下3种：

* **EXACTLY**

精确模式，在控件的属性设置为具体的大小值（如100dp），或者是match\_parent时，用的就是这个模式。这个模式意味着控件有指定的大小。

* **AT_MOST**

最大值模式，在控件的属性设置是wrap_content时，用的就是这个模式。这个模式是随着内容的变化而变化的。

* **UNSPECIFIED**

这个属性的意思是不指定模式，View想多大就多大，通常比较少用。

需要注意的是，如果不重写onMeasure()方法，那么这个自定义View就只能用EXACTLY模式。如果想要使用wrap_content的话必须重写该方法。

重写示例：

    @Override
    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        setMeasuredDimension(
        	measureWidth(widthMeasureSpec),
            measureHeight(heightMeasureSpec));
    }

可以看到，我们通过setMeasuredDimension(int measuredWidth, int measuredHeight)方法把处理后的长宽传进去完成测量。其中，measureWidth和measureHeight是我们即将写的自定义方法。

因此我们重写onMeasure的重点就是要计算出我们的View的大小。

下面是我们自定义的measureWidth方法（measureHeight类似）：

	  private int measureWidth(int widthMeasureSpec) {
        int result = 0;

        int specMode = MeasureSpec.getMode(widthMeasureSpec);
        //指定的模式，为EXACTLY时明确了长宽，为AT_MOST时，即wrapContent，
        //需要和指定的大小进行比较，如果是UNSPECIFIED，则像多大就多大
        int specSize = MeasureSpec.getSize(widthMeasureSpec);

        if (specMode == MeasureSpec.EXACTLY) {
            result = specSize;
        } else {
            result = px2dip(getContext(), 1500);//将1500px转成dp单位的长度
            if (specMode == MeasureSpec.AT_MOST) {
                result = Math.min(result, specSize);
            }
        }
        return result;
    }

可以看到，我们通过getMode和getSize就完成了提取模式和大小的操作，result作为最终确定的大小返回。

**这段代码可以作为模板使用。**

### 2.绘制View
经过onMeasure，我们已经确定的View的大小，下面我们可以进行View的绘制了。

毫无疑问我们这次重写的是onDraw()方法。我们先来看一下默认重写是怎么样的。

	    @Override
    protected void onDraw(Canvas canvas) {
    	super.onDraw(canvas);
    }
那么这里的Canvas是什么东西呢？

这是画布的意思，我们通过它，可以把我们的图像绘制在这上面。自定义View的各种图形都可以在这上面绘制出来，功能强大。

Canvas有以下基本方法：

* save()
* restore()
* translate()
* rotate()

简单讲一下这四个方法的作用。save()和restore()是成对使用的，save可以理解为是保存当前绘制的图层。restore则是把保存前后的图层进行合并。translate()、rotate()的意思分别是将绘制的坐标轴移动到(x,y)，rotate()则是将坐标轴进行旋转。

那么怎么开始在canvas上画出想要的图形呢？

我们可以调用canvas.drawXXXX()来绘制自己想要的图形，XXXX是不同的形状，包括Circle、line、Oval、Point等等，分别代表圆，线、椭圆，点等等；

具体到我们现在做的动画，我们的代码看起来是这样的：

	    @Override
    protected void onDraw(Canvas canvas) {
        drawRound1(canvas);
        drawRound2(canvas);
        drawRound3(canvas);
        drawScaleRound(canvas);
    }
其中的四个方法都是自定义方法，没有什么大的区别，因此我们看其中一个方法：

	    private void drawRound3(Canvas canvas) {
        canvas.drawCircle(round3.getX(), round3.getY(), round3.getSize() / 2, round3.getPaint());
    }

可以看到我们用了drawCircle绘制了一个圆，正是有了这四个方法，我们才能看到前面图片的四个小圆。

这里有必要提一下，round3是一个自定义类，包含了绘制小圆的所以信息。还有，Paint类是在Canvas上绘制需要用到的类，它是画笔的意思，可以设置画笔的颜色，alpha值等信息。

篇幅所限，前篇就暂时到这里啦，我将在后篇介绍如何控制小圆的运动效果的。

有兴趣的同学也可以去看看源码啦。

欢迎继续关注！
