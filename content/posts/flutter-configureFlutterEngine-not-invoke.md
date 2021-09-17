---
title: "Add-to-app的MethodCallHandler不生效抛MissingPluginException解决"
categories: [Android, Flutter, Dart]
tags: [Flutter, add-to-app]
date: 2020-04-25T19:09:23+08:00
draft: false
---

Add-to-app模式下，在FlutterActivity子类configureFlutterEngine里，因设置的MethodChannel及MethodCallHandler未触发，导致Flutter Module通过MethodChannel调用原生平台时一直抛MissingPluginException的问题分析及解决。

<!--more-->

### 1. 需求背景及遇到的问题现象

经过一段时间的Flutter调研试用后，我们决定在现有App中添加一个用Flutter实现的业务模块。Flutter Module与Host App之间的互相调用通信是必须具备的能力。

Flutter定义了三种不同类型的Platform Channel用于Flutter与Host App平台进行通信，它们分别是

* BasicMessageChannel：用于传递字符串和半结构化的信息，可以双向的请求数据。
* MethodChannel：用于传递方法调用（method invocation，即Flutter端可以调用Platform端的方法并通过Result接口回调结果数据。

* EventChannel: 用于数据流（event streams）的通信，即Flutter端监听Platform端的实时消息，一旦Platform端产生了数据，立即回调到Flutter端。

我们选用了MethodChannel，解决Flutter module调用Host App已有方法的需求：

* 根据“[向 Android 应用中添加 Flutter 页面](https://flutter.cn/docs/development/add-to-app/android/add-flutter-screen)”指引，在Host App中完成Flutter module的依赖

* 根据“[撰写双端平台代码](https://flutter.cn/docs/development/platform-integration/platform-channels?tab=android-channel-java-tab)”指引：
  * 在Host App里，继承[FlutterActivity](https://api.flutter.dev/javadoc/io/flutter/embedding/android/FlutterActivity.html)，在[configureFlutterEngine](https://api.flutter.dev/javadoc/io/flutter/embedding/android/FlutterActivity.html#configureFlutterEngine-io.flutter.embedding.engine.FlutterEngine-)([FlutterEngine](https://api.flutter.dev/javadoc/io/flutter/embedding/engine/FlutterEngine.html) flutterEngine)中添加[MethodChannel](https://api.flutter.dev/javadoc/io/flutter/plugin/common/MethodChannel.html)，并设置[MethodChannel.MethodCallHandler](https://api.flutter.dev/javadoc/io/flutter/plugin/common/MethodChannel.MethodCallHandler.html)

  * 在Flutter module里，使用相同的channelName创建[MethodChannel](https://api.flutter-io.cn/flutter/services/MethodChannel/MethodChannel.html)

* 在Host App的任意原生页面位置，使用以下方式跳转FlutterModule指定路由页面

```java
startActivity(FlutterActivity
        .withNewEngine()
        .initialRoute("/my_route")
        .build(currentActivity));
```

* 在Flutter页面，试图使用methodChannel调起Host App方法，即报MissingPluginException：

```dart
methodChannel.invokeMethod('testMethod').then((value) {
  //获取HostApp对应方法的返回值value
}).catchError((e) {});
```

### 2. 简化问题重现demo代码

为聚集于问题描述，这里简化问题重现demo代码如下：

* HostApp

定义`FlutterActivity`子类`BridgeActivity`，并在`configureFlutterEngine`设置`MethodChannel`及`MethodCallHandler`

```java
package com.peter.myapplication;
import androidx.annotation.NonNull;
import io.flutter.embedding.android.FlutterActivity;
import io.flutter.embedding.engine.FlutterEngine;
import io.flutter.plugin.common.MethodChannel;

public class BridgeActivity extends FlutterActivity {
    private static final String CHANNEL_NAME = "flutter.bridge.call_platform";

    @Override
    public void configureFlutterEngine(@NonNull FlutterEngine flutterEngine) {
        super.configureFlutterEngine(flutterEngine);
        getFlutterEngine();
        //config a method channel
        new MethodChannel(flutterEngine.getDartExecutor().getBinaryMessenger(), CHANNEL_NAME)
                .setMethodCallHandler((call, result) -> {
                    switch (call.method) {
                        case "testMethod":
                            result.success("This is respond result from Android native method");
                            break;
                        default:
                            result.notImplemented();
                            break;
                    }
                });
    }
}
```

在其他原生页面`MainActivity`中设置一个按钮，点击跳转Flutter页面

```java
public class MainActivity extends AppCompatActivity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        // click button to jump flutter activity
        findViewById(R.id.tv_test).setOnClickListener((view) -> {
            Intent intent = BridgeActivity.withNewEngine()
                    .build(MainActivity.this);
            intent.setClass(MainActivity.this, BridgeActivity.class);
            startActivity(intent);
        });
    }
}
```

将Activity配置`AndroidManifest.xml`

注意：只配置`BridgeActivity`，运行会抛`FlutterActivity`未配置的错误？

* Flutter module

在`main.dart`的`StatefulWidget`中设置按钮，在跳转来Flutter页面后，点击按钮时，期望通过约定channelName的`MethodChannel`获取host app中`testMethod`方法的返回值

```dart
class MyAppState extends State<MyAppStatefulPage> {
  static const channelName = 'flutter.bridge.call_platform';
  static const _platform = MethodChannel(channelName);

  String _hostAppMethodRst = '';

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
        title: 'Flutter Demo',
        home: Scaffold(
            appBar: AppBar(
              title: Text('Flutter Module Page'),
            ),
            body: Container(
              width: double.infinity,
              padding: EdgeInsets.all(16),
              child: Column(
                crossAxisAlignment: CrossAxisAlignment.center,
                children: <Widget>[
                  RaisedButton(
                    child: Text('Call Host App'),
                    onPressed: () {
                      _platform.invokeMethod('testMethod').then((value) {
                        setState(() => _hostAppMethodRst = _hostAppMethodRst + '\n' + value);
                      });
                    },
                  ),
                  Text('Respond from host App:\n $_hostAppMethodRst')
                ],
              ),
            )));
  }
}
```

* 运行demo

在跳转Flutter页面后，点击按钮`Call Host App`，抛MissingPluginException

```shell
[ERROR:flutter/lib/ui/ui_dart_state.cc(157)] Unhandled Exception: MissingPluginException(No implementation found for method testMethod on channel flutter.bridge.call_platform)
    #0      MethodChannel._invokeMethod (package:flutter/src/services/platform_channel.dart:154:7)
    <asynchronous suspension>
    #1      MethodChannel.invokeMethod (package:flutter/src/services/platform_channel.dart:329:12)
    #2      MyAppState.build.<anonymous closure> (package:fluttermodule/main.dart:48:33)
    #3      _InkResponseState._handleTap (package:flutter/src/material/ink_well.dart:779:19)
    #4      _InkResponseState.build.<anonymous closure> (package:flutter/src/material/ink_well.dart:862:36)
    #5      GestureRecognizer.invokeCallback (package:flutter/src/gestures/recognizer.dart:182:24)
    #6      TapGestureRecognizer.handleTapUp (package:flutter/src/gestures/tap.dart:504:11)
    #7      BaseTapGestureRecognizer._checkUp (package:flutter/src/gestures/tap.dart:282:5)
    #8      BaseTapGestureRecognizer.handlePrimaryPointer (package:flutter/src/gestures/tap.dart:217:7)
    #9      PrimaryPointerGestureRecognizer.handleEvent (package:flutter/src/gestures/recognizer.dart:475:9)
    #10     PointerRouter._dispatch (package:flutter/src/gestures/pointer_router.dart:76:12)
    #11     PointerRouter._dispatchEventToRoutes.<anonymous closure> (package:flutter/src/gestures/pointer_router.dart:122:9)
    #12     _LinkedHashMapMixin.forEach (dart:collection-patch/compact_hash.dart:379:8)
    #13     PointerRouter._dispatchEventToRoutes (package:flutter/src/gestures/pointer_router.dart:120:18)
    #14     PointerRouter.route (package:flutter/src/gestures/pointer_router.dart:106:7)
    #15     GestureBinding.handleEvent (package:flutter/src/gestures/binding.dart:218:19)
    #16     GestureBinding.dispatchEvent (package:flutter/src/gestures/binding.dart:198:22)
    #17     GestureBinding._handlePointerEvent (package:flutter/src/gestures/binding.dart:156:7)
    #18     GestureBinding._flushPointerEventQueue (package:flutter/src/gestures/binding.dart:102:7)
    #19     GestureBinding._handlePointerDataPacket (package:flutter/src/gestures/binding.dart:86:7)
    #20     _rootRunUnary (dart:async/zone.dart:1196:13)
    #21     _CustomZone.runUnary (dart:async/zone.dart:1085:19)
    #22     _CustomZone.runUnaryGuarded (dart:async/zone.dart:987:7)
    #23     _invoke1 (dart:ui/hooks.dart:275:10)
    #24     _dispatchPointerDataPacket (dart:ui/hooks.dart:184:5)
```

### 3. MissingPluginException问题排查分析

调试发现，`BridgeActivity#configureFlutterEngine`方法根本就没执行，不但这个方法没执行，其实`BridgeActivity#onCreate`一样未被执行，即其实以下方法启动的Flutter页面并非attach于`BridgeActivity`：

```java
Intent intent = BridgeActivity.withNewEngine().build(MainActivity.this);
startActivity(intent);
```

进去父类`FlutterActivity`的`io.flutter.embedding.android.FlutterActivity#withNewEngine`及其`io.flutter.embedding.android.FlutterActivity.NewEngineIntentBuilder#build`方法：

```java
public static NewEngineIntentBuilder withNewEngine() {
    //初步发现问题：这里直接写死了`FlutterActivity.class`，而非动态获取到`BridgeActivity.class`
    return new NewEngineIntentBuilder(FlutterActivity.class);
}
public static class NewEngineIntentBuilder {
    private final Class<? extends FlutterActivity> activityClass;
    private String initialRoute = DEFAULT_INITIAL_ROUTE;
    private String backgroundMode = DEFAULT_BACKGROUND_MODE;

    /**
     * Constructor that allows this {@code NewEngineIntentBuilder} to be used by subclasses of
     * {@code FlutterActivity}.
     *
     * <p>Subclasses of {@code FlutterActivity} should provide their own static version of {@link
     * #withNewEngine()}, which returns an instance of {@code NewEngineIntentBuilder} constructed
     * with a {@code Class} reference to the {@code FlutterActivity} subclass, e.g.:
     *
     * <p>{@code return new NewEngineIntentBuilder(MyFlutterActivity.class); }
     */
    protected NewEngineIntentBuilder(@NonNull Class<? extends FlutterActivity> activityClass) {
      this.activityClass = activityClass;
    }

    /**
     * Creates and returns an {@link Intent} that will launch a {@code FlutterActivity} with the
     * desired configuration.
     */
    @NonNull
    public Intent build(@NonNull Context context) {
      //build方法里，直接使用NewEngineIntentBuilder Constructor传入的activityClass构建Intent
      return new Intent(context, activityClass)
          .putExtra(EXTRA_INITIAL_ROUTE, initialRoute)
          .putExtra(EXTRA_BACKGROUND_MODE, backgroundMode)
          .putExtra(EXTRA_DESTROY_ENGINE_WITH_ACTIVITY, true);
    }
  }
```

可以看到，`io.flutter.embedding.android.FlutterActivity#withNewEngine`里直接写死了`FlutterActivity.class`，而非动态获取到`BridgeActivity.class`！

再看`io.flutter.embedding.android.FlutterActivity.NewEngineIntentBuilder#NewEngineIntentBuilder`构造函数，可发现这个Constructor已经考虑到了，是应该要支持传入`FlutterActivity`子类的，javadoc也提醒了我们`FlutterActivity`子类必须自行提供withNewEngine()返回用子类构建的`NewEngineIntentBuilder`

然而，问题来了，**这个Constructor被`protected`修饰！**我们根本无法在`FlutterActivity`子类`BridgeActivity`参考父类提供`withNewEngine`方法来返回新的`NewEngineIntentBuilder`实例。

至此，在配置`AndroidManifest.xml`只配置`BridgeActivity`，运行会抛`FlutterActivity`未配置的错误，原因也就找到了：按上述代码`NewEngineIntentBuilder#build`出来的`Intent`是指向`FlutterActivity`！

也因此，其子类`BridgeActivity#onCreate`、`BridgeActivity#configureFlutterEngine`未被执行，本就未创建`BridgeActivity`……

这里附带上Flutter sdk版本信息：

```shell
$ flutter doctor -v
[✓] Flutter (Channel beta, v1.17.0-3.2.pre, on Mac OS X 10.15.3 19D76, locale
    zh-Hans-CN)
    • Flutter version 1.17.0-3.2.pre at /Users/peter/Data/flutter
    • Framework revision 2a7bc389f2 (11 days ago), 2020-04-21 20:34:20 -0700
    • Engine revision 4c8c31f591
    • Dart version 2.8.0 (build 2.8.0-dev.20.10)
```

### 4. 问题解决

定位到问题原因，通常解决方法也就明朗，这已经可以肯定是Flutter SDK的bug，只需将`io.flutter.embedding.android.FlutterActivity.NewEngineIntentBuilder#NewEngineIntentBuilder`构造函数改用`public`修饰就好，我们就可以根据其javadoc的指引，在子类提供对应withNewEngine()，返回用子类构建的`NewEngineIntentBuilder`

但是，我们项目肯定不能等待sdk升级才发版

* 方法一：重新设置目标class

其实，也不难发现，使用`NewEngineIntentBuilder#build`出来的`Intent`，最后还是由我们代码的`context.startActivity`进行触发的，那我们在触发前再重新设置一下目标class不就行了？

```java
Intent intent = BridgeActivity.withNewEngine().build(context);
//重新设置一下目标class到子类BridgeActivity.class
intent.setClass(context, BridgeActivity.class);
context.startActivity(intent);
```

调整好代码运行，问题果然得到解决。

* 方法二：通过继承调用被`protected`修饰的Constructor

利用`protected`修饰的方法可以被子类访问的特性，通过继承`NewEngineIntentBuilder`调用被`protected`修饰的Constructor：

```java
public class BridgeActivity extends FlutterActivity {
    //...
    //继承`NewEngineIntentBuilder`调用被`protected`修饰的Constructor
		public static class MyNewEngineIntentBuilder extends NewEngineIntentBuilder {
        MyNewEngineIntentBuilder(@NonNull Class<? extends FlutterActivity> activityClass) {
            super(activityClass);
        }
    }
    //接着则可以按提示自行提供withNewEngine()返回用子类构建的`NewEngineIntentBuilder`
    public static FlutterActivity.NewEngineIntentBuilder withNewEngine() {
        return new MyNewEngineIntentBuilder(BridgeActivity.class);
    }
  
    //类似的，如果有使用CachedEngine，也可能继承`CachedEngineIntentBuilder`调用被`protected`修饰的Constructor来返回用子类构建的`CachedEngineIntentBuilder`
    public static FlutterActivity.CachedEngineIntentBuilder withCachedEngine(@NonNull String cachedEngineId) {
        return new MyCachedEngineIntentBuilder(BridgeActivity.class, cachedEngineId);
    }
    public static class MyCachedEngineIntentBuilder extends CachedEngineIntentBuilder{
        protected MyCachedEngineIntentBuilder(@NonNull Class<? extends FlutterActivity> activityClass, @NonNull String engineId) {
            super(activityClass, engineId);
        }
    }
}
```

之后就可以按照官方的指导方式进行Flutter页面跳转了：

```java
//直接跳转，不使用cacheEngine
Intent intent = BridgeActivity
            .withNewEngine()
            .initialRoute("initialRoute")
            .backgroundMode(FlutterActivityLaunchConfigs.BackgroundMode.transparent)
            .build(context);
context.startActivity(intent);

//使用cacheEngine跳转
Intent intent = BridgeActivity
        .withCachedEngine("your_cache_id")
        .backgroundMode(FlutterActivityLaunchConfigs.BackgroundMode.transparent)
        .build(context);
context.startActivity(intent);
```



### 5. 其他

[这里整理了demo完整代码](https://gitee.com/ptlpt/flutter_add-to-app_bugfix.git)



