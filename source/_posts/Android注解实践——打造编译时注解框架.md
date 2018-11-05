---
title: Android注解实践——打造编译时注解框架
date: 2017-04-8 15:33:40
categories: Android文章
tags: Android, Annotation, 注解
---
最近一直在做项目的重构工作，因为是做组件化，正好看到阿里云开源一个路由框架[ARouter](https://github.com/alibaba/ARouter)。看了一下源码，发现是项目主要也是运用到了编译时注解的技术。联想到之前最早用过的[ButterKnife](https://github.com/JakeWharton/butterknife)，到现在的[Retrofit](https://github.com/square/retrofit)等等各种主流的开源框架，其实都有使用到这项强大技术，之前也只是简单了解了一下原理，这次就想自己也写一个这种框架，搞清楚实现原理。
首先，注解处理器(Anonotation Processor)分为编译时(Compile time)注解和运行时(Runtime)通过反射机制运行的注解，因为编译期注解实际上是生成.java文件辅助我们实现功能，所以不会有效率上的损耗，上面提到的开源框架也都是基于这种基础实现的。
这篇文章只涉及编译时注解(类似于[ButterKnife](https://github.com/JakeWharton/butterknife))，教大家如何打造一个简单的编译时注解框架，如何调试，和一些在实践中碰到的问题。
<!-- more -->
#### 基本概念
注解处理器是Javac的一个工具，它用来在编译时扫描和处理注解，更多信息可以查看[官方文档](http://docs.oracle.com/javase/tutorial/java/annotations/index.html)。
一个注解的注解处理器，以Java代码作为输出，生成文件（通常是.java文件）作为输出。这意味着你可以生成java代码，当然生成的.java代码是在新的文件中，你不能修改原有的Java类。但是我们可以生成辅助类，来帮助我们完成工作，就像[ButterKnife](https://github.com/JakeWharton/butterknife)这样，我们省去了findviewById的重复工作，仅需一行注解，他就能帮我们完成操作。我们掌握这项技术，也可以在之后的工作中减少重复无意义的工作，更重要的是注解能够帮我们更好的解耦我们的各个模块。
##### 注解类型
首先举一个我们最常见的注解：
```java
@Override
```
大家都知道这是重写的意思，我们具体看看它的代码。
```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.SOURCE)
public @interface Override {
}
```
出现了两个东西，@Target、 @Retention，前者代表该注解可以作用于什么地方，后者代表要在什么级别保存该注解信息。我们看看枚举里支持的类型。
```java
public enum ElementType {
    /** Class, interface (including annotation type), or enum declaration */
    TYPE,

    /** Field declaration (includes enum constants) */
    FIELD,

    /** Method declaration */
    METHOD,

    /** Formal parameter declaration */
    PARAMETER,

    /** Constructor declaration */
    CONSTRUCTOR,

    /** Local variable declaration */
    LOCAL_VARIABLE,

    /** Annotation type declaration */
    ANNOTATION_TYPE,

    /** Package declaration */
    PACKAGE,

    /**
     * Type parameter declaration
     *
     * @since 1.8
     */
    TYPE_PARAMETER,

    /**
     * Use of a type
     *
     * @since 1.8
     */
    TYPE_USE
}
```

```java
public enum RetentionPolicy {
    /**
     * Annotations are to be discarded by the compiler.
     */
    SOURCE,

    /**
     * Annotations are to be recorded in the class file by the compiler
     * but need not be retained by the VM at run time.  This is the default
     * behavior.
     */
    CLASS,

    /**
     * Annotations are to be recorded in the class file by the compiler and
     * retained by the VM at run time, so they may be read reflectively.
     *
     * @see java.lang.reflect.AnnotatedElement
     */
    RUNTIME
}
```
上面贴出了枚举值，里面的注释也写的很清楚，我简单说一下RetentionPolicy，SOURCE是会被编译器丢弃的注解，CLASS是在编译器保存，RUNTIME是在运行期保留，可以通过反射获取到的。所以我们编译期注解框架的Retention是CLASS类型。
##### AbstractProcessor
接下来我们要了解一个注解的核心，[AbstractProcessor类](https://docs.oracle.com/javase/7/docs/api/javax/annotation/processing/AbstractProcessor.html)，它是一个抽象的注释处理器，设计来为大多数注解实践类提供方便的超类。我们所有的Processor API都是继承自AbstractProcessor类。
我们继承AbstractProcessor类之后，需要实现四个方法：
```java
public class MyProcessor extends AbstractProcessor {

    @Override
    public synchronized void init(ProcessingEnvironment env){ }

    @Override
    public boolean process(Set<? extends TypeElement> annoations, RoundEnvironment env) { }

    @Override
    public Set<String> getSupportedAnnotationTypes() { }

    @Override
    public SourceVersion getSupportedSourceVersion() { }

}
```
* **init(ProcessingEnvironment env)**：javac 会在 Processor 创建时调用并执行的初始化操作，该方法会传入 一个参数 ProcessingEnvironment env ，通过 env 可以访问 Elements、Types、Filer等工具类。
* **getSupportedAnnotationTypes()**：返回需要注册的注解集合。
* **getSupportedSourceVersion()**：返回支持的java版本，通常返回SourceVersion.latestSupported()。
* **process(Set<? extends TypeElement> annoations, RoundEnvironment env)**：这个方法相当于Processor类的main方法，所有扫描和处理注解、生成.java文件的操作，都是在这里完成。

#### 实践步骤
了解了上面的基础知识，这里讲讲具体如何实现。我们的目的是通过注解实现替代我们findViewById的繁琐操作，类似于ButterKnife的@Bind操作。
```java
   @BindView(R.id.test_text)
   TextView textView;
```
整个项目分为四个模块，app，annotation（注解），api，compiler（注解处理器）。
首先讲一下整体的思路，在我们的Activity中，使用注解定义控件，在onCreate方法中，调用Api模块的bind(this)。bind方法内其实就是通过反射获取到注解处理器生成的辅助类，通过辅助类完成控件的初始化工作。

##### annotation
在项目中，New Module，选择Java Library，新建类，定义注解
```java
@Retention(RetentionPolicy.CLASS)
@Target(ElementType.FIELD)
public @interface BindView {
	int value();
}
```
@BindView对成员变量进行注解，接收一个int类型的参数。

##### api
New Module，选择Android Library，
首先我们需要定一个一个接口，生成的注解类需要实现这个接口，然后我们通过这个接口去完成注入的操作从而达到目的。
```java
public interface ViewBind<T> {
	void inject(T t, Object obj);
}
```
还需要一个类，提供给需要使用注解的activity，完成绑定操作，通过bind(this)方法，获取到注解类，调用inject方法注入。
```java
public class ViewBinder {

    private final static String SUFFIX = "$$ViewBinder";

    public static void bind(Activity activity){
        ViewBind proxyActivity = findProxyActivity(activity);
        proxyActivity.inject(activity, activity);
    }

    public static void injectView(Object object, View view)
    {
        ViewBind proxyActivity = findProxyActivity(object);
        proxyActivity.inject(object, view);
    }

    private static ViewBind findProxyActivity(Object activity){
        try {
            Class clazz = activity.getClass();
            Class viewBindClazz = Class.forName(clazz.getName() + SUFFIX);
            return (ViewBind) viewBindClazz.newInstance();
        } catch (ClassNotFoundException | IllegalAccessException | InstantiationException e) {
            e.printStackTrace();
        }
        throw new RuntimeException(String.format("can not find %s, something error when compiler.", activity.getClass().getSimpleName() + SUFFIX));
    }

}
```
##### compiler
注解处理器，这是核心的模块。New Module，选择Java Library。
在gradle中添加：
```gradle
    compile 'com.google.auto.service:auto-service:1.0-rc2'
    compile 'com.squareup:javapoet:1.7.0'
```
前者是自动生成 META-INF/services/javax.annotation.processing.Processor文件的库，后者[JavaPoet](https://github.com/square/javapoet)是一个生成java代码的库，免去了我们拼字符串的繁琐。
* **Processor类**
```java
/**
 * 使用 Google 的 auto-service 库可以自动生成 META-INF/services/javax.annotation.processing.Processor 文件
 */
@AutoService(Processor.class)
public class BindProcessor extends AbstractProcessor{

    //元素处理辅助类
    private Elements elementUtils;

    //日志辅助类
    private Messager messager;

    private Map<String, BindProxy> mProxyMap = new HashMap<String, BindProxy>();;

    @Override
    public synchronized void init(ProcessingEnvironment processingEnv) {
        super.init(processingEnv);
        elementUtils = processingEnv.getElementUtils();
        messager = processingEnv.getMessager();
    }

    /**
     * @return 指定哪些注解应该被注解处理器注册
     */
    @Override
    public Set<String> getSupportedAnnotationTypes() {
        HashSet<String> supportType = new HashSet<String>();
        supportType.add(BindView.class.getCanonicalName());
        return supportType;
    }

    /**
     * @return 指定使用的 Java 版本。通常返回 SourceVersion.latestSupported()。
     */
    @Override
    public SourceVersion getSupportedSourceVersion()
    {
        return SourceVersion.latestSupported();
    }

    @Override
    public boolean process(Set<? extends TypeElement> annotations, RoundEnvironment roundEnv) {
        messager.printMessage(Diagnostic.Kind.NOTE, "process...");
        //获取BindView注释的元素集合
        Set<? extends Element> elements = roundEnv.getElementsAnnotatedWith(BindView.class);
        if (elements == null || elements.size() < 1){
            return true;
        }
        //遍历集合
        for (Element element : elements){
            //检查是否是作用于FIELD
            if (checkElement(element)){
                VariableElement variable = (VariableElement) element;
                TypeElement typeElement = (TypeElement) element.getEnclosingElement();
                String className = typeElement.getQualifiedName().toString();
                //从缓存中取得BindProxy类,不存在则new
                BindProxy proxy = mProxyMap.get(className);
                if (proxy == null){
                    proxy = new BindProxy(elementUtils, typeElement);
                    mProxyMap.put(className, proxy);
                }
                BindView bindView = variable.getAnnotation(BindView.class);
                proxy.injectInfo.put(bindView.value(), variable);
            } else {
                messager.printMessage(Diagnostic.Kind.ERROR, "error...");
            }
        }
        //遍历mProxyMap 取出所有的BindProxy类 去生成代码
        for (String key : mProxyMap.keySet()){
            BindProxy proxy = mProxyMap.get(key);
            try {
                proxy.generateCode().writeTo(processingEnv.getFiler());
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
        return true;
    }

    private boolean checkElement(Element element){
        if (element.getKind() != ElementKind.FIELD)
        {
            messager.printMessage(Diagnostic.Kind.ERROR, "%s must be declared on field.", element);
            return false;
        }
        return true;
    }

}
```
* **生成代码类**
```java

public class BindProxy {

	public Map<Integer, VariableElement> injectInfo = new HashMap<>();

	private TypeElement element;

	private String packageName, className;

	private static final String PROXY = "$$ViewBinder";

	public BindProxy(Elements elements, TypeElement typeElement) {
		element = typeElement;
		PackageElement packageElement = elements.getPackageOf(typeElement);
		packageName = packageElement.getQualifiedName().toString();
		className = typeElement.getSimpleName() + PROXY;
	}

	public JavaFile generateCode() {

		//生成方法代码
		MethodSpec.Builder injectMethodBuilder = MethodSpec.methodBuilder("inject")
				.addModifiers(Modifier.PUBLIC)
				.addAnnotation(Override.class)
				.addParameter(TypeName.get(element.asType()), "host", Modifier.FINAL)
				.addParameter(TypeName.OBJECT, "obj");

		//在方法中插入一行findViewById代码,遍历所有的元素
		for (int id : injectInfo.keySet()) {
			VariableElement element = injectInfo.get(id);
			String name = element.getSimpleName().toString();
			TypeMirror type = element.asType();
			injectMethodBuilder.addStatement("host.$N = ($T)((($T) obj).findViewById($L))"
					, name, type, TypeUtil.ANDROID_ACTIVITY, id);
		}

		//生成class代码
		TypeSpec clazz = TypeSpec.classBuilder(className)
				//这里添加的接口类,并添加了泛型
				.addSuperinterface(ParameterizedTypeName.get(TypeUtil.VIEWBIND, TypeName.get(element.asType())))
				.addModifiers(Modifier.PUBLIC)
				.addMethod(injectMethodBuilder.build())
				.build();

		return JavaFile.builder(packageName, clazz).build();
	}

}
```
##### 配置
主要的代码就在上面，注释写的应该比较清楚了，剩下的就是对项目进行配置。
我们需要在根项目的build.gradle中添加dependencies
```gradle
	classpath 'com.neenbedankt.gradle.plugins:android-	apt:1.8'
```
在app的build.gradle中添加
```gradle
	apply plugin: 'com.neenbedankt.android-apt'
```
然后添加dependencies
```gradle
	compile project(':api')
    compile project(':annotation')
    apt project(':compiler')
```
##### 效果
然后运行或者build项目就可以看到注解生成的类，目录是build/generated/source/apt/debug/包名
```java
public class MainActivity$$ViewBinder implements ViewBind<MainActivity> {
  @Override
  public void inject(final MainActivity host, Object obj) {
    host.textView = (TextView)(((Activity) obj).findViewById(2131492942));
  }
}
```

#### 常见问题
##### 编译报错
在我开始尝试这个项目的时候，参考了很多别人的文章，都是正常的编码思路和源码，没有讲容易碰到的问题。我在参照别人写完demo，build的时候一直报错。
```
Error:Execution failed for task ':app:compileDebugJavaWithJavac'.
> java.lang.NullPointerException
```
这个错之前也碰到过，但是具体原因忘了，起初以为是gradle配置的问题，改来改去还是解决不了，搜索也解决不了。
后来注释掉了processor中的部分代码，发现build成功。原来是Processor中异常了。
所以报这种错的时候，检查一下自己的代码，肯定是有异常。

##### 调试Processor
当Processor出现问题的时候，最直观寻找问题的方法就是debug。
Debug Processor步骤
* 在Android studio中添加Remote Debugger，并确定port的设置是跟下面的参数一致
* 在 gradle.properties文件中添加如下
```
org.gradle.parallel=true
org.gradle.jvmargs=-agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=5005
```
然后添加断点，在你需要调试的地方，项目build的时候就可以调试了。

##### 为什么要分开注解和处理器
一方面是更好的解耦，我们的注解处理器可以用于其他项目。还有一个就是能避免65K方法数问题。

#### 总结
这个项目就是一个简单的APT demo，主要运用的就是注解的基础知道以及AbstractProcessor类和JavaPoet库生成Java代码，通过这个我们可以学习如何编写APT项目，实现起来并不复杂，但是注解处理器是一个非常强大的工具。整个的重心在于，通过这个我们能够知道有这么一种方式可以再编译期生成代码，简化我们的工作。更重要的是要有这么一个思路，可以去设计我们的架构，对架构实现更好的解耦，ARouter就是通过APT实现依赖反转，我也是抱着这些目的来学习注解。之后我会写运行时的注解处理器。

#### 项目地址

https://github.com/PengsongAndroid/MyAnnotation

**参考：**
[Java注解处理器](http://www.race604.com/annotation-processing/)
[How to debug the apt AbstractProcessor code generation?](http://stackoverflow.com/questions/30959145/how-to-debug-the-apt-abstractprocessor-code-generation)
[Android 如何编写基于编译时注解的项目](http://blog.csdn.net/lmj623565791/article/details/51931859)
