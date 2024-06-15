---
title: Android应用启动时间
categories: [Android]
tags: [Android,]
date: 2023-09-10T18:18:18+08:00
draft: false
---

# Android应用启动时间



【转自：[App startup time  | App quality  | Android Developers](https://developer.android.com/topic/performance/vitals/launch-time)】

用户希望应用能够及时响应并快速加载。启动时间过长的应用不能满足这个期望，并且可能会令用户失望。这种糟糕的体验可能会导致用户在 Play 商店针对您的应用给出很低的评分，甚至完全抛弃您的应用。

本文档提供的信息可以帮助您优化应用的启动时间。 首先介绍启动过程的内部机制。然后，讨论如何分析启动性能。最后，介绍一些常见的启动时间问题，并给出一些有关如何解决这些问题的提示。

## 了解不同的应用启动状态

应用有三种启动状态，每种状态都会影响应用向用户显示所需的时间：冷启动、温启动或热启动。在冷启动中，应用从头开始启动。在另外两种状态中，系统需要将后台运行的应用带入前台。建议您始终在假定冷启动的基础上进行优化。这样做也可以提升温启动和热启动的性能。

如需优化应用以实现快速启动，了解系统和应用层面的情况以及它们在各个状态中的互动方式很有帮助。

### 冷启动

冷启动是指应用从头开始启动：系统进程在冷启动后才创建应用进程。发生冷启动的情况包括应用自设备启动后或系统终止应用后首次启动。这种启动给最大限度地减少启动时间带来了最大的挑战，因为系统和应用要做的工作比在另外两种启动状态中更多。

在冷启动开始时，系统有三个任务，分别是：

1. 加载并启动应用。
2. 在启动后立即显示应用的空白启动窗口。
3. 创建应用[进程](https://developer.android.google.cn/guide/components/processes-and-threads?hl=zh-cn#Processes)。

系统一创建应用进程，应用进程就负责后续阶段：

1. 创建应用对象。
2. 启动主线程。
3. 创建主 activity。
4. 扩充视图。
5. 布局屏幕。
6. 执行初始绘制。

一旦应用进程完成第一次绘制，系统进程就会换掉当前显示的后台窗口，替换为主 activity。此时，用户可以开始使用应用。

图 1 显示系统进程和应用进程之间如何交接工作。

![img](cold-launch.png)

**图 1.** 以可视化方式呈现应用冷启动的重要部分。

在创建应用和创建 activity 的过程中可能会出现性能问题。

#### 应用创建

当应用启动时，空白启动窗口将保留在屏幕上，直到系统首次完成应用绘制。完成后，系统进程会换掉应用的启动窗口，允许用户开始与应用互动。

如果您在自己的应用中替换了 [`Application.onCreate()`](https://developer.android.google.cn/reference/android/app/Application?hl=zh-cn#onCreate())，系统将对应用对象调用 `onCreate()` 方法。之后，应用生成主线程（也称为界面线程），并用其执行创建主 activity 的任务。

从此时开始，系统级和应用级进程根据[应用生命周期阶段](https://developer.android.google.cn/guide/topics/processes/process-lifecycle?hl=zh-cn)继续运行。

#### activity 创建

在应用进程创建 activity 后，activity 将执行以下操作：

1. 初始化值。
2. 调用构造函数。
3. 根据 activity 的当前生命周期状态，相应地调用回调方法，如 [`Activity.onCreate()`](https://developer.android.google.cn/reference/android/app/Activity?hl=zh-cn#onCreate(android.os.Bundle))。

通常，[`onCreate()`](https://developer.android.google.cn/reference/android/app/Activity?hl=zh-cn#onCreate(android.os.Bundle)) 方法对加载时间的影响最大，因为它执行工作的开销最高：加载和膨胀视图，以及初始化运行 activity 所需的对象。

### 温启动

温启动包含了在冷启动期间发生的部分操作；同时，它的开销要比热启动高。有许多潜在状态可视为温启动。例如：

- 用户在退出应用后又重新启动应用。进程可能已继续运行，但应用必须通过调用 `onCreate()` 从头开始重新创建 activity。
- 系统将您的应用从内存中逐出，然后用户又重新启动它。进程和 activity 需要重启，但传递到 `onCreate()` 的已保存的实例 state bundle 对于完成此任务有一定助益。

### 热启动

应用的热启动比冷启动简单得多，开销也更低。在热启动中，系统的所有工作就是将您的 activity 带到前台。只要应用的所有 activity 仍驻留在内存中，应用就不必重复执行对象初始化、布局膨胀和呈现。

但是，如果一些内存为响应内存整理事件（如 [`onTrimMemory()`](https://developer.android.google.cn/reference/android/content/ComponentCallbacks2?hl=zh-cn#onTrimMemory(int))）而被完全清除，则需要为了响应热启动事件而重新创建相应的对象。

热启动显示的屏幕上行为和冷启动场景相同：

在应用完成 activity 呈现之前，系统进程将显示空白屏幕。



![](startup-modes-r1.png?hl=zh-cn)

**图 2.** 此图显示了各种启动状态及其各自的进程，其中每个状态都将从绘制的第一帧开始。



## 使用指标检测和诊断问题

如需正确诊断启动时间性能，您可以跟踪一些显示应用启动所需时间的指标。Android 提供多种方式，让您知道您的应用有问题，并帮助您进行诊断。Android Vitals 可以提醒您正在发生问题，而诊断工具可以帮助您诊断问题。

### 使用启动指标的优势

Android 使用[初步显示所用时间](https://developer.android.google.cn/topic/performance/vitals/launch-time?hl=zh-cn#time-initial)和[完全显示所用时间](https://developer.android.google.cn/topic/performance/vitals/launch-time?hl=zh-cn#time-full)指标来优化冷应用启动和温应用启动。Android 运行时 (ART) 使用这些指标的数据来高效地预编译代码，以优化未来启动。

更快的启动速度可以促进用户与应用的持续互动，从而减少过早退出、重启实例或前往其他应用的情况。

### Android Vitals

当您的应用启动时间过长时，Android Vitals 可以通过 [Play 管理中心](https://play.google.com/console/?hl=zh-cn)提醒您，从而帮助提升应用性能。Android Vitals 在您的应用出现以下情况时将其启动时间视为过长：

- [冷](https://developer.android.google.cn/topic/performance/vitals/launch-time?hl=zh-cn#cold)启动用了 5 秒或更长时间。
- [温](https://developer.android.google.cn/topic/performance/vitals/launch-time?hl=zh-cn#warm)启动用了 2 秒或更长时间。
- [热](https://developer.android.google.cn/topic/performance/vitals/launch-time?hl=zh-cn#hot)启动用了 1.5 秒或更长时间。

Android Vitals 使用[初步显示所用时间](https://developer.android.google.cn/topic/performance/vitals/launch-time?hl=zh-cn#time-initial)指标。如需了解 Google Play 如何收集 Android Vitals 数据，请参阅 [Play 管理中心](https://support.google.com/googleplay/android-developer/answer/7385505?hl=zh-cn)文档。

### 初步显示所用时间

初步显示所用时间 (TTID) 指标用于测量应用生成第一帧所用的时间，包括进程初始化（如果是冷启动）、activity 创建（如果是冷启动/温启动）以及显示第一帧。

#### 如何检索 TTID

在 Android 4.4（API 级别 19）及更高版本中，Logcat 包含一个输出行，该行包含名为 `Displayed` 的值。此值代表从启动进程到在屏幕上完成对应 activity 的绘制所用的时间。经过的时间包括以下事件序列：

- 启动进程。
- 初始化对象。
- 创建并初始化 activity。
- 扩充布局。
- 首次绘制应用。

报告的日志行类似于以下示例：

```
ActivityManager: Displayed com.android.myexample/.StartupTiming: +3s534ms
```

如果您从命令行或在终端中跟踪 Logcat 输出，查找经过的时间很简单。如需在 Android Studio 中查找经过的时间，必须在 logcat 视图中停用过滤器。停用过滤器是必要的，因为提供此日志的是系统服务器，不是应用本身。

一旦进行了正确的设置，即可轻松搜索正确术语来查看时间。图 2 展示了一个 Logcat 输出示例，其中显示了如何停用过滤器，并且在输出内容的倒数第二行显示了 `Displayed` 时间。

![img](displayed-logcat.jpg?hl=zh-cn)

**图 2.** 停用过滤器，并在 Logcat 中查找“Displayed”值。

在所有资源完全加载并显示之前，Logcat 输出中的 `Displayed` 指标不一定会捕获时间：它会省去布局文件中未引用的资源或应用作为对象初始化一部分创建的资源。它之所以排除这些资源，是因为加载它们是一个内嵌进程，并且不会阻止应用的初步显示。

有时，logcat 输出中的 `Displayed` 行中会包含用于显示总时间的附加字段。例如：

```
ActivityManager: Displayed com.android.myexample/.StartupTiming: +3s534ms (total +1m22s643ms)
```

在这种情况下，第一个时间测量值仅针对第一个绘制的 activity。`total` 时间测量值是从应用进程启动时开始计算，并且可以包含首次启动但未在屏幕上显示任何内容的另一个 activity。`total` 时间测量值仅在单个 activity 的时间和总启动时间之间存在差异时才会显示。

您也可以使用 [ADB Shell Activity Manager](https://developer.android.google.cn/studio/command-line/shell?hl=zh-cn#am) 命令运行应用来测量初步显示所用时间。示例如下：

```
adb [-d|-e|-s &lt;serialNumber>] shell am start -S -W
com.example.app/.MainActivity
-c android.intent.category.LAUNCHER
-a android.intent.action.MAIN</pre>
```

`Displayed` 指标和以前一样出现在 logcat 输出中。您的终端窗口还应显示以下内容：

```
Starting: Intent
Activity: com.example.app/.MainActivity
ThisTime: 2044
TotalTime: 2044
WaitTime: 2054
Complete
```

`-c` 和 `-a` 为可选参数，可让您指定 [``](https://developer.android.google.cn/guide/topics/manifest/category-element?hl=zh-cn) 和 [``](https://developer.android.google.cn/guide/topics/manifest/action-element?hl=zh-cn)

### 完全显示所用时间

完全显示所用时间 (TTFD) 指标用于测量应用生成具有完整内容的第一帧所用的时间，包括在第一帧之后异步加载的内容。一般情况下，这是从网络加载的主要列表内容（由应用报告）。

#### 如何检索 TTFD

您可以使用 [`reportFullyDrawn()`](https://developer.android.google.cn/reference/android/app/Activity?hl=zh-cn#reportFullyDrawn()) 方法测量从应用启动到完全显示所有资源和视图层次结构所用的时间。在应用执行延迟加载时，此数据会很有用。在延迟加载中，应用不会阻止窗口的初步绘制，但会异步加载资源并更新视图层次结构。

如果由于延迟加载，应用的初步显示不包括所有资源，您可能会将完全加载并显示所有资源及视图视为单独的指标：例如，您的界面可能已完全加载，并绘制了一些文本，但尚未显示应用必须从网络中提取的图片。

如需解决此问题，您可以手动调用 `reportFullyDrawn()`，让系统知道您的 activity 已完成延迟加载。当您使用此方法时，logcat 显示的值为从创建应用对象到调用 `reportFullyDrawn()` 时所用的时间。以下是 logcat 输出的示例：

```
system_process I/ActivityManager: Fully drawn {package}/.MainActivity: +1s54ms
```

logcat 输出有时包含 `total` 时间，如[初步显示所用时间](https://developer.android.google.cn/topic/performance/vitals/launch-time?hl=zh-cn#time-initial)中所述。

如果您发现您的显示时间比希望的时间长，则可以继续尝试识别启动过程中的瓶颈。

##### 识别瓶颈

查找瓶颈的一个好方法是使用 Android Studio CPU 性能分析器。相关信息请参阅[使用 CPU 性能分析器检查 CPU 活动](https://developer.android.google.cn/studio/profile/cpu-profiler?hl=zh-cn)。

您也可以通过应用和 activity 的 `onCreate()` 方法内部的内嵌跟踪记录深入了解潜在瓶颈。如需了解内嵌跟踪记录，请参阅 [`Trace`](https://developer.android.google.cn/reference/android/os/Trace?hl=zh-cn) 函数文档和[系统跟踪记录概览](https://developer.android.google.cn/topic/performance/tracing?hl=zh-cn)。

## 注意常见问题

本节讨论几个通常会影响应用启动性能的问题。这些问题主要涉及初始化应用和 activity 对象，以及屏幕加载。

### 密集型应用初始化

在您的代码替换 `Application` 对象，并在初始化该对象过程中执行密集工作或复杂逻辑时，启动性能可能会受影响。 如果您的应用子类执行尚不需要完成的初始化，您的应用可能会在启动过程中浪费时间。有些初始化可能完全没有必要：例如，当应用为了响应 intent 而实际上已经启动时，初始化主 activity 的状态信息就是不必要的。通过 intent，应用仅使用之前初始化状态数据的一个子集。

应用初始化过程中的其他挑战包括影响范围较大或数量众多的垃圾回收事件，或与初始化同时发生、会进一步阻止初始化过程的磁盘 I/O。垃圾回收是 Dalvik 运行时特别需要考虑的问题；Art 运行时同时执行垃圾回收，从而最大限度地减少该操作的影响。

#### 诊断问题

您可以使用方法跟踪记录或内嵌跟踪记录来尝试诊断问题。

##### 方法跟踪记录

运行 CPU 性能分析器显示，[`callApplicationOnCreate()`](https://developer.android.google.cn/reference/android/app/Instrumentation?hl=zh-cn#callApplicationOnCreate(android.app.Application)) 方法最终调用您的 `com.example.customApplication.onCreate` 方法。如果该工具显示这些方法需要很长时间才能完成执行，您应该进一步探索以查看正在进行哪些工作。

##### 内嵌跟踪记录

使用内嵌跟踪记录调查可能的问题根源，包括：

- 应用的初始 [`onCreate()`](https://developer.android.google.cn/reference/android/app/Application?hl=zh-cn#onCreate()) 函数。
- 应用初始化的任何全局单例对象。
- 在瓶颈期间可能发生的任何磁盘 I/O、反序列化或紧密循环。

#### 问题解决方案

不管问题在于不必要的初始化还是磁盘 I/O，解决方案都是延迟初始化。换言之，您应当仅初始化立即需要的对象。采用单例模式，让应用仅在首次需要对象时初始化对象，而不是创建全局静态对象。

此外，考虑使用依赖项注入框架（如 [Hilt](https://developer.android.google.cn/training/dependency-injection/hilt-android?hl=zh-cn)），它们会在首次注入时创建对象和依赖项。

如果您的应用使用 content provider 在启动时初始化应用组件，请考虑改用 [App Startup 库](https://developer.android.google.cn/topic/libraries/app-startup?hl=zh-cn)。

### 密集型 activity 初始化

创建 activity 通常需要进行大量的高开销工作。通常有机会优化这项工作以实现性能改进。此类常见问题包括：

- 膨胀大型或复杂的布局。
- 阻止磁盘上的屏幕绘制或网络 I/O。
- 加载和解码位图。
- 栅格化 [`VectorDrawable`](https://developer.android.google.cn/reference/android/graphics/drawable/VectorDrawable?hl=zh-cn) 对象。
- 初始化 activity 的其他子系统。

#### 诊断问题

事实证明，在这种情况下，方法跟踪记录和内嵌跟踪记录同样很有用。

##### 方法跟踪记录

使用 CPU 性能分析器时，请注意应用的 [`Application`](https://developer.android.google.cn/reference/android/app/Application?hl=zh-cn) 子类构造函数和 `com.example.customApplication.onCreate()` 方法。

如果该工具显示这些方法需要很长时间才能完成执行，您应该进一步探索以查看正在进行哪些工作。

##### 内嵌跟踪记录

使用内嵌跟踪记录调查可能的问题根源，包括：

- 应用的初始 `onCreate()` 函数。
- 应用初始化的任何全局单例对象。
- 在瓶颈期间可能发生的任何磁盘 I/O、反序列化或紧密循环。

#### 问题解决方案

潜在瓶颈有很多，但两种常见问题和补救措施如下所示：

- 您的视图层次结构越大，应用膨胀它所花的时间就越长。解决此问题的两个步骤是：
  - 通过减少冗余或嵌套布局，展平您的视图层次结构。
  - 不要膨胀在启动期间无需显示的界面部分，而是使用 [`ViewStub`](https://developer.android.google.cn/reference/android/view/ViewStub?hl=zh-cn) 对象作为应用可以在更合适的时间膨胀的子层次结构的占位符。
- 在主线程上进行所有资源初始化也会降低启动速度。您可以按以下方式解决此问题：
  - 转移所有资源初始化，以便应用可以在其他线程上延迟执行。
  - 允许应用加载并显示您的视图，稍后再更新依赖于位图和其他资源的可视属性。

### 自定义启动画面

如果您之前曾在 Android 11（API 级别 30）或更低版本中使用以下某种方法来实现自定义启动画面，则可能会增加额外的启动时间：

- 使用 [`windowDisablePreview`](https://developer.android.google.cn/reference/android/R.attr?hl=zh-cn#windowDisablePreview) 主题属性关闭系统在启动过程中绘制的初始空白屏幕。
- 使用专用 activity。

从 Android 12 开始，必须迁移到 [`SplashScreen`](https://developer.android.google.cn/reference/android/window/SplashScreen?hl=zh-cn) API。此 API 可以缩短启动时间，并允许您通过以下方式调整启动画面：

- [设置主题以更改启动画面的外观](https://developer.android.google.cn/guide/topics/ui/splash-screen?hl=zh-cn#set-theme)
- [控制启动画面的显示时长](https://developer.android.google.cn/guide/topics/ui/splash-screen?hl=zh-cn#suspend-drawing)
- [确定启动画面动画需要多长的额外时间，并合理处理动画以关闭启动画面](https://developer.android.google.cn/guide/topics/ui/splash-screen?hl=zh-cn#customize-animation)

此外，compat 库会[向后移植 SplashScreen API](https://developer.android.google.cn/reference/kotlin/androidx/core/splashscreen/SplashScreen?hl=zh-cn) 以支持向后兼容性，并在所有 Android 版本上实现一致的启动画面显示效果。

如需了解详情，请参阅[启动画面迁移指南](https://developer.android.google.cn/guide/topics/ui/splash-screen/migrate?hl=zh-cn)。