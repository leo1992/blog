---
layout: post
title: 图片框架优缺点整理（Fresco和Glide）
date: 2018-05-18
img: blog_head_1.jpeg 
tags: [Android, 图片加载]
---


在功能上，Picasso无法支持gif，在加载时会加载全尺寸的图片到内存，因为一般项目中都会有gif图的加载，所以暂时不考虑Picaso。不存在384和重复请求url的问题

# Fresco 
优点：  
1. Fresco会**自动复用相同URL的缓存**，两个相同URL同时进行请求Fresco也只会进行一次网络请求，然后第二个进行复用。  
2. Fresco 中设计有一个叫做 **Drawees 模块**，方便地显示loading图，当图片不再显示在屏幕上时，及时地释放内存和空间占用。  
3. 在5.0以下系统，Fresco将图片放到一个**特别的内存区域**。这会使得APP更加流畅，减少因图片内存占用而引发的OOM。在更底层的Native层对OOM进行处理，图片将不再占用App的内存  
4. 支持**渐进式的图片呈现**    
5. 在加载gif图中，Fresco的**java heap基本保持较低平稳状态**，而Glide的java heap基本为Fresco的一倍。但是nativheap中Fresco高出很多
6. 支持**圆角图**，（下载失败之后点击重新下载，自定义展位图，overlay或者进度条，指定用户按压时的overlay）
7. 加载上：Fresco 的 image pipeline 设计，允许用户在多方面控制图片的加载：为同一个图片指定不同的远程路径，或者使用已经存在本地缓存中的图片。加载完成回调通知。对于本地图，如有EXIF缩略图，在大图加载完成之前，可先显示缩略图，缩放或者旋转图片，处理已下载的图片  
问题：  
1. 当Fresco加载384张图片以后，就无法显示出图片了。当activity销毁数值会释放的。  
2. Fresco依赖的库很多  
3. 布局依赖SimpleDraweeView控件

# Glide  
优点：  
1. 除了支持gif和webP图片，还支持缩略图和video（可以将任何的本地视频解码成一张静态图片）；  
2. 生命周期集成（根据Activity或者Fragment的生命周期管理图片加载请求；方便处理bitmap，比如 Paused状态在暂停加载，在Resumed的时候又自动重新加载。）    
3. 有高效的缓存策略，**支持原始图片和结果图片的缓存**，而Picasso和Fresco只会缓存原始尺寸的图片，**由于不需要再次处理图片大小，加载比较快**。
4. 加载速度在**静态图上是比Fresco快**的，在内存上，javaheap内存开销比Fresco大，但是NATIVE HEAP上不大。  
5. Glide可以很容易的获取Bitmap  
问题：  
1. 相同URL的不同控件之间无法复用缓存。确认  
2. 在加载gif图时时间比较慢  
3. Glide没有文件缓存  
4. Glide加载的图片没有Picasso那么平滑  
5. Glide只有占位图  
6. 没有缓存进度  
  
默认Bitmap格式是RGB_565 

来源：  
https://blog.csdn.net/AIR_GRU/article/details/71081193
https://blog.csdn.net/github_33304260/article/details/70213300
https://blog.csdn.net/github_33304260/article/details/54140164

