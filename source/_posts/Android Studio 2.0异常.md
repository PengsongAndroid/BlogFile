---
title: Android Studio 2.0异常
date: 2016-05-25 07:33:44
categories: 问题
tags: 问题
---
今天在导入同事的项目完成后没报错，运行的时候却报了一段错误，自己弄了会儿都没解决。

    :app:transformClassesWithInstantRunForDebug FAILED
	Error:Execution failed for task ':app:transformClassesWithInstantRunForDebug'.
	> Invalid signature file digest for Manifest main attributes

后来经过搜索发现解决方案，禁用2.0的快速启动新功能，按照下图取消勾选即可。
![alt text](http://7xrxl6.com1.z0.glb.clouddn.com/as%E5%BC%82%E5%B8%B8.png "")
