---
title: 一个由SharedPreferences引起的ANR
date: 2018-09-10 20:12:46
tags: SharedPreferences
---
## 0x01 起因
商店每次发布新版本之后，崩溃统计平台排行第一的总是一个奇怪的ANR，他的主线程卡在了这里：
```log
DALVIK THREADS (108):
"main" prio=5 tid=1 Waiting
  | group="main" sCount=1 dsCount=0 obj=0x74b59000 self=0xf4827800
  | sysTid=13783 nice=0 cgrp=default sched=0/0 handle=0xf73c9bec
  | state=S schedstat=( 20710331467 9075524599 28858 ) utm=1820 stm=251 core=5 HZ=100
  | stack=0xff139000-0xff13b000 stackSize=8MB
  | held mutexes=
  at java.lang.Object.wait!(Native method)
  - waiting on <0x031350a8> (a java.lang.Object)
  at java.lang.Thread.parkFor(Thread.java:1220)
  - locked <0x031350a8> (a java.lang.Object)
  at sun.misc.Unsafe.park(Unsafe.java:299)
  at java.util.concurrent.locks.LockSupport.park(LockSupport.java:157)
  at java.util.concurrent.locks.AbstractQueuedSynchronizer.parkAndCheckInterrupt(AbstractQueuedSynchronizer.java:813)
  at java.util.concurrent.locks.AbstractQueuedSynchronizer.doAcquireSharedInterruptibly(AbstractQueuedSynchronizer.java:973)
  at java.util.concurrent.locks.AbstractQueuedSynchronizer.acquireSharedInterruptibly(AbstractQueuedSynchronizer.java:1281)
  at java.util.concurrent.CountDownLatch.await(CountDownLatch.java:202)
  at android.app.SharedPreferencesImpl$EditorImpl$1.run(SharedPreferencesImpl.java:363)
  at android.app.QueuedWork.waitToFinish(QueuedWork.java:88)
  at android.app.ActivityThread.handleStopActivity(ActivityThread.java:3952)
  at android.app.ActivityThread.access$1200(ActivityThread.java:186)
  at android.app.ActivityThread$H.handleMessage(ActivityThread.java:1626)
  at android.os.Handler.dispatchMessage(Handler.java:111)
  at android.os.Looper.loop(Looper.java:194)
  at android.app.ActivityThread.main(ActivityThread.java:5905)
  at java.lang.reflect.Method.invoke!(Native method)
  at java.lang.reflect.Method.invoke(Method.java:372)
  at com.android.internal.os.ZygoteInit$MethodAndArgsCaller.run(ZygoteInit.java:1127)
  at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:893)
```
可以看到，主线程卡在了Activity的handleStopActivity处，正在等待Sharepreference的写入完成。这是什么情况？
## 0x02 Android源码分析（以android7.1.1源码为准）
### 1. 这件事情要从SharedPreference的apply方法说起
可以看到SharedPreferences是一个接口，他的实现在SharedPreferencesImpl里面，我们可以看以下代码：
```java
        public void apply() {
            final MemoryCommitResult mcr = commitToMemory();
            final Runnable awaitCommit = new Runnable() {
                    public void run() {
                        try {
                            mcr.writtenToDiskLatch.await();
                        } catch (InterruptedException ignored) {
                        }
                    }
                };

            QueuedWork.add(awaitCommit);//注意这里！！！

            Runnable postWriteRunnable = new Runnable() {
                    public void run() {
                        awaitCommit.run();
                        QueuedWork.remove(awaitCommit);
                    }
                };

            SharedPreferencesImpl.this.enqueueDiskWrite(mcr, postWriteRunnable);

            // Okay to notify the listeners before it's hit disk
            // because the listeners should always get the same
            // SharedPreferences instance back, which has the
            // changes reflected in memory.
            notifyListeners(mcr);
        }
```
看到那句QueuedWork.add(awaitCommit)了吗？他把一个Runnable放进了QueuedWork中，而这个Runnable在等待写入的成功返回。

### 2. QueuedWork是什么鬼？
还是看代码：
```java
   // The set of Runnables that will finish or wait on any async
   // activities started by the application.
    private static final ConcurrentLinkedQueue<Runnable> sPendingWorkFinishers =
            new ConcurrentLinkedQueue<Runnable>();

    /**
     * Add a runnable to finish (or wait for) a deferred operation
     * started in this context earlier.  Typically finished by e.g.
     * an Activity#onPause.  Used by SharedPreferences$Editor#startCommit().
     *
     * Note that this doesn't actually start it running.  This is just
     * a scratch set for callers doing async work to keep updated with
     * what's in-flight.  In the common case, caller code
     * (e.g. SharedPreferences) will pretty quickly call remove()
     * after an add().  The only time these Runnables are run is from
     * waitToFinish(), below.
     */
    public static void add(Runnable finisher) {
        sPendingWorkFinishers.add(finisher);
    }

    /**
     * Finishes or waits for async operations to complete.
     * (e.g. SharedPreferences$Editor#startCommit writes)
     *
     * Is called from the Activity base class's onPause(), after
     * BroadcastReceiver's onReceive, after Service command handling,
     * etc.  (so async work is never lost)
     */
    public static void waitToFinish() {
        Runnable toFinish;
        while ((toFinish = sPendingWorkFinishers.poll()) != null) {
            toFinish.run();
        }
    }
```
可以看到QueuedWork.add(Runnable)把一个Runnable放进了一个ConcurrentLinkedQueue，一个等待队列。然后还看到了上面trace中看到的熟悉的waitToFinish，是不是一切都一目了然了？
android在处理activity、service或广播的时候会执行waitToFinish，以确保之前加入等待队列sPendingWorkFinishers的异步处理能够执行到结束。SharedPreferences就写入了一个等待写入完成的runnable，由于写入性能差、数据量大等原因，可能会导致此处等待时间过长，从而出现ANR的情况。
## 0x03 如何解决？
1. 把apply改为在异步线程执行commit

    可以看源码：
    ```java
        public boolean commit() {
            MemoryCommitResult mcr = commitToMemory();
            SharedPreferencesImpl.this.enqueueDiskWrite(
                mcr, null /* sync write on this thread okay */);
            try {
                mcr.writtenToDiskLatch.await();
            } catch (InterruptedException e) {
                return false;
            }
            notifyListeners(mcr);
            return mcr.writeToDiskResult;
        }
    ```
    好处：没有将等待嫁接到Activity的onStop，很好避免了这个anr。

    缺点：只能修改本地代码，对于第三方SDK使用Apply导致的问题难以解决;需要考虑线程切换问题。
2. 避免同时修改多个值时使用多次提交
    
    根据《Android移动性能实战》里的说法，commit方法每调用一次就意味着一次文件的打开和关闭，从而造成因commit()方法的随意调用而导致文件的重复打开和关闭。
3. 通过hook的方式，清除sPendingWorkFinishers中的runnable

    目前商店正常尝试使用这个方法进行修复。
    实现方式主要是通过修改ActivityThread.mH变量，使用代理模式将其包装起来，拦截其中可能造成ANR的代码，在这些时刻清空QueuedWork的等待队列。
    具体代码参考如下：
    ```java
    /**
     * 修复由Sharepreferences导致的ANR
     * hook ActivityThread#mH
     */
    public static void tryHookActityThreadH() {
        boolean hookSuccess = false;
        try {
            Class activityThread = Class.forName("android.app.ActivityThread");
            Method mH = ReflectionCache.build().getMethod(activityThread, "currentActivityThread");
            if (mH != null) {
                Object obj = mH.invoke(activityThread);
                if (obj != null) {
                    Handler handler = (Handler) ReflectHelper.getField(obj, "mH");
                    if (handler != null) {
                        Field mCallbackField = ReflectionCache.build().getDeclaredField(Class.forName("android.os.Handler"), "mCallback");
                        if (mCallbackField != null) {
                            mCallbackField.setAccessible(true);
                            ActivityThreadHCallbackProxy activityThreadHandler = new ActivityThreadHCallbackProxy((Handler.Callback) mCallbackField.get(handler));
                            mCallbackField.set(handler, activityThreadHandler);
                            hookSuccess = true;
                        }
                    }
                }
            }
        } catch (Exception e) {
            Mlog.tag(TAG).i("HookActityThreadH:{}", e.getLocalizedMessage());
        }
        Mlog.tag(TAG).i("HookActityThreadH:{}", hookSuccess);
    }


    public class ActivityThreadHCallbackProxy implements Handler.Callback {

        public static final String TAG = "ActivityThreadHCallbackProxy";
        /**
        * {@link ActivityThread#H}
        */
            public static final int PAUSE_ACTIVITY          = 101;
            public static final int PAUSE_ACTIVITY_FINISHING= 102;
            public static final int STOP_ACTIVITY_SHOW      = 103;
            public static final int STOP_ACTIVITY_HIDE      = 104;
            public static final int SERVICE_ARGS            = 115;
            public static final int STOP_SERVICE            = 116;
            public static final int SLEEPING                = 137;


        private Handler.Callback mRawCallback;

        public ActivityThreadHCallbackProxy(Handler.Callback callback) {
            mRawCallback = callback;
        }

        @Override
        public boolean handleMessage(Message message) {
            switch (message.what) {
                case STOP_ACTIVITY_HIDE:
                case STOP_ACTIVITY_SHOW:
                    //stop activity
                    beforeWaitToFinished();
                    break;
                case SERVICE_ARGS:
                    //SERVICE ARGS
                    beforeWaitToFinished();
                    break;
                case STOP_SERVICE:
                    //STOP SERVICE
                    beforeWaitToFinished();
                    break;

                case SLEEPING:
                    //SLEEPING
                    beforeWaitToFinished();
                    break;
                case PAUSE_ACTIVITY:
                case PAUSE_ACTIVITY_FINISHING:
                    //pause activity
                    beforeWaitToFinished();
                    break;
                default:
                    break;
            }
            if (mRawCallback != null) {
                mRawCallback.handleMessage(message);
            }
            return false;//不能返回true，否则会消耗掉事件
        }

        private void beforeWaitToFinished() {
            QuenedWorkProxy.cleanAll();
        }
    }

    public class QuenedWorkProxy {
        private static final String TAG = "QuenedWorkProxy";
        private static final String CLASS_NAME = "android.app.QueuedWork";
        private static final String FILE_NAME_PENDDING_WORK_FINISH = "sPendingWorkFinishers";

        public static Collection<Runnable> sPendingWorkFinishers = null;
        private static boolean sSupportHook = true;

            /**
            * 不支持android O
            * android O变量名改为sFinishers
            */
        public static void cleanAll(){
                if (sPendingWorkFinishers == null && sSupportHook) {
                try {
                   sPendingWorkFinishers = (ConcurrentLinkedQueue<Runnable>) ReflectHelper.getStaticField(CLASS_NAME, FILE_NAME_PENDDING_WORK_FINISH);
                    } catch (Exception e) {
                        Mlog.tag(TAG).w("{}", e.getLocalizedMessage());
                        sSupportHook = false;
                    }
                }
                if(sPendingWorkFinishers != null){
                    Mlog.tag(TAG).d("clean QuenedWork.sPendingWorkFinishers({}) size {}" , sPendingWorkFinishers.hashCode() , sPendingWorkFinishers.size());
                    sPendingWorkFinishers.clear();
                }
            }
    }
    ```
在Application启动的时候调用tryHookActityThreadH即可实现Hook的效果，需要注意的是sPendingWorkFinishers在android O及以上改为了一个名为sFinishers的LinkedList，但是效果是一样的。这里没有针对android O进行适配。
