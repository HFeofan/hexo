---
title: Runloop的疑惑（一）
date: 2016-12-03 00:06:40
tags: iOS
---

# 从动画说起

在项目中会看到如下代码，动画结束之后，执行`NSRunloop`的`runUntilDate`方法。
``` objective-c
NSRunLoop *currentLoop = [NSRunLoop currentRunLoop];
NSTimeInterval durantion = 0.3;
[UIView animateWithDuration:durantion animations:^{
    self.label.frame = CGRectMake(20, 300, 70, 30);
} completion:^(BOOL finished) {
    self.label.frame = CGRectMake(20, 20, 70, 30);
}];
[currentLoop runUntilDate:[NSDate dateWithTimeIntervalSinceNow:durantion]];
```
为什么要调用这个方法，这个方法是什么作用。先来看看文档是怎么说的：

## runUntilDate:

在runloop中没有input source或者timers，这个方法会立即返回。否则会重复使用`NSDefaultRunLoopMode`模式调用`runMode:beforeDate:`直到指定时间。

另外，如果手动把input sources或者timers从runloop中移除，并不会导致runloop退出。系统可能会根据需要添加或移除input source到runloop，这些操作可能导致runloop不会退出。

## runMode:beforeDate:

如果runloop启动并处理inputsource一次然后阻塞到指定时间就会返回`YES`，否则，如果runloop没有启动返回`NO`。

只是看这两个方法的解释感觉看不出啥，所以需要先了解下Runloop是什么东西。

<!--more-->

# Runloop的内部结构

可以从 https://opensource.apple.com/tarballs/CF/ 这里下载CF代码来看。Runloop的主要结构如下：

![Runloop Struct](/images/runloop0_runloopstruct.png)

## Runloop

Runloop让线程在有工作的时候忙碌，没工作的时候休眠。每一个线程都会关联一个Runloop，主线程的Runloop会由系统启动，其他线程需要手动启动。

Runloop其实就是线程里的一个循环。它从两个地方获取事件，Input sources和Timer sources。Input sources分发异步事件，比如说从其他线程或者App发过来的消息。Timer sources分发同步事件，比如定时或者重复任务。

如上图所示，一个runloop对象包含了若干个mode，但是每次只能以一种mode来运行。

## Run Loop Mode

Runloop Mode是需要被监控的input sources和timers的集合，并且在Runloop状态改变时通知观察者。Runloop只接受当前mode中的sources发出的事件，并对当前mode中的observes通知。而其他mode中的sources发送的事件会被阻塞，等到runloop以其mode运行。

在官方文档中列出了以下mode：

- `NSDefaultRunLoopMode`(`kCFRunLoopDefaultMode`) 大部分操作都是在这个mode里的
- `NSConnectionReplyMode` macos上线程通信用的
- `NSModalPanelRunLoopMode` macos上NSSavePanel或者NSOpenPanel用来等待输入用的
- `NSEventTrackingRunLoopMode` 用来接收用户交互事件
- `NSRunLoopCommonModes`(`kCFRunLoopCommonModes`) mode group，向这个组添加input sources会同步到里面的每个mode，默认包含了`NSDefaultRunLoopMode`、`NSModalPanelRunLoopMode`、`NSEventTrackingRunLoopMode`。也可以通过`CFRunLoopAddCommonMode`添加自定义的mode到这个组

其他文章提到的mode
- `UIInitializationRunLoopMode` 这个是刚启动时使用的mode
- `GSEventReceiveRunLoopMode` 接收系统内部事件的mode

## Input Sources

Input sources分为port-based sources和custom sources，但是对于runloop来说并不会区分这两种sources。这两种source唯一的区别就是port-based sources是由内核自动触发的，而custom sources是从其他线程手动触发的。

### Port-Based Sources

在Cocoa中只需要创建一个port对象并通过`NSPort`的方法添加到runloop中，port对象会自动创建一个input sources。而在Core Foundation中就需要手动创建port和它的sources。

关于如何初始化和设置port-based sources可以参考官方文档。

### Custom Sources

通过`CFRunLoopSourceRef`类型创建一个custom input source，并自定义相关的事件处理和传递机制。

### Perform Selector Sources

Cocoa定义了一个custom input source，这样可以在任何线程执行一个selector。selector的执行在目标线程是线性的，减少了同步问题。在selector执行完之后，相应的source会被移除。如果目标线程没有一个活动的runloop，那么selector不会执行，这点需要注意下。

NSObject的perform selector方法：
- `performSelectorOnMainThread:withObject:waitUntilDone:`
- `performSelectorOnMainThread:withObject:waitUntilDone:modes:`

在主线程的下一个runloop循环中执行selector。wait的作用，是否阻塞当前线程直到slector被执行。YES时阻塞该线程，NO时持有selector并立即返回。

- `performSelector:onThread:withObject:waitUntilDone:`
- `performSelector:onThread:withObject:waitUntilDone:modes:`
同上，可以指定任意已有的线程。

- `performSelector:withObject:afterDelay:`
- `performSelector:withObject:afterDelay:inModes:`

delay一段时间并在线程的下一次runloop循环中执行，因为需要等到下次循环，所以会自动有一个很小的delay时间。selector会被保存到一个队列里，并依次执行。

- `cancelPreviousPerformRequestsWithTarget:`
- `cancelPreviousPerformRequestsWithTarget:selector:object:`

如果通过`performSelector:withObject:afterDelay:`和`performSelector:withObject:afterDelay:inModes:`执行的selector可以通过这两个方法取消。


## Timer Sources

Timer用来执行一些定时任务，虽然它会生成一个time-based的通知，但它并不是一个实时的。timer在被注册后会自动根据注册的时间间隔设定事件，如果中间因为某些原因导致事件的执行被延迟，那么这个事件会被跳过。Tiemr有个`tolerance`宽容度属性，可以接受一定的延迟，默认值为0。

如果runloop没有运行，那么timer也不会激活。

## Run Loop Observer

``` C
enum CFRunLoopActivity {
   kCFRunLoopEntry = (1 << 0),
   kCFRunLoopBeforeTimers = (1 << 1),
   kCFRunLoopBeforeSources = (1 << 2),
   kCFRunLoopBeforeWaiting = (1 << 5),
   kCFRunLoopAfterWaiting = (1 << 6),
   kCFRunLoopExit = (1 << 7),
   kCFRunLoopAllActivities = 0x0FFFFFFFU 
};
typedef enum CFRunLoopActivity CFRunLoopActivity;
```
注册通知
``` OBjective-C
CFRunLoopObserverContext  context = {0, (__bridge void *)(self), NULL, NULL, NULL};
CFRunLoopObserverRef observer = CFRunLoopObserverCreate(kCFAllocatorDefault, kCFRunLoopEntry, YES, 0, &mainRunLoopObserver, &context);
if (observer)
{
    CFRunLoopAddObserver(CFRunLoopGetMain(), observer, kCFRunLoopCommonModes);
}
```

# Runloop的运行过程

1. 通知观察者进入runloop(`kCFRunLoopEntry`)
2. 通知观察者将要处理timer(`kCFRunLoopBeforeTimers`)
3. 通知观察者将要处理非port-based(source0)的input sources的事件(`kCFRunLoopBeforeSources`)
4. 处理source0的事件
5. 如果port-based sources(source1)的事件ready，就处理这些事件并跳转到 9
6. 通知观察者runloop将要休眠(`kCFRunLoopBeforeWaiting`)
7. 如果发生以下事件就唤醒runloop
	- port-based sources的事件到来
	- timer
	- runloop超时
	- 手动被唤醒
8. 通知观察者runloop被唤醒(`kCFRunLoopAfterWaiting`)
9. 处理被挂起的事件
	- 处理用户定义的timer，处理之后重启runloop并进入 2
	- 如果有input sources事件，就发送该事件
	- 如果runloop是被手动唤醒且没有超时，就重启runloop并进入 2
10. 通知观察者runloop退出(`kCFRunLoopExit`)

因为timer和input sources的通知是在真正执行事件之前，所以这两个时间之间会有间隔。如果觉得这个间隔很重要，可以通过`kCFRunLoopBeforeWaiting`和`kCFRunLoopAfterWaiting`这两个通知来关联与真正执行事件的时间。

向runloop添加非port-base的input source也可以使runloop唤醒。

Runloop的初始结构
``` C
{
  current mode = UIInitializationRunLoopMode,
  common modes = {
    "UITrackingRunLoopMode",
    "kCFRunLoopDefaultMode"
  },
  common mode items = {
    //source0
    CFRunLoopSource = {order = -1, {callout = PurpleEventSignalCallback}},
    CFRunLoopSource = {order = 0, {callout = FBSSerialQueueRunLoopSourceHandler}},
    CFRunLoopSource = {order = -2, {callout = __handleHIDEventFetcherDrain}},
    CFRunLoopSource = {order = -1, {callout = __handleEventQueue}},
    //source1
    CFRunLoopSource = {order = 0, {port = 18955}},
    CFRunLoopSource = {order = -1, {callout = PurpleEventCallback}},
    CFRunLoopSource = {order = 0, {port = 1d03, callout = _ZL20notify_port_callbackP12__CFMachPortPvlS1_}},
    CFRunLoopSource = {order = 0, {port = 23819}},
    //observers
    //0xa0 kCFRunLoopAfterWaiting | kCFRunLoopExit
    //0x1 kCFRunLoopEntry
    //0x20 kCFRunLoopBeforeWaiting
    CFRunLoopObserver = {activities = 0xa0, order = 2000000, callout = _ZN2CA11Transaction17observer_callbackEP19__CFRunLoopObservermPv},
    CFRunLoopObserver = {activities = 0xa0, order = 2147483647, callout = _wrapRunLoopWithAutoreleasePoolHandler},
    CFRunLoopObserver = {activities = 0x1, order = -2147483647, callout = _wrapRunLoopWithAutoreleasePoolHandler},
    CFRunLoopObserver = {activities = 0xa0, order = 2001000, callout = _afterCACommitHandler},
    CFRunLoopObserver = {activities = 0xa0, order = 1999000, callout = _beforeCACommitHandler},
    CFRunLoopObserver = {activities = 0x20, order = 0, callout = _UIGestureRecognizerUpdateObserver},
  },
  modes = {
    CFRunLoopMode = {
      name = UITrackingRunLoopMode, 
      sources0 = { /*同common mode items*/ },
      sources1 = { /*同common mode items*/ },
      observers = { /*同common mode items*/ },
      timers = (null),
    },

    CFRunLoopMode = {
      name = GSEventReceiveRunLoopMode, 
      sources0 = {
        CFRunLoopSource = {order = -1,{callout = PurpleEventSignalCallback}}
      },
      sources1 = {
        CFRunLoopSource = {order = -1, {callout = PurpleEventCallback}}
      },
      observers = (null),
      timers = (null),
    },

    CFRunLoopMode = {
      name = kCFRunLoopDefaultMode, 
      sources0 = { /*同common mode items*/ },
      sources1 = { /*同common mode items*/ },
      observers = { /*同common mode items*/ },
      timers = {
        CFRunLoopTimer = {callout = (Delayed Perform) UIApplication _accessibilitySetUpQuickSpeak}
			},
      },

    CFRunLoopMode = {
      name = UIInitializationRunLoopMode, 
      sources0 = {
        CFRunLoopSource = {callout = FBSSerialQueueRunLoopSourceHandler}
      },
      sources1 = {},
      observers = {
        CFRunLoopObserver = {activities = 0xa0, callout = _ZN2CA11Transaction17observer_callbackEP19__CFRunLoopObservermPv}
      },
      timers = (null),
    },

    CFRunLoopMode = {
      name = kCFRunLoopCommonModes, 
      sources0 = (null),
      sources1 = (null),
      observers = (null),
      timers = (null),
     },
  }
}
```

# 动画与runloop的关系？

从上边的runloop初始结构中可以看到有一个observe的回调为`_ZN2CA11Transaction17observer_callbackEP19__CFRunLoopObservermPv`，这个是Core Animation注册的observe，用于将UI的中间态提交到GPU，如果有动画，Core Animation会通过DisplayLink机制重复相关流程。
所以，如果在动画代码的下边有一些会占用CPU的任务，导致当前runloop不能到退出或者唤醒状态，这时候动画就会被阻塞。可以通过`runUntilDate`方法驱动runloop完成动画显示。

具体的渲染过程可以看下面的参考链接。

# 参考资料
1, [官方资料](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/Multithreading/RunLoopManagement/RunLoopManagement.html#//apple_ref/doc/uid/10000057i-CH16-SW1)
2, [走进Runloop的世界](http://chun.tips/blog/2014/10/20/zou-jin-run-loopde-shi-jie-%5B%3F%5D-:shi-yao-shi-run-loop%3F/)
3, [深入理解Runloop](http://blog.ibireme.com/2015/05/18/runloop/)
4, [iOS 保持界面流畅的技巧](http://blog.ibireme.com/2015/11/12/smooth_user_interfaces_for_ios/)
5, [iOS 事件处理机制与图像渲染过程](http://mp.weixin.qq.com/s?__biz=MzAwNDY1ODY2OQ==&mid=400417748&idx=1&sn=0c5f6747dd192c5a0eea32bb4650c160&scene=4#wechat_redirect)
