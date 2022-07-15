Drakeet fork:

```
maven { url 'https://jitpack.io' }

classpath 'com.github.drakeet:sdk-editor-plugin:1.1.4.1'
```

----

## 简介
sdk-editor是为实现修改APP依赖的第三方SDK而开发的Gradle插件，插件利用Android Plugin官方提供的Transform API干预APK Build流程，实现对三方SDK中特定类的替换修改，不影响APP运行性能，也不会增加APK体积。
## 适用场景
- 修复SDK中存在的Bug；

- 暴露出SDK某些未提供的接口；

- 扩展SDK功能；

- 其他需要修改SDK已有类的需求；
## 基本用法
#### 1. 在根项目（最外层）的build.gradle文件中添加插件依赖：
```gradle
buildscript {
    dependencies {
        classpath 'com.iwhys.sdkeditor:plugin:1.1.4'
    }
}
```
#### 2. 在项目主模块（app module）的build.gradle文件应用插件：
```gradle
apply plugin: 'sdk-editor'
```
#### 3. 找到三方SDK中需要修改的类文件（以下称为Bug类），在app module中新建与Bug类同包名同类名的新类（以下称为Fix类），同时拷贝Bug类的内容到Fix类，给Fix类添加类注解@ReplaceClass，在注解的值中标记该类所在SDK的名字，最后在Fix类中实现要修改的内容即可。

下面以demo module中替换support v4包中的BuildCompat类为例进行说明，我们需要BuildCompat类中的isAtLeastQ方法，在其中添加一条Toast语句，修改流程如下：

1）在demo工程的main/java下新建android/support/v4/os/BuildCompat类；

2）拷贝原SDK中BuildCompat类的内容，并修改新建的BuildCompat类；
```java
@ReplaceClass("com.android.support:support-compat:28.0.0")
public class BuildCompat {
    ...
    public static boolean isAtLeastQ() {
        // 注意：这里增加了我们自定义的代码
        Toast.makeText(MyAppKt.getAppContext(), "you have invoked the method: BuildCompat#isAtLeastQ()", Toast.LENGTH_LONG).show();
        return VERSION.CODENAME.length() == 1 && VERSION.CODENAME.charAt(0) >= 'Q' && VERSION.CODENAME.charAt(0) <= 'Z';
    }
}
```
3）编译并运行demo，点击按钮弹出一个Toast，即表明support v4包中的BuildCompat类被成功的修改；
## 高级用法
如果有多个项目用到了同一个需要修改的SDK，为了在多个项目中共享修复后的代码，我们可以把修复代码封装成一个单独jar/aar包。下面以demo模块中libs引用的三方SDK DuappsAd-HW-v1.1.1.6-release.aar为例，我们需要修改SDK中的com.duapps.ad.DuNativeAd类，在其中添加广告请求监听器，具体流程如下：

1）在任意项目中新建一个module（如工程中的library_fix）；

2）在module的build.gradle文件中添加对需要修复的SDK（如：DuappsAd-HW-v1.1.1.6-release.aar）以及插件注解包（com.iwhys.classeditor:domain:1.1.0）的依赖；

3）在module的main/java目录下新建包com/duapps/ad，并在com/duapps/ad中新建DuNativeAd类，同时拷贝原SDK中DuNativeAd类的内容

4）在新建的DuNativeAd类中添加注解@ReplaceClass("DuappsAd-HW-v1.1.1.6-release")；

5）在新建的DuNativeAd类中添加需要新增的广告监听器逻辑；

6）在终端(Terminal)中执行命令：gradlew library_fix:build，即可在该module的build/output目录中看到生成的aar文件；

7）重命名aar文件(或者其中的jar文件)为：du_hack，拷贝该文件到主module(app module)的libs文件夹，并添加对该文件的依赖；

8）在主module(app module)的build.gradle中指定用于SDK修复的jar/aar包信息即可，格式如下：
```gradle
sdkEditor {
    // 这里是一个数组，有多个用于修复的包，需要用","分隔开
    extraJarNames = ['hack_du']
}
```
9）此时主module（app module）便可以在编译的时候使用jar/aar(hack_du)中的Fix类来替换原SDK中的Bug类；

10）插件默认使用单线程处理任务，可通过配置parallel开启多线程并发来提高处理速度；
```gradle
sdkEditor {
    // 默认值为false，即单线程处理任务
    parallel = true
}
```
## 常见问题
#### 1. 新建与Bug类同包名的Fix类时，编译器提示"Package 'xxx.xxx.xxx' clashes with class of same name"
这种情况是因为包路径中的包和类重名了，我们可以通过把java类转换为kotlin类来修复这个问题。

注意：此时需要添加kotlin相关的支持，且因为kotlin编译器在把类编译为class的时候，默认会把文件名改为：原文件名+kt，因此在kotlin版的Fix类中添加文件命名注解 @file:JvmName("Bug类的名称")
#### 2. 在新建的Fix类中，存在部分形如"a.b.c"的类无法正确的导入，或者导入之后与当前类的成员重名
这种情况我们可以把类改为kotlin版，并利用kotlin提供的 import xxx as yyy功能，对导入的有问题的类进行重命名。个人感觉通过导入重命名方式能够解决99%的这种问题，剩下的1%可以通过反射来实现。
#### 3. 新建的Fix类时，如果其所在包的名字同级已经存在一个同名的类（如已存在类com.a.a，Fix类路径com.a.a.Fix,则IDE提示"Redeclare a")
我们可以通过"高级用法"的笨办法，新建module把已存在类com.a.a和Fix类com.a.a.Fix分别放在不同的module来实现。
#### 4. Fix过程正常，但是APK运行到Fix类发生Crash，提示Fix类中缺少xxx方法
通常我们会使用IDE来浏览依赖的SDK文件，并在IDE中把Bug类的源码拷贝到Fix类中，但有些情况下IDE反编译的class代码并不完整，建议使用jeb反编译SDK中的Bug类。
## 项目简析
[sdk-editor结构及原理简析](项目简析.md)
## 特别感谢
[javassist](https://github.com/jboss-javassist/javassist)
## 协议
[The Apache Software License, Version 2.0](http://www.apache.org/licenses/LICENSE-2.0.txt)
