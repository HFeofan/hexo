---
title: iOS10通知实践
date: 2016-10-11 13:13:29
tags:
- iOS
---

最近项目里在开发iOS10的新特性——UserNotification，看了下[喵神的博看](https://onevcat.com/2016/08/notification/)，变化却似挺大。写一篇自己的总结。

# 授权

    UNUserNotificationCenter.current().requestAuthorization(options: [.sound, .badge, .alert]) { (granted, error) in
        if granted {
            UIApplication.shared.registerForRemoteNotifications()
        } else if let e = error {
            print(e.localizedDescription)
        }
    }

