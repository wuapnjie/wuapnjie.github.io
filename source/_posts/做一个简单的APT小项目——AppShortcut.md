---
title: 做一个简单的APT小项目——AppShortcut
date: 2016-11-30
tags: [Android]
---

最近学习了编译时注解框架的制作，写了一个小项目。阅读本文前希望大家有关于注解的相关知识。

本文介绍一个简单的编译时注解小项目的制作过程。我选择了Android API 25的新功能App Shortcut，使用注解来快速制作一个Shortcut。为什么选择Shortcut呢，因为我觉得很多应用只需要使用到静态加载的Shortcut就好了，而对于静态加载的Shortcut要写一个比较长的Xml配置文件，我觉得特别麻烦。

先来看一下我们实现的效果。

Java代码

```java
@ShortcutApplication
@AppShortcut(resId = R.mipmap.ic_launcher,
        description = "First Shortcut")
public class MainActivity extends AppCompatActivity {
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
      	//API 调用
        ShortcutCreator.create(this);
    }
}
```

效果图：

![](http://7xrqmj.com1.z0.glb.clouddn.com/shortcut.gif)





### APT简单介绍

APT，全称Annotation Processing Tool，它用来在编译时处理源代码中的注解信息，我们可以根据注解来生成一些Java文件，防止编写冗余的代码，比如ButterKnife项目，正是利用了APT工具，帮助我们少写了许多重复冗余的代码。本文中，通过注解来少写一些配置文件。

### 项目结构

本项目共分为4个Module，两个Java Library module，一个Android Library module和用于演示的Android Application module

* **easyshortcuts-api**:Android Library module，用于供客户端的调用。
* **easyshortcuts-annotation**:Java Library module，用于提供注解类。
* **easyshortcuts-compiler**:Java Library module，用于编写处理注解并生成相关`Processor`的注解处理模块
* 还有一个普通的应用模块

其中**easyshortcuts-api**和**easyshortcuts-compiler**模块依赖**easyshortcuts-annotation**模块。



### 注解模块的编写

搭好项目后，第一个动手编写的应该是**easyshortcuts-annotation**模块，通过查看Android Developer官网上的Shortcut介绍后，发现通过Java代码，我们只可以通过`ShortcutManager`生成动态Shortcut，生成一个动态Shortcut的代码简单重复，每个Shortcut需要一个String类型的Id，图标的ResId，显示的文字，以及一个Intent的Action字段。

```Java
@Retention(RetentionPolicy.SOURCE)
@Target(ElementType.TYPE)
public @interface AppShortcut {
    int resId();

    int rank() default 1;

    String description();

    String action() default Define.SHORTCUT_ACTION;
}
```

之后我用注解所在类的类名称作为Shortcut的Id。

这里，我还建了一个注解`ShortcutApplication`，是一个没有任何字段的注解，这个注解应该用在用户第一个打开的Activity。因为这里使用了动态加载的方式创建Shortcut，所以必须要执行代码才可以生成Shortcut，所以应该在Launcher Activity使用。

### 注解处理器的编写

确定了注解后，我们要编写相应的注解处理器来处理注解并生成相应的Java文件，这里**easyshortcuts-compiler**依赖了google的[auto-service](https://github.com/google/auto/tree/master/service)库和square的[javapoet](https://github.com/square/javapoet)库。

```groovy
compile "com.squareup:javapoet:$rootProject.ext.squareJavaPoetVersion"
compile "com.google.auto.service:auto-service:$rootProject.ext.googleAutoServiceVersion"
```

其中auto-service库可以很方便的帮助我们生成配置文件，javapoet库可以很方便的帮助我们自动生成Java代码，以下代码通过查看Javapoet的README就可以很快理解。

新建一个继承自`AbstractProcessor`的`ShortcutProcessor`，下面是一个基本的`Processor`应有的要素，我们的重点在与`process()`方法。

```java
//帮助我们生成配置文件的注解
@AutoService(Processor.class)
public class ShortcutsProcessor extends AbstractProcessor {
    private Filer mFiler;
    private Elements mElementUtils;
    private Messager mMessager;
	……
      
    //在初始化时获得相关帮助对象
    @Override
    public synchronized void init(ProcessingEnvironment processingEnvironment) {
        super.init(processingEnvironment);
        mFiler = processingEnvironment.getFiler();
        mElementUtils = processingEnvironment.getElementUtils();
        mMessager = processingEnvironment.getMessager();
    }

  	//根据相应的注解进行处理
    @Override
    public boolean process(Set<? extends TypeElement> set, RoundEnvironment roundEnvironment) {
       	……
        return true;
    }

  	//返回要支持的注解
    @Override
    public Set<String> getSupportedAnnotationTypes() {
        Set<String> types = new LinkedHashSet<>();
        types.add(ShortcutApplication.class.getCanonicalName());
        types.add(AppShortcut.class.getCanonicalName());
        return types;
    }

  	//返回Java语言的支持版本
    @Override
    public SourceVersion getSupportedSourceVersion() {
        return SourceVersion.latestSupported();
    }
	
    //辅助的日志打印方法
    private void printNote(String message) {
        mMessager.printMessage(Diagnostic.Kind.NOTE, message);
    }

    private void printError(String error) {
        mMessager.printMessage(Diagnostic.Kind.NOTE, error);
    }
}

```

在`process()`方法中，要获取所有有相关注解的`Element`，并获取每个注解中附带的字段，最后根据这些信息生成一个Java文件。为了更好的获取储存这些字段，我建立了一个model类`Shortcut`。

```java
public class Shortcut {
    private int mResId;
    private int mRank;
    private String mDescription;
    private String mAction;
    private TypeElement mTypeElement;

    public Shortcut(Element element) {
        mTypeElement = (TypeElement) element;
        AppShortcut appShortcut = mTypeElement.getAnnotation(AppShortcut.class);
        mResId = appShortcut.resId();
        mRank = appShortcut.rank();
        mDescription = appShortcut.description();
        mAction = appShortcut.action();
    }
  
  	//相关的getXXX()方法
  	……
      
}
```

之后在`process()`方法中遍历所有带有相关注解的`Element`，并生成model对象

```java
for (Element element : roundEnvironment.getElementsAnnotatedWith(AppShortcut.class)) {
     //检查注解所标注的元素是否为我们需要
     if (!isValid(element)) {
         return false;
     }
	 //解析这个element并生成相应的Shortcut对象
     parseShortcut(element);
}
```

最后根据所有`Shortcut`对象生成Java文件

```java
mShortcutClass.generateCode().writeTo(mFiler);
```

生成代码我使用Javapoet，可以很方便的生成代码。

```java
public JavaFile generateCode() {
        MethodSpec.Builder methodBuilder = MethodSpec.methodBuilder("create")
                .addModifiers(Modifier.PUBLIC)
                .addAnnotation(Override.class)
                .addParameter(CONTEXT, "context")
                .addStatement("$T shortcutManager = context.getSystemService($T.class)", SHORTCUT_MANAGER, SHORTCUT_MANAGER)
                .addStatement("$T.Builder builder",SHORTCUT_INFO)
                .addStatement("$T intent",INTENT);

        for (Shortcut shortcut : mShortcuts) {
            methodBuilder.
                    addStatement("builder = new $T.Builder(context,$S)", SHORTCUT_INFO, shortcut.getTypeElement().getSimpleName().toString())
                    .addStatement("intent = new $T(context, $T.class)", INTENT, TypeName.get(shortcut.getTypeElement().asType()))
                    .addStatement("intent.setAction($S)", shortcut.getAction())
                    .addStatement("builder.setIntent(intent)")
                    .addStatement("builder.setShortLabel($S)", shortcut.getDescription())
                    .addStatement("builder.setLongLabel($S)", shortcut.getDescription())
                    .addStatement("builder.setRank($L)", shortcut.getRank())
                    .addStatement("builder.setIcon($T.createWithResource(context, $L))", ICON, shortcut.getResId())
                    .addStatement("shortcutManager.addDynamicShortcuts(singletonList(builder.build()))");
        }

        TypeSpec shortcutClass = TypeSpec.classBuilder(mTypeElement.getSimpleName() + SUFFIX)
                .addModifiers(Modifier.PUBLIC)
                .addSuperinterface(CREATOR)
                .addMethod(methodBuilder.build())
                .build();

        String packageName = mElementUtils.getPackageOf(mTypeElement).getQualifiedName().toString();

        return JavaFile
                .builder(packageName, shortcutClass)
                .addStaticImport(Collections.class, "singletonList")
                .build();
}
```

### 提供调用接口

写完了注解的解释器后，我们每次编译都生成了一个Java类文件，但是我们并没有调用它，我们要提供一个接口来调用，本项目中提供了这样一个静态方法

```java
ShortcutCreator.create(this);
```

```java
public class ShortcutCreator {
    public static void create(Context context) {
        try {
            Class<?> targetClass = context.getClass();
            Class<?> creatorClass = Class.forName(targetClass.getName() + "$$Shortcut");
            Creator creator = (Creator) creatorClass.newInstance();
            creator.create(context);
        } catch (ClassNotFoundException | InstantiationException | IllegalAccessException e) {
            e.printStackTrace();
        }
    }
}
```

由于我们利用APT自动生成的Java类的类名称是知道的且提供了默认的无参数构造器，所以我们很容易生成一个对象，并调用其相关方法来生成相应的Shortcut。

### 总结

编译时注解可以大大加快我们的开发效率，希望大家可以多制作一些编译时注解的库来造福广大开发者，让大家少些许多简单重复的代码。最后附上源码地址：[https://github.com/wuapnjie/EasyShortcuts](https://github.com/wuapnjie/EasyShortcuts)



