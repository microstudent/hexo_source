---
title: Android性能优化之使用AOP结合Systrace查找性能短板
date: 2018-08-19 15:45:22
tags: Android
---
# 0x01 什么是AOP？怎么使用Systrace？
AOP意即面向切面编程，通过预编译方式和运行期动态代理实现程序功能的统一维护的一种技术。
我在Android上使用AOP主要是为了做性能检测，Systrace默认没有针对所有方法进行跟踪，如果需要针对特殊方法进行跟踪，那么需要在java代码里面插入begin和endTrace等方法。这样不仅破坏了原有代码结构，而且难以对所有方法做追踪。
AspectJ做到了不侵入代码就可以插入追踪代码。
AOP有几个关键词：
1. Join Points即是程序运行的一个执行点，包括函数执行、函数调用、构造方法调用等等都是一个Join Points。
2. Advice Advice就是我们插入的代码以何种方式插入，有Before还有After、Around。
3. 自定义Pointcuts可以让我们更加精确的切入一个或多个指定的切入点。

# 0x02 准备工作
首先需要在根目录下build.gradle文件添加classpath:
```gradle
classpath 'org.aspectj:aspectjtools:1.8.9'
```

在各module的build.build下引入AspectJ:
```gradle
compile 'org.aspectj:aspectjrt:1.8.9'
```
同时添加以下gradle脚本:
```gradle
import org.aspectj.bridge.IMessage
import org.aspectj.bridge.MessageHandler
import org.aspectj.tools.ajc.Main

final def log = project.logger
final def variants = project.android.applicationVariants

//在构建工程时，执行编织
variants.all { variant ->
    if (!variant.buildType.isDebuggable()) {
        //非debug版本跳过
        log.debug("Skipping non-debuggable build type '${variant.buildType.name}'.")
        return;
    }

    JavaCompile javaCompile = variant.javaCompile
    javaCompile.doLast {
        String[] args = ["-showWeaveInfo",
                     "-1.5",
                     "-inpath", javaCompile.destinationDir.toString(),
                     "-aspectpath", javaCompile.classpath.asPath,
                     "-d", javaCompile.destinationDir.toString(),
                     "-classpath", javaCompile.classpath.asPath,
                     "-bootclasspath", project.android.bootClasspath.join(File.pathSeparator)]
        log.debug "ajc args: " + Arrays.toString(args)

        MessageHandler handler = new MessageHandler(true)
        new Main().run(args, handler)
        for (IMessage message : handler.getMessages(null, true)) {
           switch (message.getKind()) {
                case IMessage.ABORT:
                case IMessage.ERROR:
                case IMessage.FAIL:
                    log.error message.message, message.thrown
                    break
                case IMessage.WARNING:
                    log.warn message.message, message.thrown
                    break
                case IMessage.INFO:
                    log.info message.message, message.thrown
                    break
                case IMessage.DEBUG:
                    log.debug message.message, message.thrown
                    break
            }
        }
    }
}
```

# 0x03编写AspectJ切面代码
废话不多说,直接看代码:
1. 首先可以定义一个注解
    ```java
    package com.leaves.tools;
    import java.lang.annotation.ElementType;
    import java.lang.annotation.Retention;
    import java.lang.annotation.RetentionPolicy;
    import java.lang.annotation.Target;

    @Target(ElementType.METHOD)
    @Retention(RetentionPolicy.RUNTIME)
    public @interface SysTrace {
    }
    ```
    这个注解可以配合切面代码，将Trace的跟踪代码编织到方法的起始和终止时。例如可以这么使用：
    ```java
        @SysTrace
        public static void test() {
            //AspectJ add Trace.beginSection(TAG); before
            doSomeThing();
            //AspectJ add Trace.endSection(); after
        }
    ```
    这样便可以快速在某个需要跟踪的方法处添加Systrace的跟踪代码了！

2. 编写切面代码
    ```java
    import org.aspectj.lang.ProceedingJoinPoint;
    import org.aspectj.lang.Signature;
    import org.aspectj.lang.annotation.Around;
    import org.aspectj.lang.annotation.Aspect;
    import org.aspectj.lang.annotation.Pointcut;

    import android.os.SystemClock;
    import android.os.Trace;
    import android.util.Log;

    @Aspect
    public class MstoreAspect {
        public static final String TAG = "MstoreAspect";
        public static boolean sPrintLog = false;
        public static boolean sNeedTrace = true;
        public static final long MS = 1000000L;


        //以下定义了2个POINTCUT语句，分别代表在不同的POINTCUT进行织入。
        //具体语法可以参考https://blog.csdn.net/kq721/article/details/70255855
        public static final String POINTCUT_ALL_METHOD =
                "(" +
                        "execution(* *.*(..)) ||" +
                        "execution(* *(..))" + // 所有方法
                        ")";

        // 方法调用前后
        public static final String POINTCUT_ANNOTATION_SYSTRACE_METHOD =
                "(" +
                        "execution(@com.leaves.tools.SysTrace * *(..))" +
                        ")";

        @Pointcut(POINTCUT_ALL_METHOD+"||"+POINTCUT_ANNOTATION_SYSTRACE_METHOD)
        public void allMethod() {}

        /**
        * advice
        **/
        @Around("allMethod()")
        public Object weaveJoinPoint(ProceedingJoinPoint point) throws Throwable{
            Object result = null;
            long time = 0;
            boolean needTrace = sNeedTrace;
            boolean needTime = sPrintLog;
            Signature signature = point.getSignature();
            String className = signature.getDeclaringType().getName();
            String traceTag = String.format("m_%s_%s_%s",
                    className.substring(className.lastIndexOf(".") + 1), // 类名
                    signature.getName(), // 方法名
                    point.getSourceLocation().getLine() // 行号
            );
            if(signature.getName().startsWith("access$")){
                needTrace = false;
            }
            if (needTime) {
                time = SystemClock.elapsedRealtimeNanos();
            }
            if (needTrace) {
                Trace.beginSection(traceTag);
            }
            result = point.proceed();
            if (needTrace) {
                Trace.endSection();
            }
            if (needTime) {
                time = SystemClock.elapsedRealtimeNanos() - time;
                String logString = String.format("%s: %s", traceTag, (time / (float)MS));
                if (time > 32 * MS){
                    Log.e(TAG, logString);
                } else if (time > 16 * MS){
                    Log.w(TAG, logString);
                } else if (time > 8 * MS) {
                    Log.i(TAG, logString);
                } else if (time > 4 * MS) {
                    Log.d(TAG, logString);
                } else {
                    Log.v(TAG, logString);
                }
            }
            return result;
        }
    }
    ```

