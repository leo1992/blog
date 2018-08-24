---
layout: post
title: something interesting
date: 2018-08-24
img: blog_head_2.jpeg 
tags: [Android]
---

# findViewById
findviewbyid用来根据id找到对应的view组件，可以直接调用activity的findviewbyid方法，也可以通过一个view的findviewbyid方法找到。这两个有什么区别？  
activity的方法，是通过getwindow--getdecorview的findviewbyid方法去查找的，decorview是最顶层的父控件，所以最终是通过view的getviewbyid方法去查找的。findviewbyid没有再被重写，但是其中调用的findViewTraversal会被viewgroup重写，在这个方法中，viewgroup会在自己的children中遍历查找id，这也是一个树形遍历的过程，每个child也会去找自己的children，最终到view就是去比较view的id跟当前的id是不是匹配。由此可以看出，parent的层级锁定的约低，查找的view就最少。

# SharedPreferences
SharedPreferences是用来做一些简单数据的存储共享的，线程安全的，其实现类是SharedPreferencesImpl，在创建的时候会去load数据，因为sharedPreferences中的内容都是通过xml文件存储的，这是个比较耗时的操作，所以会开启新线程去做这件事，load的时候会加对象锁，加载后会notifyall，对应每次读取数据前awaitLoadedLocked的wait，等到读取完成之后才能返回数据。
getSharedPreferences的方法在context中，其实现最终是在ContextImpl中，其获取的时的name与file一一对应，放在context的mSharedPrefsPaths，而file与SharedPreference对应，
1. 开新线程觉得耗时问题
2. 线程安全，进程不安全。

-------------------------
之所以看这部分的源码是因为一个面试官说到SharedPreferences的关联启动优化问题，所以简单看了load部分的源码，系统已经做过开启线程的处理，所以在启动的时候在最早去做一次开启新线程load的处理，并不会起到优化的作用啊？
不过这里面提到了另一个生命周期的问题，在onCreate里面去做这个处理已经比较晚了，所以有没有比oncreate更早的时候，搜了一下，大概是这个过程：  
main()->Application:attachBaseContext()->onCreate()->Activity:onCreate()->onStart()->onPostCreate()->onResume()->onPostResume()。可以在attachBaseContext里面做这个操作。
关联----multidex 和  启动优化的相关知识。


view设置gone时，view在layout布局文件中不占用位置，但是该view还是会创建对象，会被初始化，会占用资源。
