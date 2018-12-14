---
title: okhttp3请求头中含中文报错原因及解决方案
date: 2018-12-5 15:26:14
categories: Android常见问题
tags: Android, 常见问题
---

最近在负责做一个图片加载模块，测试过程中反馈一个问题：有两个测试设备上加载不了图片。我就纳闷了，我就一个加载图片模块怎么还跟机型适配扯上关系了。然后查了下日志异常如下：
```
java.lang.IllegalArgumentException: Unexpected char 0x8d2d at 13 in content-disposition value: filename="3.6购买页.jpg"
	at com.bumptech.glide.request.RequestFutureTarget.doGet(RequestFutureTarget.java:189)
	at com.bumptech.glide.request.RequestFutureTarget.get(RequestFutureTarget.java:100)
	at com.pptv.tvsports.common.disk.ImageDiskCache.lambda$putCacheImage$0$ImageDiskCache(ImageDiskCache.java:100)
	at com.pptv.tvsports.common.disk.ImageDiskCache$$Lambda$0.run(Unknown Source)
	at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1113)
	at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:588)
	at java.lang.Thread.run(Thread.java:818)
Caused by: java.lang.IllegalArgumentException: Unexpected char 0x8d2d at 13 in content-disposition value: filename="3.6购买页.jpg"
	at okhttp3.Headers$Builder.checkNameAndValue(Headers.java:283)
	at okhttp3.Headers$Builder.add(Headers.java:233)
	at okhttp3.internal.http.Http2xStream.readHttp2HeadersList(Http2xStream.java:263)
	at okhttp3.internal.http.Http2xStream.readResponseHeaders(Http2xStream.java:149)
	at okhttp3.internal.http.HttpEngine.readNetworkResponse(HttpEngine.java:723)
	at okhttp3.internal.http.HttpEngine.access$200(HttpEngine.java:81)
	at okhttp3.internal.http.HttpEngine$NetworkInterceptorChain.proceed(HttpEngine.java:708)
	at okhttp3.internal.http.HttpEngine.readResponse(HttpEngine.java:563)
	at okhttp3.RealCall.getResponse(RealCall.java:241)
	at okhttp3.RealCall$ApplicationInterceptorChain.proceed(RealCall.java:198)
	at okhttp3.SNInterceptor.intercept(SNInterceptor.java:62)
	at okhttp3.RealCall$ApplicationInterceptorChain.proceed(RealCall.java:187)
	at okhttp3.RealCall.getResponseWithInterceptorChain(RealCall.java:160)
	at okhttp3.RealCall.execute(RealCall.java:57)
	at com.pptv.tvsports.glide.MOkHttpStreamFetcher.loadData(MOkHttpStreamFetcher.java:51)
	at com.pptv.tvsports.glide.MOkHttpStreamFetcher.loadData(MOkHttpStreamFetcher.java:22)
	at com.bumptech.glide.load.engine.DecodeJob.decodeSource(DecodeJob.java:170)
	at com.bumptech.glide.load.engine.DecodeJob.decodeFromSource(DecodeJob.java:128)
	at com.bumptech.glide.load.engine.EngineRunnable.decodeFromSource(EngineRunnable.java:127)
	at com.bumptech.glide.load.engine.EngineRunnable.decode(EngineRunnable.java:106)
	at com.bumptech.glide.load.engine.EngineRunnable.run(EngineRunnable.java:58)
	at java.util.concurrent.Executors$RunnableAdapter.call(Executors.java:423)
	at java.util.concurrent.FutureTask.run(FutureTask.java:237)
	at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1113)
	at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:588)
	at java.lang.Thread.run(Thread.java:818)
	at com.bumptech.glide.load.engine.executor.FifoPriorityThreadPoolExecutor$DefaultThreadFactory$1.run(FifoPriorityThreadPoolExecutor.java:118)
```
其实从日志上，问题原因已经很明显，但是查找问题的时候我犯了个错误。就是没有根据堆栈信息查找问题，这是由于平时发现问题的时候习惯于定位应用的代码入口，而不是查看源码报错处。
当时看到这个异常的时候，第一反应是保存文件的时候使用中文出错，但是我写的代码中保存图片用的是自己定义的字符串，而这个filename在代码里根本查不到。所以这种情况下这个filename只能是通过图片url获取到的，然后我打开chrome调试，可以看到图片url的响应报头中有这个东西：
```
	Content-Disposition：filename="3.6购买页.jpg
```
Content-disposition是 MIME 协议的扩展，MIME 协议指示 MIME 用户代理如何显示附加的文件。当 Internet Explorer 接收到头时，它会激活文件下载对话框，它的文件名框自动填充了头中指定的文件名。
知道了filename是哪里来的之后，再去找发生问题的原因。在我的代码里我只调用了：
```
	File imageFile = Glide.with(mContext).load(url).downloadOnly(Target.SIZE_ORIGINAL, 				Target.SIZE_ORIGINAL).get();
```
所以当时我的理解是，glide在加载图片时内部缓存文件时因为filename报错。看了半天源码后，发现最终调用的是项目中自定义的DataFetcher的loadData方法，然后就是okhttp的正常请求调用了。其实整个调用链跟异常日志的堆栈信息是一样的。okhttp的详细调用略过，最终的问题出现在Http2xStream的readHttp2HeadersList方法，这里会读取response的header，问题在这个调用
```
	headersBuilder.add(name.utf8(), value);
```
okhttp3.Headers.java
```
    /** Add a field with the specified value. */
    public Builder add(String name, String value) {
      checkNameAndValue(name, value);
      return addLenient(name, value);
    }
	
	private void checkNameAndValue(String name, String value) {
      if (name == null) throw new IllegalArgumentException("name == null");
      if (name.isEmpty()) throw new IllegalArgumentException("name is empty");
      for (int i = 0, length = name.length(); i < length; i++) {
        char c = name.charAt(i);
        if (c <= '\u001f' || c >= '\u007f') {
          throw new IllegalArgumentException(String.format(
              "Unexpected char %#04x at %d in header name: %s", (int) c, i, name));
        }
      }
      if (value == null) throw new IllegalArgumentException("value == null");
      for (int i = 0, length = value.length(); i < length; i++) {
        char c = value.charAt(i);
        if (c <= '\u001f' || c >= '\u007f') {
          throw new IllegalArgumentException(String.format(
              "Unexpected char %#04x at %d in %s value: %s", (int) c, i, name, value));
        }
      }
    }
```
这里就是问题出现的原因，checkNameAndValue这个方法会对请求头的name、value进行校验。

##解决思路
既然发现了出现问题的原因，现在就是找解决方案，其实在网上搜索okhttp请求头中文，这个关键字也会搜到一些文章。但是这些给出的解决方案一般都是对请求头进行转码，因为一般这种问题都出现在前端request的时候，而我碰到的服务端返回的请求头中带中文。
出现这问题之后我首先是想让后端协助解决掉这个问题，但是跟后端沟通过后发现他也不知道这个filename是哪里来的，他没有对这块进行处理。出于各种原因他也不能去专门修改这个问题，同时他也指出你们使用的框架不支持请求头中文，本身就不合理。我听他这么一说也挺有道理，就放弃了这个想法。
既然靠后端修改走不通，我又查了下资料。既然filename有一个名字，那肯定是哪里传过去的，后台配置图片都配置的是英文，这中文必然是上传图片时存储在本地的文件名。然后跟产品和运营沟通了一下，果然这个命名是他们本地的。然后让他们修改本地文件名重新上传了一下，这问题算是解决了。
但是，问题肯定不能到这为止。如果其他人碰到这个问题，又没办法让在源头做修改，那这个问题如何解决？

下面说一下我个人的解决方案：
###方案一：使用拦截器（适用于发起request时）
这个方案比较常规，你也可以在出错的请求发起时对对应的request请求头进行转码。也可以在构造OkHttpClient时addInterceptor，然后在intercept中对request统一进行转码。具体代码就不赘述了，到处都能找到。但是需要注意的是，在intercept中是无法规避response中请求头有中文的，因为出错的位置在你通过chain.proceed(request)拿到response之前，这点可以在源码中看到后续我也会写文章讲okhttp整个流程。

###方案二：反射
这个方案是我自己使用的方案，源于分析代码的时候，问题出在readHttp2HeadersList的
```
	if (!HTTP_2_SKIPPED_RESPONSE_HEADERS.contains(name)) {
        headersBuilder.add(name.utf8(), value);
      }
```
这里的add方法，而我碰到的问题是某一个请求头会返回中文，并且客户端不需要这个请求头的参数。那既然这样，如果我去修改HTTP_2_SKIPPED_RESPONSE_HEADERS这个List，不就可以实现我需要的功能并且改动最小吗。具体代码如下：
```
	private boolean hookOkHttpReadHeader(){
        try {
            Class clz = Class.forName("okhttp3.internal.http.Http2xStream");
            Field field = clz.getField("HTTP_2_SKIPPED_RESPONSE_HEADERS");
            field.setAccessible(true);
            field.set(new ArrayList<>(), HTTP_2_SKIPPED_RESPONSE_HEADERS);
			return true;
        } catch (ClassNotFoundException e) {
            e.printStackTrace();
        } catch (NoSuchFieldException e) {
            e.printStackTrace();
        } catch (IllegalAccessException e) {
            e.printStackTrace();
        }
        return false;
    }
```
此方法适用于客户端不需要请求头中的参数的情况。

###方案三：方法替换
虽然我自己的问题解决了，但是我还是在想能不能去完美的解决这个问题而不是取巧，有没有一个方法能够实现替换Headers中的add方法实现，让add方法不去调用checkNameAndValue，或者是修改checkNameAndValue的内部实现。
想过之后突然发现这不就是热修复中的方法替换，andfix不就是通过替换方法指针达到修复的目的吗，这个情况跟我想要实现的一模一样。既然如此，可以使用andfix的方案解决问题。但是这样会导致包体积增大以及兼容性问题，而且就算有源码实现这个过程也是要耗费大量时间的，这里只是提供一种思路。

###方案四：自行编译
这个方案是你自己去pull okhttp源码，修改对应位置，然后生成依赖供自己使用。这样可以完整的规避这种问题。
