## 准备工作

您首先需要一个 Android 工程，这个工程可以是您现有的工程，也可以是您新建的一个空的工程。

## 第一步：创建项目和应用（已完成请跳过）

在使用我们的服务前，您必须先在 MobileLine 控制台上 [创建项目和应用](https://cloud.tencent.com/document/product/666/15345)。

## 第二步：添加配置文件（已完成请跳过）

在您创建好的应用上点击【下载配置】按钮来下载该应用的配置文件的压缩包：

![](http://tacimg-1253960454.cosgz.myqcloud.com/guides/project/downloadConfig.png)

解压该压缩包，您会得到 `tac_service_configurations.json` 和 `tac_service_configurations_unpackage.json` 两个文件，请您如图所示添加到您自己的工程中去。

<img src="http://tac-android-libs-1253960454.cosgz.myqcloud.com/tac_android_configuration.jpg" width="50%" height="50%">

>**注意：**
>请您按照图示来添加配置文件，`tac_service_configurations_unpackage.json` 文件中包含了敏感信息，请不要打包到 apk 文件中，MobileLine SDK 也会对此进行检查，防止由于您误打包造成的敏感信息泄露。


## 第三步：集成 SDK

您需要在您应用级 build.gradle 文件（通常是 app/build.gradle）中添加 analytics 服务依赖：

```
dependencies {
    // 增加这行
    compile 'com.tencent.tac:tac-core:1.0.0'
}
```

## 第四步：初始化

集成好我们提供的 SDK 后，您需要在您自己的工程中添加初始化代码，从而让 MobileLine 服务在您的应用中进行自动配置。

### 在 `Application` 子类中添加初始代码（已完成请跳过）

如果您自己的应用中已经有了 `Application` 的子类，请在该类的 `onCreate()` 方法中添加配置代码，如果没有，请自行创建：

```
public class MyCustomApp extends Application {
  @Override
  public void onCreate() {
    super.onCreate();
    ...
    //增加这行
    TACApplication.configure(this);
  }
}

```

>**注意：**
> 如果您同时集成了我们多个服务，只需要添加一次初始化代码，请不要重复添加。

### 在 `AndroidManifest.xml` 文件中注册（已完成请跳过）

在创建好 `Application` 的子类并添加好初始化代码后，您需要在工程的 `AndroidManifest.xml` 文件中注册该 `Application` 类：

```
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
  package="com.example.tac">
  <application
    <!-- 这里替换成你自己的 Application 子类 -->
    android:name="com.example.tac.MyCustomApp"
    ...>
  </application>
</manifest>
```

>**注意：**
>如果您的 `Application` 子类已经在 `AndroidManifest.xml` 文件中注册，请不要重复注册。


### 开启实时上报

Analytics 服务默认采用批量上报策略，在本地缓存事件到达一定数量之后才能集中上报。如果您在调试时，希望每个事件都独立上报，从而能在控制台实时看到手机的上报事件，可以通过下面的方式开启实时上报：

```
TACAnalyticsOptions tacAnalyticsOptions = tacApplicationOptions.sub("analytics");

tacAnalyticsOptions.strategy(TACAnalyticsStrategy.INSTANT);
```

>**注意：**
>由于每次上报都会建立网络连接，会增加手机流量，也会损耗手机电量，影响终端体验，因此建议您在 release 模式下关闭实时上报，采用默认的批量上报策略。

### 启动服务

MobileLine Android SDK 不会自动帮您启动 analytics 服务，请在初始化时创建的 `Application` 子类的 `onCreate()` 方法中来启动 analytics 服务：

```
public class MyCustomApp extends Application {
  @Override
  public void onCreate() {
    super.onCreate();
    ...
    TACApplication.configure(this); // 初始化服务
    
    // 添加这行，必须在初始化服务后调用
    TACAnalyticsService.getInstance().start(this);
  }
}
```

>**注意：**
>您也可以选择在其他地方启动 Analytics 服务，但是必须保证在初始化代码后调用。


到此您已经成功接入了 MobileLine 移动分析服务。
