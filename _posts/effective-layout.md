---
layout: post
title:  "effective layout"
date:   2017-01-03 14:15:00 +0800
categories: book
---

**android 了解内存使用情况**

<!--more-->

原文链接 [http://blog.csdn.net/guolin_blog/article/details/42238633](http://blog.csdn.net/guolin_blog/article/details/42238633)

# 了解手机堆大小（Heap Size）, Nexus5的堆大小192MB

调用如下代码获取,获取手机堆大小:

```
ActivityManager manager = (ActivityManager)getSystemService(Context.ACTIVITY_SERVICE);  
int heapSize = manager.getMemoryClass();
```

结果是以MB为单位进行返回的，我们在开发应用程序时所使用的内存不能超出这个限制，否则就会出现OutOfMemoryError。因此，比如说我们的程序中需要缓存一些数据，就可以根据堆大小来决定缓存数据的容量。


# Android GC(Garbage Collection)原理

Android系统会在适当的时机触发GC操作，一旦进行GC操作，就会将一些不再使用的对象进行回收。

![](https://raw.githubusercontent.com/dreaminglion/dreaminglion.github.io/master/images/AnalyseMemory_GC.jpg)

可以看到，目前所有黄色的对象仍然会被系统继续保留，而蓝色的对象就会在GC操作当中被系统回收掉了，这大概就是Android系统一次简单的GC流程。

系统每进行一次GC操作时，都会在LogCat中打印一条日志，我们只要去分析这条日志就可以了，日志的基本格式如下所示：

```
D/dalvikvm: <GC_Reason> <Amount_freed>, <Heap_stats>,  <Pause_time>
```
首先第一部分GC_Reason，这个是触发这次GC操作的原因，一般情况下一共有以下几种触发GC操作的原因：

- GC_CONCURRENT:   当我们应用程序的堆内存快要满的时候，系统会自动触发GC操作来释放内存。
- GC_FOR_MALLOC:   当我们的应用程序需要分配更多内存，可是现有内存已经不足的时候，系统会进行GC操作来释放内存。
- GC_HPROF_DUMP_HEAP:   当生成HPROF文件的时候，系统会进行GC操作，关于HPROF文件我们下面会讲到。
- GC_EXPLICIT:   这种情况就是我们刚才提到过的，主动通知系统去进行GC操作，比如调用System.gc()方法来通知系统。或者在DDMS中，通过工具按钮也是可以显式地告诉系统进行GC操作的。

接下来第二部分Amount_freed，表示系统通过这次GC操作释放了多少内存。

然后Heap_stats中会显示当前内存的空闲比例以及使用情况（活动对象所占内存 / 当前程序总内存）。


下面是一次GC操作在LogCat中打印的日志：

![](https://raw.githubusercontent.com/dreaminglion/dreaminglion.github.io/master/images/AnalyseMemory_dalvikvm.jpg)

那么这是使用dalvik运行环境时所打印的GC日志，而自Android 4.4版本之后加入了art运行环境，在art中打印GC日志基本和dalvik是相同的，如下图所示：

![](https://raw.githubusercontent.com/dreaminglion/dreaminglion.github.io/master/images/AnalyseMemory_art.jpg)


# 通过DDMS中提供的工具,更加清楚地实时知晓当前应用程序的内存使用情况.

打开DDMS界面，在左侧面板中选择你要观察的应用程序进程，然后点击Update Heap按钮，接着在右侧面板中点击Heap标签，之后不停地点击Cause GC按钮来实时地观察应用程序内存的使用情况即可，如下图所示：

![](https://raw.githubusercontent.com/dreaminglion/dreaminglion.github.io/master/images/AnalyseMemory_Monitor.png)

接着继续操作我们的应用程序，然后继续点击Cause GC按钮，如果你发现反复操作某一功能会导致应用程序内存持续增高而不会下降的话，那么就说明这里很有可能发生内存泄漏了。

# Android中内存泄漏的问题
Android中的垃圾回收机制并不能防止内存泄漏的出现，导致内存泄漏最主要的原因就是某些长存对象持有了一些其它应该被回收的对象的引用，导致垃圾回收器无法去回收掉这些对象，那也就出现内存泄漏了。比如说像Activity这样的系统组件，它又会包含很多的控件甚至是图片，如果它无法被垃圾回收器回收掉的话，那就算是比较严重的内存泄漏情况了。

# 分析OOM奔溃原因见原文,使用内存分析工具，叫做Eclipse Memory Analyzer（MAT）.

下载地址: [http://eclipse.org/mat/downloads.php](http://eclipse.org/mat/downloads.php)


# 高性能编码优化

*原文链接：* [http://blog.csdn.net/guolin_blog/article/details/42318689](http://blog.csdn.net/guolin_blog/article/details/42318689)

使用合适的算法与数据结构将永远是你优化程序性能的最主要手段，但本篇文章中不会讨论这一块的内容。因此，这里我们即将学习的并不是什么灵丹妙药，而是大家应该把这些技巧当作一种好的编码规范，我们在平时写代码时就可以潜移默化地使用这些编码规范，不仅能够在微观层面提升程序一定的性能，也可以让我们的代码变得更加专业。

- 避免创建不必要的对象
- 静态优于抽象
- 对常量使用static final修饰符
- 多使用系统封装好的API
- 使用增强型for循环语法，不建议使用zero()

```
public void zero() {  
    int sum = 0;  
    for (int i = 0; i < mArray.length; ++i) {  
        sum += mArray[i].mCount;  
    }  
}  

public void one() {  
    int sum = 0;  
    Counter[] localArray = mArray;  
    int len = localArray.length;  
    for (int i = 0; i < len; ++i) {  
        sum += localArray[i].mCount;  
    }  
}  

public void two() {  
    int sum = 0;  
    for (Counter a : mArray) {  
        sum += a.mCount;  
    }  
```
