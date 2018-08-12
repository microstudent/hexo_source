---
title: Glide、Picasso性能对比报告
date: 2017-1-7 20:04:19
tags:
---

好久不见，Glide和Picasso都是目前网络上比较常见的一些图片加载框架，下面就他们的一些差异和优缺点进行分析。
本文主要讲述以下几点：
* 图片加载API的区别
* 图片的缓存策略区别
* 互相的优劣势
 >本文对比的是Picasso2.5.2和Glide3.7.0
 
<!-- more -->

## 0x01图片加载API的区别
1. 对于最普通的Url加载图片到ImageView中，两者的写法分别是

  Glide：

```
      Glide.with(context)//with处输入参数的具体类型可以影响到Glide是否能够正确管理图片的生命周期，下面细讲
          .load(url)
          .asBitmap()//对于Glide来说，除了Bitmap可以是Gif
          .format(DecodeFormat.PREFER_RGB_565)//Glide默认Bitmap的色彩模式是565
          .transform(new CircleTransform())
          .placeholder(placeholder)
          .error(error)
          .override(width,height)//覆盖图片加载大小，与Picasso用法不同，且只能输入px值
          .listener(listener)//监听加载事件，与Picasso用法不同
          .into(imageView);
```
  Picasso：

```
        Picasso.with(context.getContext().getApplicationContext())//最好传入AppContext以避免内存泄露
                          .load(url)
                          .config(Bitmap.Config.RGB_565)//Picasso的默认色彩模式是8888
                          .transform(new CircleTransform())
                          .fit()//Picasso需要调用fit()来自适应ImageView的大小，下面会仔细描述这一区别
                          .placeholder(placeholder)
                          .error(error)
                          .resize(widthRes,heightRes)///覆盖图片加载大小，与Glide用法不同，可以输入Res值
                          .into(imageView,callback);//监听加载事件，与Glide用法不同
```
  可以看到，Glide和Picasso的用法基本一致，除了一些比较特殊的调用。

2. 缓存相关API

  在Picasso里面，由于没有内存缓存清理的API，我们用的是通过反射来获取缓存进行清空，使用反射始终不是一种妥当的方法。
  ```
        public static void clearPicassoCache(Context context) {
         try {
             Cache cache = (Cache) ReflectHelper.getField(Picasso.with(context.getApplicationContext()), "cache");
             cache.clear();
             } catch (NoSuchFieldException e) {
             LogHelper.logW(e);
            }
          }
```

    在Glide里面则可以方便的使用以下语句清理内存缓存。
```
        Glide.get(context).clearMemoryCache();
```
    另外，Picasso本身不实现磁盘缓存，我们需要通过配置Okhttp来实现缓存，而这依赖于Http的响应头或者请求头，对服务器有要求，亦或是我们需要自己修改请求头。
    而Glide默认实现了磁盘缓存，同时还有BitmapPool作为重用池，对于一个废弃的bitmap，GLide会把它放进BitmapPool，用以重用。同时，这样还可以避免在垃圾回收过程中造成的停顿。Glide还支持自定义标识来标识一个缓存内容。
    >Glide 以 url、viewwidth、viewheight、屏幕的分辨率等做为联合key，官方api中没有提供删除图片缓存的函数，官方提供了signature()方法，将一个附加的数据加入到缓存key当中，多媒体存储数据，可用MediaStoreSignature类作为标识符，会将文件的修改时间、mimeType等信息作为cacheKey的一部分，通过改变key来实现图片失效相当于软删除。

3. 图片加载顺序
  对于图片的加载，我们有时会考虑到，对于顶部的图片我们可能需要优先加载，而对于底部的图片我们希望最后加载，那么Glide也能够让我们为每次加载配置优先级priority。
  >这个枚举给了四个不同的选项，下面是按照递增priority(优先级)的列表：
    * Priority.LOW
    * Priority.NORMAL
    * Priority.HIGH
    * Priority.IMMEDIATE

        Glide
         .with(context)
         .load(url)
         .priority(Priority.HIGH)
         .into(ImageView);

4. Glide.with()传入参数对图片加载的影响
  Glide有一个称为生命周期集成（Lifecycle integration）的特性，图片请求能够根据我们在 Glide.with()传入的对象的不同来暂停或重新加载，同时，Gif动画也能够在onStop后自行暂停，节省电量。此外，当手机的网络状态发生了改变，所有失败的图片请求都要重新加载以确保没有请求因为暂时的网络问题而永远无法加载。

  关于生命周期绑定的实现原理，我粗略的翻阅了一下源代码，下面简单描述一下Glide是如何做到生命周期绑定的。

  Glide.with()方法接受包括Activity、FragmentActivity、Fragment、support包的Fragment，context这几种参数。他们的实现都是由RequestManagerRetriever的单例来获取一个RequestManager来加载图片。

          public static RequestManager with(Context context) {
            RequestManagerRetriever retriever = RequestManagerRetriever.get();
            return retriever.get(context);
        }
  RequestManagerRetriever是真正用于获取RequestManager的类，他内部包括两个HashMap，两个都是用于存放泛型为<FragmentManager, RequestManagerFragment>的，只是一个支持的是support包的Fragment。

  无论输入的是Activity还是Fragment，在获取RequestManager的时候都是以他们内部的FragmentManager来作为绑定依据。例如以下代码：

      public RequestManager get(FragmentActivity activity) {
            if (Util.isOnBackgroundThread()) {
                return get(activity.getApplicationContext());//应用运行在后台，使用AppContext
            } else {
                assertNotDestroyed(activity);
                FragmentManager fm = activity.getSupportFragmentManager();//获取activity的Fragment，如果这里传入的是Fragment，那么获取的是ChildFragmentManager，目的都是为了附加一个Glide的一个不可见fragment，同时作为key和这个不可见fragment绑定在一起
                return supportFragmentGet(activity, fm);//见下
            }
        }


        RequestManager supportFragmentGet(Context context, FragmentManager fm) {
                SupportRequestManagerFragment current = getSupportRequestManagerFragment(fm);//Glide为了绑定生命周期所附加的Fragment，无视图
                RequestManager requestManager = current.getRequestManager();//在frgament中获取RequestManager
                if (requestManager == null) {
                  //若还没有RequestManager，则new一个出来
                    requestManager = new RequestManager(context, current.getLifecycle(), current.getRequestManagerTreeNode());
                    current.setRequestManager(requestManager);
                }
                return requestManager;
            }
  对于上面提到的RequestManagerFragment，他的内部实现是这样的：

  RequestManagerFragment包含了RequestManager、ActivityFragmentLifecycle等成员变量。因为RequestManagerFragment通过了FragmentManager附加到了Glide.with()传入的Activity或者Fragment，因此RequestManagerFragment的生命周期是相对应的。

  * ActivityFragmentLifecycle是起到一个事件传递的作用，对每一个RequestManagerFragment传递的生命周期事件，ActivityFragmentLifecycle都会相应地notify其所添加的所有Listener。

  * RequestManager在构造方法中监听了ActivityFragmentLifecycle，因此他也会接收到生命周期的变化，并根据相应的生命周期做出反应。比如在onStop的时候暂停请求，在onDestroy的时候取消所有请求：
```
      /**
       * Lifecycle callback that unregisters for connectivity events (if the android.permission.ACCESS_NETWORK_STATE
       * permission is present) and pauses in progress loads.
       */
        @Override
        public void onStop() {
            pauseRequests();
        }

      /**
       * Lifecycle callback that cancels all in progress requests and clears and recycles resources for all completed
       * requests.
       */
      @Override
      public void onDestroy() {
          requestTracker.clearRequests();
      }
```

## 0x02图片的缓存策略区别

对于图片的缓存，Glide和Picasso是两种不同的做法。
  - 对于Glide来说，他会首先计算ImageView的大小，下载完图片后对图片进行重绘，而不是直接将图片加载到内存。
  - 而对于Picasso来说，他的策略是下载完图片后，加载原图进内存缓存，如果需要resize再根据这原图进行resize放进ImageView中。

此外，Glide和Picasso的默认色彩模式不一样，Glide的默认色彩模式是RGB_565,而Picasso的默认色彩模式是ARGB_8888。他们的区别主要是色彩值存放的位数不一样，因此对于同一张图片，他们占用的内存也不一样。就感官上来说，他们没有太大的差异，而565的内存占用要比8888少一半。565在占用内存小的同时，图片的质量会随之下降（肉眼还是挺难看出来的），并没有透明度的特性。
>Bitmap.Config ARGB_4444：每个像素占四位，即A=4，R=4，G=4，B=4，那么一个像素点占4+4+4+4=16位
Bitmap.Config ARGB_8888：每个像素占四位，即A=8，R=8，G=8，B=8，那么一个像素点占8+8+8+8=32位
Bitmap.Config RGB_565：每个像素占四位，即R=5，G=6，B=5，没有透明度，那么一个像素点占5+6+5=16位

### 测试用例

  我在应用商店项目里面将图片加载库替换成了Glide，同时配置默认色彩模式与Picasso一样，为8888，用以替换Picasso。

  对于同一台手机，分别安装替换了Glide的安装包和原来的Picasso的安装包，结束进程并清空数据，进入首页精选首页，匀速往下滚动直至最底部，保证加载了精选页所有的图片，观察他们的内存占用情况，结果如图：
  ![Cache](http://ww3.sinaimg.cn/mw1024/5dbb16bcgw1fbh1gogkbsj20yg075q38.jpg)

  可以看出，对于同样的测试用例和测试环境，Glide在内存占用上会比Picasso更具有优势。

## 0x03互相的优劣势

Picasso的优势：
  * 方法数少，整个库所占的大小较小
    Picasso (v2.5.1)的大小约118kb，而Glide (v3.5.2)的大小约430kb。
    Picasso和Glide的方法个数分别是840和2678个。
  * 自带统计监控功能
    Picasso可以统计缓存命中率或者节省流量大小等

Picasso的劣势：
  * 相对比Glide，内存占用较大。
  * API较少，不提供较为常见的例如清理内存缓存的API

Glide的优势：
  * 加载速度较Picasso快，因为其不需要在显示之前先对图片调整大小
  * 具有完善的缓存体系，包括内存缓存、磁盘缓存、BitmapPool复用
  * 能够加载Gif动画图（3.7.0加载某些GIF图有BUG，3.8.0已经解决）
  * 能够和Okhttp或者Volley关联使用
  * 生命周期绑定

Glide的劣势：
  * 方法数过多，容易触发65535方法数限制，包体积较Picasso大
  * Glide.with()传入Activity或者fragment的代码侵入性较强，不容易封装
  * 在一般情况下，Glide加载的图片质量稍比Picasso差

## 0x04总结
Glide和Picasso都是非常优秀的图片加载库，两者的用法都非常接近。Picasso 所能实现的功能 Glide 都能做到，只是所需设置不同。两者的区别是 Picasso 比 Glide 体积小很多且图像质量比 Glide 高，但Glide 的速度比 Picasso 更快，内存占用也更小。就用法而言，Glide是比Picasso更加丰富的。

在当前内存占用需要尽可能减小的情况下，替换Picasso成Glide是较为合理的解决方法！
