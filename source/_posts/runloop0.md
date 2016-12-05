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

另外，如果手动把input source或者timers从runloop中移除，并不会导致runloop退出。系统可能会根据需要添加或移除input source到runloop，这些操作可能导致runloop不会退出。

## runMode:beforeDate:

如果runloop启动并处理inputsource一次然后阻塞到指定时间就会返回`YES`，否则，如果runloop没有启动返回`NO`。

只是看这两个方法的解释感觉看不出啥，所以需要先了解下Runloop是什么东西。

# Runloop的内部结构

可以从 https://opensource.apple.com/tarballs/CF/ 这里下载CF代码来看。Runloop的主要结构如下：

![Runloop Struct](/images/runloop0_runloopstruct.png)

## Runloop

Runloop让线程在有工作的时候忙碌，没工作的时候休眠。每一个线程都会关联一个Runloop，主线程的Runloop会由系统启动，其他线程需要手动启动。

Runloop其实就是线程里的一个循环。它从两个地方获取事件，Input sources和Timer sources。Input sources分发异步事件，比如说从其他线程或者App发过来的消息。Timer sources分发同步事件，比如定时或者重复任务。

如上图所示，一个runloop对象包含了若干个mode，但是每次只能以一种mode来运行。

## Run Loop Mode

Runloop Mode是需要被监控的input sources和timer的集合，并且在Runloop状态改变时通知观察者。Runloop只有当前mode中的sources允许发送事件，observers也是一样。而其他mode中的sources发送的事件会被阻塞，等到runloop以其mode运行。

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

### Perform Selector Sources

Cocoa定义了一个custom input source，这样可以在任何线程执行一个selector。selector的执行在目标线程是线性的，减少了同步问题。在selector执行完之后，相应的source会被移除。如果目标线程没有一个活动的runloop，那么selector不会执行，这点需要注意下。

NSObject的perform selector方法：
- `performSelectorOnMainThread:withObject:waitUntilDone:`
- `performSelectorOnMainThread:withObject:waitUntilDone:modes:`

## Timer Source


## Run Loop Observer

# Runloop的运行机制

在输出
```
{
	current mode = UIInitializationRunLoopMode,
	common modes = {
		"UITrackingRunLoopMode",
		"kCFRunLoopDefaultMode"
	},
	common mode items = {
		CFRunLoopSource = {signalled = No, valid = Yes, order = -1, context = <CFRunLoopObserver context>{version = 0, callout = PurpleEventSignalCallback }},

		CFRunLoopSource = {signalled = No, valid = Yes, order = 0, context = <CFRunLoopSource MIG Server> { port = 18955, subsystem = 0x10c29a9a0, context = 0x0}},

		CFRunLoopSource ={signalled = Yes, valid = Yes, order = 0, context = <CFRunLoopSource context>{version = 0, callout = FBSSerialQueueRunLoopSourceHandler}},

		CFRunLoopObserver = {valid = Yes, activities = 0xa0, repeats = Yes, order = 2000000, callout = _ZN2CA11Transaction17observer_callbackEP19__CFRunLoopObservermPv},

		CFRunLoopObserver = {valid = Yes, activities = 0xa0, repeats = Yes, order = 2147483647, callout = _wrapRunLoopWithAutoreleasePoolHandler, context = <CFArray>{type = mutable-small, count = 0, values = ()}}

		9 : <CFRunLoopObserver>{valid = Yes, activities = 0x1, repeats = Yes, order = -2147483647, callout = _wrapRunLoopWithAutoreleasePoolHandler, context = <CFArray>{type = mutable-small, count = 0, values = ()}}

		10 : <CFRunLoopObserver>{valid = Yes, activities = 0xa0, repeats = Yes, order = 2001000, callout = _afterCACommitHandler, context = <CFRunLoopObserver context 0x7fa9b9e01c40>}

		12 : <CFRunLoopObserver>{valid = Yes, activities = 0xa0, repeats = Yes, order = 1999000, callout = _beforeCACommitHandler, context = <CFRunLoopObserver context 0x7fa9b9e01c40>}

		13 : <CFRunLoopSource>{signalled = No, valid = Yes, order = -1, context = <CFRunLoopSource context>{version = 1, info = 0x3903, callout = PurpleEventCallback}}

		14 : <CFRunLoopObserver>{valid = Yes, activities = 0x20, repeats = Yes, order = 0, callout = _UIGestureRecognizerUpdateObserver (0x10b9346e1), context = <CFRunLoopObserver context 0x6000000c9b50>}

		15 : <CFRunLoopSource>{signalled = No, valid = Yes, order = 0, context = <CFRunLoopSource MIG Server> {port = 23819, subsystem = 0x10c2b04d0, context = 0x600000038420}}

		20 : <CFRunLoopSource>{signalled = No, valid = Yes, order = -1, context = <CFRunLoopSource context>{version = 0, info = 0x600000116260, callout = __handleEventQueue}}

		21 : <CFRunLoopSource>{signalled = No, valid = Yes, order = 0, context = <CFMachPort 0x600000144a40 [0x10b29ec70]>{valid = Yes, port = 1d03, source = 0x60000017c740, callout = _ZL20notify_port_callbackP12__CFMachPortPvlS1_, context = <CFMachPort context 0x0>}}

		22 : <CFRunLoopSource>{signalled = No, valid = Yes, order = -2, context = <CFRunLoopSource context>{version = 0, info = 0x600000052c00, callout = __handleHIDEventFetcherDrain}}
	},
	modes = {
		 : <CFRunLoopMode>{
		name = UITrackingRunLoopMode, 
		port set = 0x2303, 
		queue = 0x60000017c980, 
		source = 0x6000001d8240 (not fired), 
		timer port = 0x2503, 
		sources0 = {
			0 : <CFRunLoopSource 0x60000017d100 [0x10b29ec70]>{signalled = No, valid = Yes, order = -1, context = <CFRunLoopSource context>{version = 0, info = 0x0, callout = PurpleEventSignalCallback (0x10ed3a733)}}
		1 : <CFRunLoopSource 0x60000017d880 [0x10b29ec70]>{signalled = No, valid = Yes, order = -1, context = <CFRunLoopSource context>{version = 0, info = 0x600000116260, callout = __handleEventQueue (0x10bc0c508)}}
		2 : <CFRunLoopSource 0x60000017d700 [0x10b29ec70]>{signalled = No, valid = Yes, order = -2, context = <CFRunLoopSource context>{version = 0, info = 0x600000052c00, callout = __handleHIDEventFetcherDrain (0x10bc0dbe5)}}
		3 : <CFRunLoopSource 0x60800017dc40 [0x10b29ec70]>{signalled = Yes, valid = Yes, order = 0, context = <CFRunLoopSource context>{version = 0, info = 0x608000076380, callout = FBSSerialQueueRunLoopSourceHandler (0x10e54bdfc}}
	},
	sources1 = {
		0 : <CFRunLoopSource 0x60800017cb00 [0x10b29ec70]>{signalled = No, valid = Yes, order = -1, context = <CFRunLoopSource context>{version = 1, info = 0x3903, callout = PurpleEventCallback (0x10ed3cc30)}}
		1 : <CFRunLoopSource 0x60000017c740 [0x10b29ec70]>{signalled = No, valid = Yes, order = 0, context = <CFMachPort 0x600000144a40 [0x10b29ec70]>{valid = Yes, port = 1d03, source = 0x60000017c740, callout = _ZL20notify_port_callbackP12__CFMachPortPvlS1_ (0x10fb47f79), context = <CFMachPort context 0x0>}}
		5 : <CFRunLoopSource 0x60000017e000 [0x10b29ec70]>{signalled = No, valid = Yes, order = 0, context = <CFRunLoopSource MIG Server> {port = 23819, subsystem = 0x10c2b04d0, context = 0x600000038420}}
		6 : <CFRunLoopSource 0x60000017d940 [0x10b29ec70]>{signalled = No, valid = Yes, order = 0, context = <CFRunLoopSource MIG Server> {port = 18955, subsystem = 0x10c29a9a0, context = 0x0}}
	},
	observers = (
    	"<CFRunLoopObserver 0x60000013ae00 [0x10b29ec70]>{valid = Yes, activities = 0x1, repeats = Yes, order = -2147483647, callout = _wrapRunLoopWithAutoreleasePoolHandler (0x10b3fda9a), context = <CFArray 0x600000053710 [0x10b29ec70]>{type = mutable-small, count = 0, values = ()}}",
    	"<CFRunLoopObserver 0x60000013aae0 [0x10b29ec70]>{valid = Yes, activities = 0x20, repeats = Yes, order = 0, callout = _UIGestureRecognizerUpdateObserver (0x10b9346e1), context = <CFRunLoopObserver context 0x6000000c9b50>}",
    	"<CFRunLoopObserver 0x60000013ac20 [0x10b29ec70]>{valid = Yes, activities = 0xa0, repeats = Yes, order = 1999000, callout = _beforeCACommitHandler (0x10b42f676), context = <CFRunLoopObserver context 0x7fa9b9e01c40>}",
    	"<CFRunLoopObserver 0x60000013afe0 [0x10b29ec70]>{valid = Yes, activities = 0xa0, repeats = Yes, order = 2000000, callout = _ZN2CA11Transaction17observer_callbackEP19__CFRunLoopObservermPv (0x10fbeb012), context = <CFRunLoopObserver context 0x0>}",
    	"<CFRunLoopObserver 0x60000013ad60 [0x10b29ec70]>{valid = Yes, activities = 0xa0, repeats = Yes, order = 2001000, callout = _afterCACommitHandler (0x10b42f71e), context = <CFRunLoopObserver context 0x7fa9b9e01c40>}",
    	"<CFRunLoopObserver 0x60000013aea0 [0x10b29ec70]>{valid = Yes, activities = 0xa0, repeats = Yes, order = 2147483647, callout = _wrapRunLoopWithAutoreleasePoolHandler (0x10b3fda9a), context = <CFArray 0x600000053710 [0x10b29ec70]>{type = mutable-small, count = 0, values = ()}}"
	),
	timers = (null),
},

	3 : <CFRunLoopMode 0x60000018c7e0 [0x10b29ec70]>{name = GSEventReceiveRunLoopMode, port set = 0x3003, queue = 0x60000017d1c0, source = 0x6000001d8420 (not fired), timer port = 0x3203, 
	sources0 = <CFBasicHash 0x600000052ed0 [0x10b29ec70]>{type = mutable set, count = 1,
entries =>
	0 : <CFRunLoopSource 0x60000017d100 [0x10b29ec70]>{signalled = No, valid = Yes, order = -1, context = <CFRunLoopSource context>{version = 0, info = 0x0, callout = PurpleEventSignalCallback (0x10ed3a733)}}
	}
,
	sources1 = <CFBasicHash 0x600000052f00 [0x10b29ec70]>{type = mutable set, count = 1,
entries =>
	0 : <CFRunLoopSource 0x60800017cc80 [0x10b29ec70]>{signalled = No, valid = Yes, order = -1, context = <CFRunLoopSource context>{version = 1, info = 0x3903, callout = PurpleEventCallback (0x10ed3cc30)}}
}
,
	observers = (null),
	timers = (null),
	currently 502435236 (95658819009078) / soft deadline in: 1.84466484e+10 sec (@ -1) / hard deadline in: 1.84466484e+10 sec (@ -1)
},

	4 : <CFRunLoopMode 0x60000018c570 [0x10b29ec70]>{name = kCFRunLoopDefaultMode, port set = 0x1f03, queue = 0x60000017c800, source = 0x6000001d8060 (not fired), timer port = 0x2103, 
	sources0 = <CFBasicHash 0x600000051d30 [0x10b29ec70]>{type = mutable set, count = 4,
entries =>
	0 : <CFRunLoopSource 0x60000017d100 [0x10b29ec70]>{signalled = No, valid = Yes, order = -1, context = <CFRunLoopSource context>{version = 0, info = 0x0, callout = PurpleEventSignalCallback (0x10ed3a733)}}
	1 : <CFRunLoopSource 0x60000017d880 [0x10b29ec70]>{signalled = No, valid = Yes, order = -1, context = <CFRunLoopSource context>{version = 0, info = 0x600000116260, callout = __handleEventQueue (0x10bc0c508)}}
	2 : <CFRunLoopSource 0x60000017d700 [0x10b29ec70]>{signalled = No, valid = Yes, order = -2, context = <CFRunLoopSource context>{version = 0, info = 0x600000052c00, callout = __handleHIDEventFetcherDrain (0x10bc0dbe5)}}
	3 : <CFRunLoopSource 0x60800017dc40 [0x10b29ec70]>{signalled = Yes, valid = Yes, order = 0, context = <CFRunLoopSource context>{version = 0, info = 0x608000076380, callout = FBSSerialQueueRunLoopSourceHandler (0x10e54bdfc)}}
}
,
	sources1 = <CFBasicHash 0x600000051d60 [0x10b29ec70]>{type = mutable set, count = 4,
entries =>
	0 : <CFRunLoopSource 0x60800017cb00 [0x10b29ec70]>{signalled = No, valid = Yes, order = -1, context = <CFRunLoopSource context>{version = 1, info = 0x3903, callout = PurpleEventCallback (0x10ed3cc30)}}
	1 : <CFRunLoopSource 0x60000017c740 [0x10b29ec70]>{signalled = No, valid = Yes, order = 0, context = <CFMachPort 0x600000144a40 [0x10b29ec70]>{valid = Yes, port = 1d03, source = 0x60000017c740, callout = _ZL20notify_port_callbackP12__CFMachPortPvlS1_ (0x10fb47f79), context = <CFMachPort context 0x0>}}
	5 : <CFRunLoopSource 0x60000017e000 [0x10b29ec70]>{signalled = No, valid = Yes, order = 0, context = <CFRunLoopSource MIG Server> {port = 23819, subsystem = 0x10c2b04d0, context = 0x600000038420}}
	6 : <CFRunLoopSource 0x60000017d940 [0x10b29ec70]>{signalled = No, valid = Yes, order = 0, context = <CFRunLoopSource MIG Server> {port = 18955, subsystem = 0x10c29a9a0, context = 0x0}}
}
,
	observers = (
    "<CFRunLoopObserver 0x60000013ae00 [0x10b29ec70]>{valid = Yes, activities = 0x1, repeats = Yes, order = -2147483647, callout = _wrapRunLoopWithAutoreleasePoolHandler (0x10b3fda9a), context = <CFArray 0x600000053710 [0x10b29ec70]>{type = mutable-small, count = 0, values = ()}}",
    "<CFRunLoopObserver 0x60000013aae0 [0x10b29ec70]>{valid = Yes, activities = 0x20, repeats = Yes, order = 0, callout = _UIGestureRecognizerUpdateObserver (0x10b9346e1), context = <CFRunLoopObserver context 0x6000000c9b50>}",
    "<CFRunLoopObserver 0x60000013ac20 [0x10b29ec70]>{valid = Yes, activities = 0xa0, repeats = Yes, order = 1999000, callout = _beforeCACommitHandler (0x10b42f676), context = <CFRunLoopObserver context 0x7fa9b9e01c40>}",
    "<CFRunLoopObserver 0x60000013afe0 [0x10b29ec70]>{valid = Yes, activities = 0xa0, repeats = Yes, order = 2000000, callout = _ZN2CA11Transaction17observer_callbackEP19__CFRunLoopObservermPv (0x10fbeb012), context = <CFRunLoopObserver context 0x0>}",
    "<CFRunLoopObserver 0x60000013ad60 [0x10b29ec70]>{valid = Yes, activities = 0xa0, repeats = Yes, order = 2001000, callout = _afterCACommitHandler (0x10b42f71e), context = <CFRunLoopObserver context 0x7fa9b9e01c40>}",
    "<CFRunLoopObserver 0x60000013aea0 [0x10b29ec70]>{valid = Yes, activities = 0xa0, repeats = Yes, order = 2147483647, callout = _wrapRunLoopWithAutoreleasePoolHandler (0x10b3fda9a), context = <CFArray 0x600000053710 [0x10b29ec70]>{type = mutable-small, count = 0, values = ()}}"
),
	timers = <CFArray 0x6000000a37e0 [0x10b29ec70]>{type = mutable-small, count = 1, values = (
	0 : <CFRunLoopTimer 0x60000017d280 [0x10b29ec70]>{valid = Yes, firing = No, interval = 0, tolerance = 0, next fire date = 502435218 (-18.531627 @ 95640399418039), callout = (Delayed Perform) UIApplication _accessibilitySetUpQuickSpeak (0x10a587da7 / 0x10b8679ef) (/Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator.sdk/System/Library/Frameworks/UIKit.framework/UIKit), context = <CFRunLoopTimer context 0x6000000710c0>}
)},
	currently 502435236 (95658819067861) / soft deadline in: 1.84467441e+10 sec (@ 95640399418039) / hard deadline in: 1.84467441e+10 sec (@ 95640399418039)
},

	5 : <CFRunLoopMode 0x60800018d000 [0x10b29ec70]>{name = UIInitializationRunLoopMode, port set = 0x4507, queue = 0x60800017dd00, source = 0x6080001d8150 (not fired), timer port = 0x4607, 
	sources0 = <CFBasicHash 0x608000055b70 [0x10b29ec70]>{type = mutable set, count = 1,
entries =>
	2 : <CFRunLoopSource 0x60800017dc40 [0x10b29ec70]>{signalled = Yes, valid = Yes, order = 0, context = <CFRunLoopSource context>{version = 0, info = 0x608000076380, callout = FBSSerialQueueRunLoopSourceHandler (0x10e54bdfc)}}
}
,
	sources1 = <CFBasicHash 0x608000055ba0 [0x10b29ec70]>{type = mutable set, count = 0,
entries =>
}
,
	observers = (
    "<CFRunLoopObserver 0x60000013afe0 [0x10b29ec70]>{valid = Yes, activities = 0xa0, repeats = Yes, order = 2000000, callout = _ZN2CA11Transaction17observer_callbackEP19__CFRunLoopObservermPv (0x10fbeb012), context = <CFRunLoopObserver context 0x0>}"
),
	timers = (null),
	currently 502435237 (95658932002833) / soft deadline in: 1.84466484e+10 sec (@ -1) / hard deadline in: 1.84466484e+10 sec (@ -1)
},

	6 : <CFRunLoopMode 0x60000018ca50 [0x10b29ec70]>{name = kCFRunLoopCommonModes, port set = 0x5307, queue = 0x60000017db80, source = 0x6000001d86f0 (not fired), timer port = 0x5403, 
	sources0 = (null),
	sources1 = (null),
	observers = (null),
	timers = (null),
	currently 502435237 (95658932144245) / soft deadline in: 1.84466484e+10 sec (@ -1) / hard deadline in: 1.84466484e+10 sec (@ -1)
},

}
}
```

# UI更新


<br/>
参考资料：
1, [官方资料](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/Multithreading/RunLoopManagement/RunLoopManagement.html#//apple_ref/doc/uid/10000057i-CH16-SW1)
2, [走进Runloop的世界](http://chun.tips/blog/2014/10/20/zou-jin-run-loopde-shi-jie-%5B%3F%5D-:shi-yao-shi-run-loop%3F/)
3, [深入理解Runloop](http://blog.ibireme.com/2015/05/18/runloop/)
4, [iOS 保持界面流畅的技巧](http://blog.ibireme.com/2015/11/12/smooth_user_interfaces_for_ios/)
