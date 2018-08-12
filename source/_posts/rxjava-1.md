---
title: RxJava使用小记录
date: 2016-06-01 15:51:05
tags: rxjava, retrofit
---

# 写在前面
这几天在看技术博客的时候，看到了经常出现的rxjava教程。因为自己之前比较浮躁，因此都没有时间静下心来仔细看，这次终于比较完整的把这篇文章看完了，于是打算在以后的项目中，尝试着使用RxJava进行开发，学习它那种“流水线”式的开发方式。我尝试了RxJava和Retrofit的结合，目前看起来效果还不错。继续加油！随着我对Rxjava的使用，这篇文章会慢慢补充。算不上什么正式的文章，只是自己平时的一些使用记录。不足之处欢迎指da出lian:)

# RxJava操作符的理解

<!-- more -->

- Map

Map的主要作用是将一个对象转换成另外一个对象。举个例子，我们有一个图片url，我们想要把图片下载下来，然后设置给ImageView，那么我们可以先把这个Url通过map转成Bitmap。
伪代码如下：

```
        String url = "...";
        Observable.just(url)
                .subscribeOn(Schedulers.io())
                .map(new Func1<String, Bitmap>() {
                    @Override
                    public Bitmap call(String s) {
                        return downloadFromUrl(s);
                    }
                })
                .observeOn(AndroidSchedulers.mainThread())
                .subscribe(new Subscriber<Bitmap>() {
                    @Override
                    public void onCompleted() {
                        //下载完成
                    }

                    @Override
                    public void onError(Throwable throwable) {
                        //下载出错
                    }

                    @Override
                    public void onNext(Bitmap bitmap) {
                        //成功拿到bitmap
                    }
                });
```

如上所示，我们在map里面把一个String类型的对象转成了一个bitmap对象，最终在订阅的时候，就可以在onNext里面把这个bitmap赋值给ImageView了。

- flatMap

flatMap的作用比较复杂。我自己的理解是，它和map的作用类似，map针对的是一对一的转换，而flatMap针对的是一对多的转换。
还是上面的那个例子，如果我们有不止一个Url，我们有一个List<String>的图片Url，那么我们可以使用flatMap把这一单个List<String>通过flatMap转换成多个String对象分别发出。（当然也可以在一开始的时候就用Observerable.from来做，这里只是为了说明flatMap的作用）。

```
        List<String> urls = new ArrayList<>();
        Observable.just(urls)
                .subscribeOn(Schedulers.io())
                .flatMap(new Func1<List<String>, Observable<String>>() {
                    @Override
                    public Observable<String> call(List<String> strings) {
                        return Observable.from(strings);)
                    }
                })
                .map(new Func1<String, Bitmap>() {
                    @Override
                    public Bitmap call(String s) {
                        return downloadFromUrl(s);
                    }
                })
                .observeOn(AndroidSchedulers.mainThread())
                .subscribe(new Subscriber<Bitmap>() {
                    @Override
                    public void onCompleted() {
                        //下载完成
                    }

                    @Override
                    public void onError(Throwable throwable) {
                        //下载出错
                    }

                    @Override
                    public void onNext(Bitmap bitmap) {
                        //成功拿到bitmap
                    }
                });
```
上面的代码除了多了个flatMap，其他的和上一个示例没有区别。

# 我的使用记录

- 使用Rxjava来使按钮不可用一段时间，并在按钮上进行读秒。

需求大概就是如标题所示，使用场合就是我们app的获取短信验证码的按钮，为了避免用户在短时间内重复请求过多的短信验证码，我们可以在一段时间内屏蔽用户的请求，把按钮设置为不可用，并且进行读秒，我们可以通过rxJava实现这个功能。

首先我定义了一个Observerable：

```
mWaitingTask = Observable.range(1, DEFAULT_WAITING_TIME, Schedulers.newThread())
                .doOnNext(new Action1<Integer>() {
                    @Override
                    public void call(Integer integer) {
                        try {
                            Thread.sleep(1000);
                        } catch (InterruptedException e) {
                            e.printStackTrace();
                        }
                    }
                })
                .observeOn(AndroidSchedulers.mainThread());
```
他的作用主要就是，从1到DEFAULT_WAITING_TIME，在一条新线程里面阻塞1000ms，然后执行onNext操作,以模拟1s的情形。然后我在按钮的onClick事件上，写了如下代码：

```
 //开始读秒
                           mCompositeSubscription.add(mWaitingTask.subscribe(new Subscriber<Integer>() {
                                @Override
                                public void onStart() {
                                    mGetCodeButton.setEnabled(false);
                                    mGetCodeButton.setText(String.valueOf(DEFAULT_WAITING_TIME));
                                }

                                @Override
                                public void onCompleted() {
                                    mGetCodeButton.setEnabled(true);
                                    mGetCodeButton.setText("获取验证码");
                                }

                                @Override
                                public void onError(Throwable e) {
                                    mGetCodeButton.setEnabled(true);
                                    mGetCodeButton.setText("获取验证码");
                                }

                                @Override
                                public void onNext(Integer integer) {
                                    //设置按钮
                                    mGetCodeButton.setText(String.valueOf(DEFAULT_WAITING_TIME - integer));
                                }
                            }));
```

如此，便实现了一个按钮在规定的时间内，屏蔽用户的新请求的功能。当然这个实现看起来很不科学=-=，应该可以有更好的实现方法，所以仅供参考哦。

