---
title: Android注解实践——运行时注解框架
date: 2017-04-9 18:45:13
categories: Android文章
tags: Android, Annotation, 注解, 运行时注解, EventBus
---

之前写了一篇编译期注解的文章，里面有提到注解作用的三种时期，SOURCE是会被编译器丢弃的注解，CLASS是在编译器保存，RUNTIME是在运行期保留。网上大多的资料讲的也都是编译期的注解，因为这是一种不影响效率的方式，也是很多热门开源库采用的方式。但是有些时候编译期注解并不能帮我们解决碰到的问题，这个时候如果了解运行时注解，也许就能解决问题。基础的概念前一篇已经讲过，这里就不再介绍了。

<!-- more -->

#### 实现原理
运行时注解操作的原理就是运用反射，去获取到我们注解所操作的元素（Filed、Method...），然后再去获得Annotation进行操作，完成我们的目的。
主要运用的有这些方法:
* **<T extends Annotation> T getAnnotation(Class<T> annotationClass)**
返回改程序元素上存在的、指定类型的注解，如果该类型注解不存在，则返回null。
* **Annotation[] getAnnotations()**
返回该程序元素上存在的所有注解。
* **boolean is AnnotationPresent(Class<?extends Annotation> annotationClass)**
判断该程序元素上是否包含指定类型的注解，存在则返回true，否则返回false。
* **Annotation[] getDeclaredAnnotations()**
返回直接存在于此元素上的所有注释。与此接口中的其他方法不同，该方法将忽略继承的注释。（如果没有注释直接存在于此元素上，则返回长度为零的一个数组。）该方法的调用者可以随意修改返回的数组；这不会对其他调用者返回的数组产生任何影响。

理论虽然简单，但是抽象，下面具体讲讲如何简单的实现。

#### 举个栗子
##### [EventBus](https://github.com/greenrobot/EventBus)
EventBus大家应该都不陌生，它是一个基于观察者模式的事件发布/订阅框架，开发者可以通过极少的代码去实现多个模块之间的通信，而不需要以层层传递接口的形式去单独构建通信桥梁。这个库的优秀以及流行更是因为它使用方便、性能高、支持多线程等各种优点。
这个库里面也运用了不少注解的技术，这篇文章就略微的讲一下EventBus是如何通过注解，指定不同线程去处理消息的。(以下源码都在是EventBus 3.0.0版本下)
```java
    @Subscribe(threadMode = ThreadMode.BACKGROUND) //在ui线程执行
    public void processEvent(TestEvent event){

    }
```
上面代码是指定在UI线程中处理消息。然后看一下注解是在org.greenrobot.eventbus.SubscriberMethodFinder中的findUsingReflectionInSingleClass()方法处理的。
```java
	private void findUsingReflectionInSingleClass(FindState findState) {
        Method[] methods;
        //获取所有方法
        try {
            // This is faster than getMethods, especially when subscribers are fat classes like Activities
            methods = findState.clazz.getDeclaredMethods();
        } catch (Throwable th) {
            // Workaround for java.lang.NoClassDefFoundError, see https://github.com/greenrobot/EventBus/issues/149
            methods = findState.clazz.getMethods();
            findState.skipSuperClasses = true;
        }
        for (Method method : methods) {
            int modifiers = method.getModifiers();
            //判断方法修饰的类型
            if ((modifiers & Modifier.PUBLIC) != 0 && (modifiers & MODIFIERS_IGNORE) == 0) {
                Class<?>[] parameterTypes = method.getParameterTypes();
                //限制参数数量
                if (parameterTypes.length == 1) {
                //取得注释
                    Subscribe subscribeAnnotation = method.getAnnotation(Subscribe.class);
                    if (subscribeAnnotation != null) {
                        Class<?> eventType = parameterTypes[0];
                        if (findState.checkAdd(method, eventType)) {
                        //获取线程类型
                            ThreadMode threadMode = subscribeAnnotation.threadMode();
                            findState.subscriberMethods.add(new SubscriberMethod(method, eventType, threadMode,
                                    subscribeAnnotation.priority(), subscribeAnnotation.sticky()));
                        }
                    }
                } else if (strictMethodVerification && method.isAnnotationPresent(Subscribe.class)) {
                    String methodName = method.getDeclaringClass().getName() + "." + method.getName();
                    throw new EventBusException("@Subscribe method " + methodName +
                            "must have exactly 1 parameter but has " + parameterTypes.length);
                }
            } else if (strictMethodVerification && method.isAnnotationPresent(Subscribe.class)) {
                String methodName = method.getDeclaringClass().getName() + "." + method.getName();
                throw new EventBusException(methodName +
                        " is a illegal @Subscribe method: must be public, non-static, and non-abstract");
            }
        }
	}
```
上面就通过反射获取到了注册类中的注解方法，然后获取注解值，在构造SubscriberMethod的时候就传入了threadMode，然后在Post的时候，会根据不同的类型去进行不同的处理：
```java
	private void postToSubscription(Subscription subscription, Object event, boolean isMainThread) {
        switch (subscription.subscriberMethod.threadMode) {
            case POSTING:
                invokeSubscriber(subscription, event);
                break;
            case MAIN:
                if (isMainThread) {
                    invokeSubscriber(subscription, event);
                } else {
                    mainThreadPoster.enqueue(subscription, event);
                }
                break;
            case BACKGROUND:
                if (isMainThread) {
                    backgroundPoster.enqueue(subscription, event);
                } else {
                    invokeSubscriber(subscription, event);
                }
                break;
            case ASYNC:
                asyncPoster.enqueue(subscription, event);
                break;
            default:
                throw new IllegalStateException("Unknown thread mode: " + subscription.subscriberMethod.threadMode);
        }
    }
```
根据不同的类型会有不同的Poster去在对应线程操作
```java
    private final HandlerPoster mainThreadPoster;
    private final BackgroundPoster backgroundPoster;
    private final AsyncPoster asyncPoster;
```

以上就是运行期注解在EventBus中的一些应用以及简单的理解。

我们自己操作的时候，也可以通过同样的方式，先声明一个运行期注解，然后在需要操作的地方反射的获取到被注解的元素，进行需要的操作。

#### 总结
至此注解的文章就写完了，可以看到原理其实比较简单。而我们学习这个的目的，一方面是更好的去理解各种流行框架的原理以及设计思路，在面对一些重复的代码工作中可以变得更加高效，也是给自己的思维一个拓展，在面对项目的设计问题时，有一个武器可以使用，也可以更好的去理解各种解耦的思想。