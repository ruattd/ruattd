---
title: macOS 禁用烦人更新提示
date: 2026-01-19 17:59:44
link: life/trick/macos/disable-update-notification
categories:
  - [随笔, 奇淫巧技]
tags:
  - macOS
  - Apple
  - 更新
---

前两天 macOS 又在给我推送那个烦人的液态玻璃更新了，苹果你对你造的破烂 M1 是真自信啊。

遂寻如何禁用更新，得此法：

```sh
defaults write com.apple.SoftwareUpdate MajorOSUserNotificationDate -date "2030-02-07 23:22:47 +0000"
```

这样 2030 年以前它就不会拿更新烦你了。

什么你问 2030 年以后怎么办？到那时候 M1 还能不能用都不知道，管他呢。
