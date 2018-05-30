---
layout: post
title: broadcast相关
date: 2018-05-30
img: blog_head_7.jpeg 
tags: [Android, broadcast]
---

**onreceiver中执行耗时操作**
不要在onreceiver中执行耗时操作，也不要开启新线程开执行这个操作，在broadcast中的代码执行位于其进程的主线程中，执行时间不要超过5s，否则会弹出超市dialog。但是如果执行不完，也不能放在线程中，Receiver只在onReceiver方法执行时是激活状态。因为开启线程——>onReceiver返回——>receiver不再处于激活状态——>receiver进程变为空进程将会在任意时刻被终止——>正在工作的子线程也会被杀死  
解决这个问题的方案是在onReceive()里开始一个Service，让这个Service去 做这件事情，那么系统就会认为这个进程里还有活动正在进行。  
 
**执行优先级**   
冷注册（权限文件）和热注册（代码中跟随activity生命周期）  
对于有序消息，热注册的BroadcastReceiver总是先于冷注册的BroadcastReceiver被触发。对于同样是动态注册的BroadcastReceiver，优先级别高（android:priority）的将先被触发，而静态注册的 BroadcastReceiver总是按照冷注册的顺序执行。

**与其他app的干扰**：  
1. 任何应用都可以向该registered receiver发送广播  
（1）可以通过对广播添加权限来控制。  
	a. 定义新权限：<permission android:name = "com.android.permission.SEND_XXX"/>   
	b. 声明权限 <uses-permission android:name="com.android.permission.SEND_XXX"></uses-permission>   	c.在<receiver>tag里添加权限SEND_XXX的声明。这样便只能接收来自具有该SEND_XXX权限的应用发出的广播。  
	（2）export属性为false。  
【exported属性决定了是否能收到应用外部的消息，在没有intent filters时，意味着只能被特定的知名了具体类名的intent对象激发，也就是说只会被内部应用使用，也就是说默认值为false，而如果有了intent filters，以为这也就意味着可以接收系统或外部的应用，默认值为true。】  
2. 本应用的广播会被其他应用接受到，解决办法  
（1）添加权限
	a. 定义新权限：<permission android:name = "com.android.permission.RECV_XXX"/>   
	 b. 使用<uses-permission android:name="com.android.permission.RECV_XXX"></uses-permission>   c. sendbroadcast时传入权限sendbroadcast("com.android.XXX_ACTION", "com.android.permission.RECV_XXX")   
	 （2）使用本地广播 LocalBroadcastManager
