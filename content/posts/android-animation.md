---
title: "Android动画"
categories: [Android]
tags: [Android, Animation]
date: 2017-10-11T14:08:12+08:00
draft: false
---

有关于Android动画基础知识的理解及笔记记录

### 1. windowAnimation和ActivityAnimation的区别

1. 包含内容

   windowAnimation包括：

   ```
   windowEnterAnimation
   windowExitAnimation
   ```

   ActivityAnimation包含：

   ```
   android:activityOpenEnterAnimation
   android:activityOpenExitAnimation
   android:activityCloseEnterAnimation
   android:activityCloseExitAnimation
   ```

   * 当我们从 A1 启动 A2 时，A1 从屏幕上消失，这个动画叫做 android:activityOpenExitAnimation
   * 当我们从 A1 启动 A2 时，A2 出现在屏幕上，这个动画叫做 android:activityOpenEnterAnimation
   * 当我们从 A2 退出回到 A1 时，A2 从屏幕上消失，这个叫做 android:activityCloseExitAnimation
   * 当我们从 A2 退出回到 A1 时，A1 出现在屏幕上，这个叫做 android:activityCloseEnterAnimation

   ActivityAnimation更为复杂，但这种复杂带来的转场效果非常好，可以同时控制2个Activity的动画，而不像WindowAnimation只能控制下一个Activity的窗体动画。

2.  项目中WindowAnimation的控制权大于Activity的控制权。Activity转场过程中，如果同时设置了WindowAnimation和ActivityAnimation,那么 可能(因为这种情况非常多) 只会执行WindowAnimation

3. WindowAnimation的定义很简单，在style.xml文件中只需要继承Animation Style即可；而ActivityAnimation，需要继承Animation Activity Style

4. 窗体动画也可以使用 Activity.overridePendingTransition来设置,也可以定义在主题中，但Activity只能使用在主题中

### 2. Transition动画的使用

1. 概念

Trasition 内部保存了所有关于动画的信息，当场景变换时动画将在目标对象上执行。Transition基于两个概念：场景（scenes）和变换（transitions），场景是UI当前状态，变换则定义了在不同场景之间动画变化的过程。

分为以下几种：

* enter：用于决定第一次打开当前Activity时的动画
* exit : 用于决定退出当前Activity时的动画
* reenter: 用于决定如果当前Activity已经打开过，并且再次打开该Activity时的动画
* shared elements:用于决定在两个Activity之间切换时，指定两个Activity中对应的View的过渡效果

2. 如何使用

可以通过代码也可以通过设置主题

代码方式，在setContentView之前调用：

```java
getWindow().requestFeature(Window.FEATURE_CONTENT_TRANSITIONS);
```

主题xml方式：

```xml
<item name="android:windowContentTransitions">true</item>
```

* Activity中设置进入，退出Transition动画，Material主题默认会将exit的transition设置成null而enter的transition设置成Fade ，如果reenter 或者 return transition没有明确设置，则将用exit 和enter的transition替代。

* 开始一个activity的content transition需要调用startActivity(Context, Bundle)方法，Bundle参数生成方法ActivityOptions.makeSceneTransitionAnimation(activity, pairs).toBundle();

* 在代码中触发通过finishAfterTransition()方法触发返回动画，而不是调用finish()方法。

[更多内容](https://www.jianshu.com/p/09834d89e733)