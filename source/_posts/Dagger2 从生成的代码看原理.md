---
title: Dagger2 从生成的代码看原理
date: 2017-04-24 07:33:44
categories: Android日常问题
tags: Android, 开源库, Dagger
---
接着之前的注解，正好最近有时间，对去年大热的开源库[Dagger](https://github.com/google/dagger)做一个分析。
首先举一个最基础的MVP模式的例子，通过使用注解

<!-- more -->

首先我们从调用的入口看
```java
DaggerMainComponent.builder().mainMoudle(new MainMoudle(this)).build().inject(this);
```
点进去我们可以看到DaggerMainComponent是实现的MainComponent接口，MainComponent就是我们添加了@Component注解的接口。所以我们可以推断一下，添加了@Component注解的接口，dagger会帮我们生成一个实现类。

然后看看builder方法，这个方法是直接返回了一个new的Builder对象。
```java
  public static Builder builder() {
    return new Builder();
  }
```
然后看看Builder对象是干嘛的。

```java
  public static final class Builder {
    private MainMoudle mainMoudle;

    private Builder() {}

    public MainComponent build() {
      if (mainMoudle == null) {
        throw new IllegalStateException(MainMoudle.class.getCanonicalName() + " must be set");
      }
      return new DaggerMainComponent(this);
    }

    public Builder mainMoudle(MainMoudle mainMoudle) {
      this.mainMoudle = Preconditions.checkNotNull(mainMoudle);
      return this;
    }
  }
```
可以看到，这个Builder类中有一个MainMoudle对象，还有一个mainMoudle提供给我们来注入对象，然后就是一个build方法返回一个DaggerMainComponent对象。
接上面的说，添加了@Component注解的接口，dagger在生成Builder类时，会生成一个module对象，这个对象就是我们在注解中添加的参数，然后会生成一个方法供我们注入对象。
从我们调用入口来看，先调用builder方法创建一个Builder对象，然后注入一个MainModule对象，再调用build方法，创建DaggerMainComponent对象。
我们看看build方法，实际是调用了DaggerMainComponent的构造方法，调用的initialize方法：
```java
  private void initialize(final Builder builder) {

    this.provideViewProvider = MainMoudle_ProvideViewFactory.create(builder.mainMoudle);

    this.mainPresenterProvider = MainPresenter_Factory.create(provideViewProvider);

    this.mainActivityMembersInjector = MainActivity_MembersInjector.create(mainPresenterProvider);
  }
```
就三行代码，在分析这三行代码之前我们先想一想，实例化Activity中的presenter的步骤，首先我们需要：获取view→提供view给presenter→在Activity中注入presenter对象。
有了这个思路，我们再来看看这三行代码。MainMoudle_ProvideViewFactory实现了Factory接口，Factory集成的Provider接口，Provider中有一个get方法（这点不展开说，暂时还不懂）。
MainMoudle_ProvideViewFactory类主要就是提供了一个get方法，get方法是通过module的provideView方法来提供view对象。provideView就是我们之前在MainMoudle类中添加了@Provides注解的方法。
然后我们返回的Factory对象会传递给MainPresenter_Factory，MainPresenter_Factory也只是保留这个对象的引用，然后在需要get方法的时候通过这个对象提供的view去实例化MainPresenter。
然后MainPresenter_Factory会传递给MainActivity_MembersInjector的create方法，MainActivity_MembersInjector类实现了MembersInjector接口，MembersInjector接口有一个injectMembers方法。
```java
  @Override
  public void injectMembers(MainActivity instance) {
    if (instance == null) {
      throw new NullPointerException("Cannot inject members into a null reference");
    }
    instance.presenter = presenterProvider.get();
  }
```
现在我们回头再看看我们在Activity中调用的inject方法，这个方法实际上调用的就是上面说的injectMembers方法，传入了MainActivity实例，然后直接对presenter进行赋值。而这个presenter就是我们在Activity中添加的注解。
```java
    @Inject
    MainPresenter presenter;
```
至此一个完整的注入流程就完了，写的比较凌乱，现在梳理一下整个思路。
首先，我们的目的是@Inject注解对MainActivity中的presenter进行注入，我们的入口是从@Component注解开始，通过这个入口去生成Component类，@Component注解中又提供了参数module，Module类中又有一个@Provides注解这个方法提供了一个View对象。在Presenter中有一个@Inject注解的方法，Moudle中provide的view对象就被注入到presenter中，presenter实例化后又被注入到Activity，这就完成了这个demo的整个注解流程。
我们的分析是顺着生成的代码，一步步查看生成代码来分析流程的，这样能弄懂对象是如何注解的，但是没法了解dagger框架的原理。
我们现在从刚刚分析的流程中进行倒推，我们做的事情只是在几个需要的地方进行注解。通过这几个注解如何生成这些代码？dagger中肯定有一个存放所有注解、分析所有注解的一个关系图。
