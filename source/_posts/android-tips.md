---
title: Android的小笔记
date: 2016-06-01 15:51:05
tags:
---
最近在知乎上关注了这个问题：[Android 开发中，有哪些坑需要注意？](https://www.zhihu.com/question/27818921)

里面有许多Android开发大神所讲的各种小点，对我来说非常有帮助。我在开发过程中有时候也会遇到一些小坑，因此在这里也贴一下。（也许就是自己用错了？（笑:)）

<!-- more -->

### - 自定义View

- 由于没对Painter设置线条Width，导致在canvas里面drawLine()没有显示效果（也可能是因为屏幕dpi太高？）
- surfaceView 每次拿到的canvas都不会清空,Path也是
- 由于不理解贝塞尔曲线控制点的含义，导致在canvas里绘制贝塞尔曲线时一直画不出曲线（这个是自己的学得太少(⊙﹏⊙)b）
- 在需要传context的时候，注意传的是哪个context，如果传的是ApplicationContext，则在在inflate的时候绘制出来的View会是系统默认样式，而在xml里定义的样式是无效的。

关于不同Context能干什么请看下图：![image](https://pic2.zhimg.com/9be7e8e2d05cd088fb79d22b211fdaad_b.png)
> 备注：大家注意看到有一些NO上添加了一些数字，其实这些从能力上来说是YES，但是为什么说是NO呢？下面一个一个解释：
> 1、数字1：启动Activity在这些类中是可以的，但是需要创建一个新的task，一般情况不推荐；
> 
> 2、数字2：在这些类中去layout inflate是合法的，但是会使用系统默认的主题样式，如果你自定义了某些样式可能不会被使用；
> 
> 3、数字3：在Receiver为null时允许，在4.2或以上的版本中，用于获取黏性广播的当前值。（可以无视）；
> 
> 4、ContentProvider、BroadcastReceiver之所以在上述表格中，是因为在其内部方法中都有一个context用于使用。
> 
> 来自知乎[@张明云](https://www.zhihu.com/people/zhang-ming-yun-88)的答案

- 默认情况下Viewgroup 的onDraw方法不会被调用，除非设置willNowDraw为false
- RecyclerView.scrollToPosition()方法与LayoutManager.scrollToPositionWithOffset()方法的区别：如果在recyclerView里面调用scrollToPosition方法，则rv确实会滚动至该位置，但是这个方法只保证你要的position在可见范围内。但scrollToPositionWithOffset则保证了该position相对顶部有多少的offset，保证该position所在item相对Rv的位置。
- 在android4.1.1里使用?accent这个颜色会报错（其他Android版本未实验）