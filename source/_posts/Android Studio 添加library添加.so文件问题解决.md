---
title: Android Studio 添加library添加.so文件问题解决
date: 2016-05-27 07:33:44
categories: 问题
tags: 问题
---
今天在Android studio中导入library和.so时发生各种异常，记录一下解决过程。
## 导入library
首先在项目中import project，如图

![alt text](http://7xrxl6.com1.z0.glb.clouddn.com/import_project.png "project")

需要注意的是，如果library是eclipse项目，需要先在eclipse中导出为AS项目。

然后在Project Structure选项中查看项目的Dependencies，点击右上角的加号，选择Module dependency,选择刚刚导入的library，点击ok让项目自动构建即可。

导入module

![alt text](http://7xrxl6.com1.z0.glb.clouddn.com/import_moudle.png "module")

等待构建完毕之后，我Run的时候却发现很多报错信息，反复修改各种配置，错误信息也是一会一个，发现错误信息全部集中在 android/support/v4 这个依赖上，错误信息如下：

    Error:Execution failed for task ':transformClassesWithJarMergingForDebug'.
	> com.android.build.api.transform.TransformException: java.util.zip.ZipException: duplicate entry: android/support/v4/content/Loader$OnLoadCompleteListener.classError:Execution failed for task ':transformClassesWithJarMergingForDebug'.

	Error:Execution failed for task ':transformClassesWithJarMergingForDebug'.
	> com.android.build.api.transform.TransformException: java.util.zip.ZipException: duplicate entry: android/support/v4/graphics/drawable/DrawableCompat$BaseDrawableImpl.class

	Error:Execution failed for task ':transformClassesWithJarMergingForDebug'.
	> com.android.build.api.transform.TransformException: java.util.zip.ZipException: duplicate entry: android/support/v4/accessibilityservice/AccessibilityServiceInfoCompat.class

发现了问题出在哪里之后，就从这里入手，一顿搜索，发现应该是存在重复的jar包。想起来导入的library是用的本地的support-v4 jar包，原本的项目也用的是本地的support-v4 jar包，觉得问题可能就是在这。然后删除了两个项目本地的jar包，通过gradle来导入。两个项目的gradle都添加。

	compile 'com.android.support:support-v4:22.1.1'
然后等待项目构建完成，再次Run，没有问题。不知道为什么通过gradle添加依赖就能解决，不过已经用了AS，最好还是按照AS的配置来构建项目，避免不必要的问题。

## 导入.so文件
上面导入的library项目，还需要在原项目中添加.so文件，两个文件夹下各有一个.so文件，添加到项目中。
![enter image description here](http://7xrxl6.com1.z0.glb.clouddn.com/so.png)

运行时报错如下：

    java.lang.UnsatisfiedLinkError: Native method not found: com.baidu.platform.comjni.map.commonmemcache.JNICommonMemCache.Create:()J
	at com.baidu.platform.comjni.map.commonmemcache.JNICommonMemCache.Create(Native Method)
	at com.baidu.platform.comjni.map.commonmemcache.a.a(Unknown Source)
	at com.baidu.platform.comapi.e.c.b(Unknown Source)
	at com.baidu.mapapi.a.c(Unknown Source)
	at com.baidu.mapapi.SDKInitializer.initialize(Unknown Source)
提示跟百度地图的Native方法找不到了，我仅仅添加了.so文件结果导致百度地图报错。排查了几次之后，又去百度官网看文档，发现demo中lib目录下各种cpu类型目录底下都会有同名的.so文件，怀疑是因为我只在armeabi文件夹下添加了百度地图的.so库，新建了armeabi-v7a文件夹后没有添加百度地图的文件，导致去这个文件夹下加载不
到.so库，然后复制了一份百度地图的.so库至armeabi-v7a文件夹，成功解决。