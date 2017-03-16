---
title: Android在应用退出后发送请求解决方案
date: 2017-03-15 17:07:44
categories: Android文章
tags: Android, 应用退出
---
  最近有这么一个需求，在app退出登录之后发送请求到后台，这个请求不是特别重要，只是为了应用过审。所以在常规情况下能够发送请求即可，下面方案不保证能够在所有情况下应用退出都能发送请求。
### 常规退出场景
常规情况下应用退出有这么几种情况：
1.app内退出按钮或者双击返回；
2.最近应用列表，划掉应用卡片（常见）；
3.应用崩溃；
我们一个个来分析解决。
<!-- more -->
### 解决方案
#### 对于app内部退出
这个是我们可控的。但是用户在退出时发请求需要考虑到网络情况，肯定不可能等到请求成功才退出应用。而我们app退出是有下面的操作：

```java
	//系统退出 清空所有缓存 取消所有请求
	UenUtils.cleanAppConfigWhenExit();
	MyApplication.cancelAllPostRequests();
	ActivityStack.popAll();
```

最后一步内其实是杀掉了当前进程

```java
	android.os.Process.killProcess(android.os.Process.myPid());
```

因为这一步应用是被杀掉的，肯定是不能在这里做发请求操作，只能开启service来帮我们完成操作。
具体的实现方案就是，在应用启动时startService，这是一个远端的service

```java
    <service
    	android:name=".service.MyService"
    	android:process=":remote"
    	android:enabled="true">
    	<intent-filter>
    		<category android:name="android.intent.category.DEFAULT" />
    		<action android:name="MyService" />
    	</intent-filter>
    </service>
```

然后我们在应用退出的时候，发送一个广播（因为这个进程间的通信很简单，所以就用这种比较方便的方式），service中处理广播，处理完成后销毁自己，思路就是这样。关键部分代码如下：
activity:
```java
    @Override
    public boolean onKeyDown(int keyCode, KeyEvent event) {
        Intent intent = new Intent();
        intent.setAction("test");
        sendBroadcast(intent);
        android.os.Process.killProcess(android.os.Process.myPid());
        return super.onKeyDown(keyCode, event);
    }
```

service：
```java
    @Override
    public void onCreate() {
        Log.i(TAG, "onCreate ");
        super.onCreate();
        IntentFilter filter = new IntentFilter();
        filter.addAction("test");
        registerReceiver(receiver, filter);
    }
	
    BroadcastReceiver receiver = new BroadcastReceiver() {
        @Override
        public void onReceive(Context context, Intent intent) {
            if ("test".equals(intent.getAction())){
                Log.e(TAG, "onReceive");
                /**发请求*/
                ....
                /**请求成功后解绑 */
                unregisterReceiver(receiver);
                android.os.Process.killProcess(android.os.Process.myPid());
            }
        }
    };
```

#### 对于在任务列表划掉应用
查了一些资料之后发现Service中有一个回调方法
```java
    @Override
    public void onTaskRemoved(Intent rootIntent) {

    }
```
官方文档解释如下，大意就是如果这个service在运行并且用户移除了这个任务，会回调这个方法，但是如果你设置了FLAG_STOP_WITH_TASK这个属性，你将不会接收到这个回调，并且service会直接停止。

![enter image description here](http://7xrxl6.com1.z0.glb.clouddn.com/develop_ontask1.png)
提到了FLAG_STOP_WITH_TASK这个属性，我们来看看这是什么：
![enter image description here](http://7xrxl6.com1.z0.glb.clouddn.com/develop_ontask2.png)
如果设置了该属性，用户删除了基于某个应用程序的任务，系统将自动停止该服务，然后这个是通过stopWithTask属性来控制的。
所以我们在service中配置，添加一行

```java
    android:stopWithTask="true"
```

然后我们测试一下，启动服务后从任务列表移除应用，方法确实被回调。我们可以在这个方法内发送请求，不过需要注意的是，我测试的机器上，移除应用后方法回调然后service就挂掉了，1s左右service又重启了，走了onCreate、onStartCommand回调。我是直接在回调方法里写请求，这样会出现接收不到请求的返回的情况。所以我建议的方式是，在回调方法里写一个SharedPreferences，然后再重新创建的时候再去通过读取这个值来发送请求。具体实现代码如下：
```java
    // 配置文件
    private static SharedPreferences g_settings = null;
    private static SharedPreferences.Editor g_editor = null;
    
    @Override
    public void onCreate() {
        super.onCreate();
        g_settings = getSharedPreferences("ps", Context.MODE_APPEND);
        test = g_settings.getBoolean("test", false);
        Log.i(TAG, "onCreate " + test);
        if (test){
            g_editor.putBoolean("test", false);
            g_editor.commit();
			/**发请求 然后销毁service*/
			...
        }
        g_editor = g_settings.edit();
    }
    
    @Override
    public void onTaskRemoved(Intent rootIntent) {
        Log.e(TAG, "onTaskRemoved");
        g_editor.putBoolean("test", true);
        g_editor.commit();
        super.onTaskRemoved(rootIntent);
    }
```

#### 对于异常崩溃的情况
可以在application中实现UncaughtExceptionHandler，然后在回调方法中发送广播，思路跟上面的差不多。
具体的实现就是这些，只写了重要部分的代码，其他的也都很简单就不贴出来了。

