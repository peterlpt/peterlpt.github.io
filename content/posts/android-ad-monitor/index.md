---
title: Android原生广告曝光点击监测实现
categories: [Android]
tags: [Android, 广告监测, 曝光]
date: 2019-09-14T21:08:32+08:00
draft: false
---

以对业务实现层最少侵入为原则，在SDK层实现对Android原生广告View的曝光监听上报、点击监听上报，做到业务层只需调用统一注册方法告知SDK层该View为广告控件，剩余工作由SDK层内部完成。

## 1 实现效果概述

![广告监测, 动图大小7.7MB](android_ad_monitor.gif "广告监测整体实现效果")

说明：

* 广告控件View在屏幕可见且无遮挡、曝光时长超过1秒、可见面积超过50%，在结束曝光时生成一条曝光事件
* 生成曝光事件同时对广告信息的曝光异步监测链接进行上报
* 广告控件View发生点击时对广告信息的点击异步监测链接进行上报
* 熄屏、切换到后台等任何会导致不满足曝光条件的事件发生时，结算曝光行为
* 动图演示中，虽然展示足够久，在广告图只展示一小部分时，不会触发曝光上报
* 动图演示中，广告图展示面积足够大，在展示时长不足1秒时，不会触发曝光上报

## 2 功能需求

* 对App内所有原生广告位的广告监测曝光、点击行为
* 为监测需同时支持自主监测、第三方（如国双、秒针等）异步监测
* 支持同步监测
* 曝光行为需记录每次曝光时长，及其他常规用户行为属性
* 只有广告曝光面积超过一半，且持续曝光时间超过1秒的，才能作为有效曝光

关于广告同步监测、异步监测：[一篇文章读懂S2S 与C2S、同步监测与异步监测](https://zhuanlan.zhihu.com/p/67810432)

## 3 实现方案分析

初步分析需求，我们可以发现，整体功能实现的关键点在于如何有效实现对一个View的显示状态实时监听。查阅了一些资料文档，原生并没有提供直接监听的方法，因此着手构想如何组合多种行为的监听，才能准确、实时，且相对优雅的方式监听View的显示状态。

常见会引起View显示状态发生变化的行为有：

* 代码逻辑需要View#setVisibility
* View所在页面发生滚动、布局重绘、View重绘、View添加或移除于窗口
* 页面发生跳转导致View所在页面不在栈顶
* 所在Activity、Fragment等页面生命周期触发，如onResume、onPause、onDestroyed等

1. View#getVisibility的变化，没有提供监听、观察事件，我们只能通过监听、观察其他事件触发时再来判断，判断时我们可以使用View#isShown()即得到View及其所有parent的Visibility判断，顺带截取其源码：

```java
/**
 * Returns the visibility of this view and all of its ancestors
 * @return True if this view and all of its ancestors are {@link #VISIBLE}
 */
public boolean isShown() {
   View current = this;
   //noinspection ConstantConditions
   do {
       if ((current.mViewFlags & VISIBILITY_MASK) != VISIBLE) {
           return false;
       }
       ViewParent parent = current.mParent;
       if (parent == null) {
           return false; // We are not attached to the view root
       }
       if (!(parent instanceof View)) {
           return true;
       }
       current = (View) parent;
   } while (current != null);
   return false;
}
```

2. 关于View所在页面发生滚动、布局重绘、View重绘、View添加或移除于窗口的监听，我们可以通过[ViewTreeObserver](https://developer.android.com/reference/android/view/ViewTreeObserver)进行订阅相关事件

3. 关于页面发生跳转导致View所在页面不在栈顶、所在Activity、Fragment等页面生命周期触发，如onResume、onPause、onDestroyed等，我们可以通过注册Application提供的Activity生命周期回调进行监听：[Application#registerActivityLifecycleCallbacks](https://developer.android.com/reference/kotlin/android/app/Application?hl=zh#registerActivityLifecycleCallbacks(android.app.Application.ActivityLifecycleCallbacks))

实现对View显示状态准确、实时监听后，其他功能实现问题就不大了：

* 曝光开始时记录开始时间
* 曝光结束时生成曝光时长并对配置的广告曝光异步监测进行上报
* 在每次判断View是否真实可见时，叠加曝光面积的判断
* View发生点击时对配置的广告点击异步监测进行上报
* 考虑到App广告位分散、监测逻辑迁移到其他App的便利性，需要将整体实现逻辑尽量下沉到SDK，在业务开发层只需要调用注册方法标记告知View为广告及广告的基本配置信息即可
* 由于Android的View提供的点击监听View#setOnClickListener，不支持添加多个监听，为不影响到业务开发层点击监听需求，只能在SDK中提供发生点击的方法给到业务开发层在发生点击时调用

同步监测功能是无需特别实现的，只要能正常跳转打开广告落地页即可满足需求。

## 4 功能实现

### 4.1 View显示状态判定

```java
/**
 * @return 返回目标View是否真实用户可见
 */
private static boolean isViewShow(View view) {
    if (view == null) {
        return false;
    }
    //如果view所在Activity并不在栈顶，此View一定不在显示状态
    if (view.getContext() != null
            && !(view.getContext().getClass().getSimpleName() + view.getContext().hashCode()).equals(topActivityInfo)) {
        return false;
    }

    int width = view.getWidth();
    int height = view.getHeight();
    if (width == 0 || height == 0) {
        //view的面积为0，一定不在显示状态
        return false;
    }
    Rect r = new Rect();
    // 首先判断View的基本visibility，再判断当前View在其ViewTree中有没被遮挡
    boolean isShow = view.isShown() && view.getLocalVisibleRect(r);
    if (!isShow) {
        return false;
    }

    //需求要求计算可见面积，如果小于View面积一半，仍要标记为未展示
    int area = width * height;
    int displayArea = (r.right - r.left) * (r.bottom - r.top);
    boolean isValidShow = area <= displayArea << 1;
    if (!isValidShow) {
        Log.i("--->", "展示面积小于一半，判定为不可见：" + shortStr(view.toString()));
    }
    return isValidShow;
}
```

### 4.2 Activity生命周期变化利用

```java
/**
 * 注册Application级别的全局Activity生命周期订阅事件<br/>
 * <strong>NOTE:</>重复调用只会在第一次调用时注册
 * @param context ApplicationContext
 */
private static void regListener(Context context) {
    if (!isLifecycleListened) {
        checkNotNull(context, "上下文不能为空");
        if (!(context instanceof Application)) {
            throw new IllegalArgumentException("上下文必须为ApplicationContext");
        }
        ((MainApplication) context).registerActivityLifecycleCallbacks(new Application.ActivityLifecycleCallbacks() {
            @Override
            public void onActivityCreated(Activity activity, Bundle savedInstanceState) {
            }
            @Override
            public void onActivityStarted(Activity activity) {
            }
            @Override
            public void onActivityResumed(Activity activity) {
                //onResumed时更新当前栈顶页面信息，之所以使用页面名+实例哈希码一起标识记录，考虑到Activity可能被多次实例于栈中
                topActivityInfo = activity.getClass().getSimpleName() + activity.hashCode();
                //找出当前页面已注册的广告控件，刷新显示状态
                List<AdViews> adViewsFound = foundActivityAdViews(activity);
                for (AdViews adView : adViewsFound) {
                    onAdViewShowStateChanged(adView.adView, true);
                }
            }
            @Override
            public void onActivityPaused(Activity activity) {
                //此时所有在当前页面的控件一定是不可见的
                List<AdViews> adViewsFound = foundActivityAdViews(activity);
                for (AdViews adView : adViewsFound) {
                    onAdViewShowStateChanged(adView.adView, false);
                }
            }
            @Override
            public void onActivityStopped(Activity activity) {
            }
            @Override
            public void onActivitySaveInstanceState(Activity activity, Bundle outState) {
            }
            @Override
            public void onActivityDestroyed(Activity activity) {
                List<AdViews> adViewsFound = foundActivityAdViews(activity);
                if (adViewsFound.size() > 0) {
                    for (AdViews adView : adViewsFound) {
                        if (adView.isShow) {
                            //如果页面异常终止，可能还有广告控件还来不及更新成未显示，补充更新
                            onAdViewShowStateChanged(adView.adView, false);
                        }
                    }
                    //把已终止页面中的所有广告控件均移除
                    adViewsList.removeAll(adViewsFound);
                }
            }
        });
        isLifecycleListened = true;
    }
}
```

### 4.3 当View显示状态可能发生变化时统一更新View的显示状态

```java
/**
 * 统一更新View的显示状态，当更新为显示时，里面会调 {@link #isViewShow(View)}进一步判断；当更新为不显示时，
 * 如果之前是在显示状态，会组装一条曝光事件记录，之后更新状态为不显示
 *
 * @param isShow 只需要初步判定可能是显示的即可传true，只有确认为一定不显示时才传false
 */
private static void onAdViewShowStateChanged(View view, boolean isShow) {
    if (adViewsList == null) {
        return;
    }
    AdViews adViewsTarget = null;
    for (AdViews adView : adViewsList) {
        if (adView.adView.equals(view)) {
            adViewsTarget = adView;
            break;
        }
    }
    if (adViewsTarget == null) {
        //未在显示中的广告控件列表中
        return;
    }
    if (isShow) {
        if (adViewsTarget.isShow) {
            //如果原来已经在显示了，本次也是"变为"显示（通常是重复调用了），则不需要处理什么
            return;
        }
        //重新判断是否在Activity内屏幕中显示
        if (isViewShow(view)) {
            //更新下目标广告控件显示状态
            adViewsTarget.setShow(true);
        }
    } else {
        if (!adViewsTarget.isShow) {
            //如果原来已经不在显示了，本次也是"变为"不显示（通常是重复调用了），则不需要处理什么
            return;
        }
        //需求要求曝光时长必须大于1秒时才能视为有效曝光
        long nowTime = System.currentTimeMillis();
        if (nowTime - adViewsTarget.startTime >= 1000) {
            //已经可以形成一对完整曝光时长时间，组成事件执行一次记录
            Log.i("--->", "生成一条广告曝光事件记录，广告位=" + adViewsTarget.adMonitorAttr.placeEventId + ", 广告标识=" +
                    adViewsTarget.adMonitorAttr.adTag + ", 曝光时长=" + (System.currentTimeMillis() - adViewsTarget.startTime) + "ms");

            doAsyncReport(adViewsTarget.adMonitorAttr, API_AD_RPT_TYPE_SHOW);

            //同时对配置的广告曝光异步监测进行上报
            onAdShow(adViewsTarget.adMonitorAttr);
        } else {
            Log.i("--->", "曝光" + (nowTime - adViewsTarget.startTime) + "ms，时长小于1秒，丢掉：广告位=" + adViewsTarget.adMonitorAttr.placeEventId +
                    ", 广告标识=" + adViewsTarget.adMonitorAttr.adTag);
        }

        //完成事件生成后更新同步控件显示状态
        adViewsTarget.setShow(false);
    }
}
```

### 4.4 订阅相关ViewTree事件

```java
private static void addViewTreeObserverListener(final View view) {
    ViewTreeObserver viewTreeObserver = view.getViewTreeObserver();
    if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.JELLY_BEAN) {
        viewTreeObserver.addOnDrawListener(new ViewTreeObserver.OnDrawListener() {
            @Override
            public void onDraw() {
                //发生绘制事件时有可能会变更显示状态
                onAdViewShowStateChanged(view, isViewShow(view));
            }
        });
    }
    viewTreeObserver.addOnScrollChangedListener(new ViewTreeObserver.OnScrollChangedListener() {
        @Override
        public void onScrollChanged() {
            //发生滑动事件时有可能会变更显示状态
            onAdViewShowStateChanged(view, isViewShow(view));
        }
    });
    if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.JELLY_BEAN_MR2) {
        viewTreeObserver.addOnWindowAttachListener(new ViewTreeObserver.OnWindowAttachListener() {
            @Override
            public void onWindowAttached() {
                //重新着附于窗口事件时有可能会变更显示状态
                onAdViewShowStateChanged(view, isViewShow(view));
            }
            @Override
            public void onWindowDetached() {
                //此时一定为不显示状态
                onAdViewShowStateChanged(view, false);
            }
        });
    }
}
```

### 4.5 设计注册入口方法

```java
/**
 * 注册一个广告控件，注册为广告控件的View，会在Sdk中完成显示状态的监听，并完成广告曝光事件的统计
 * <strong>NOTE: </strong>在页面实例未被重建前已注册过的控件无需重复注册，重复调用只更新广告监测属性
 *
 * @param view          目标控件，只有非空才能有效注册
 * @param adMonitorAttr 广告监测属性，只有非空才能有效注册
 */
public static void regAdView(final View view, AdMonitorAttr adMonitorAttr) {
    if (view == null || view.getContext() == null || adMonitorAttr == null) {
        return;
    }
    AdViews adViewsReged = findView(view);
    if (adViewsReged != null) {
        Log.i(TAG, view + " 上一次注册还有效，本次只更新广告监测属性");
        //先把之前的设置为关闭，以形成曝光事件
        onAdViewShowStateChanged(view, false);
        //再更新其他属性
        adViewsReged.adMonitorAttr = adMonitorAttr;
        adViewsReged.setShow(isViewShow(view));
        return;
    }
    if (adViewsList == null) {
        adViewsList = new ArrayList<>();
    }
    AdViews adView = new AdViews(view, adMonitorAttr);
    adView.setShow(isViewShow(view));
    adViewsList.add(adView);

    //注册所在Activity生命周期监听
    regListener(view.getContext().getApplicationContext());

    //订阅相关ViewTree事件
    addViewTreeObserverListener(view);
}
```

### 4.6 提供广告发生点击时调用的方法

```java
public static void onAdClick(View view) {
    if (view == null) {
        return;
    }
    AdViews adViews = findView(view);
    if (adViews == null || adViews.adMonitorAttr == null) {
        Log.w(TAG, "广告控件未注册，点击监测响应丢掉：" + view);
        return;
    }
    List<String> urls = adViews.adMonitorAttr.getClickRptUrls();
    Log.i("--->", "异步上报广告\"" + adViews.adMonitorAttr.adTag + "\"点击监测链接：" + getGson().toJson(adViews.adMonitorAttr.clickRptUrls));
    
    doThirdAsyncReportUrls(urls);
    doAsyncReport(adViews.adMonitorAttr, API_AD_RPT_TYPE_CLICK);
}
```

### 4.7 整体实现源码

[Android原生广告曝光点击监测实现SDK及本demo源码](https://github.com/peterlpt/AdMonitor.git)

## 5 方案已知缺陷

**5.1 通过View#getLocalVisibleRect判断当前View在其ViewTree中有没被遮挡是只能判断所在ViewTree中有没被遮挡**

这个方法只能检查出这个View在手机屏幕（或者说是相对它的父View）的位置，而不能检查出与其他兄弟View的相对位置。

比如说有一个ViewGroup，下面有View1、View2这两个子View，View1和View2是平级关系。此时如果View2盖住了View1，那么用getGlobalVisibleRect方法检查View1的可见性，得到的返回值依然是true，得到的可见矩形区域rect也是没有任何变化的。也就是说**View1.getLocalVisibleRect(rect)得到的结果与View2没有任何关系。**

好在这种case以目前App页面View结构复杂度来看几乎没有，影响暂时忽略

## 6 参考资料

[1. 一篇文章读懂S2S 与C2S、同步监测与异步监测](https://zhuanlan.zhihu.com/p/67810432)

[2. Android View的可见性检查方法-上篇](http://unclechen.github.io/2016/10/17/Android%20View%E7%9A%84%E5%8F%AF%E8%A7%81%E6%80%A7%E6%A3%80%E6%9F%A5%E6%96%B9%E6%B3%95-%E4%B8%8A%E7%AF%87/)