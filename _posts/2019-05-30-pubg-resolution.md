---
layout:     post
title:      最新版本和平精英发热问题原因及解决办法
subtitle:
date:       2019-05-16 19:00:00
author:     Liao
catalog:    true
header-img: img/in-post/pubg-resolution/pubg.jpg
permalink:  /pubg-resolution/
tags:
    - Game
---

TL;DR 如果你想直接看教程，请直接跳到 [教程](#教程)

# 原因

「刺激战场」更新成「和平精英」之后，很多「鸡友」们都遇到这样一个问题：画面设置为流畅+极限帧率之后，设备发热明显，如果不降温的话，可能会触发手机系统降频导致无法支持极限帧数的渲染，导致画面出现掉帧，极大影响了游戏体验。

究其原因，「刺激战场」在设置为流畅画质时，会降低游戏的分辨率。而到了「和平精英」，无论何种画质，游戏的分辨率始终是和物理设备的分辨率一样的。这就导致了在「和平精英」中，渲染游戏资源会更消耗 CPU/GPU，导致手机发热严重。进一步导致降频，掉帧。

那么原来的「刺激战场」流畅画质的分辨率是多少呢。这里就涉及到两个问题：

1. iOS 设备的像素值和设备像素比
2. 「刺激战场」或「和平精英」所使用的 unreal 渲染引擎

## iOS 设备的设备像素比

从 iPhone4 开始，手持设备进入了 Retina 时代。所谓的 Retina 显示屏，简单理解就是一种高分辨率显示屏。分辨率是指物理设备的像素数量，单位面积内的物理像素越多，那么设备的分辨率越高。

但是如果为了高分辨率一味的增加分辨率，则会带来另外一个问题：分辨率越高，相同像素大小的在屏幕上显示的尺寸就越小。苹果为了解决这个问题，引入了设备像素比的概念（devicePixelRatio）。

所谓设备像素比，即物理像素和显示像素的比率。那么什么是物理像素，什么是显示像素呢。可以简单理解为：

- 物理像素就是设备显示屏能够实际显示的物理显示单元，每一个物理像素可以理解为一个显示点，这个点可以展现不通的颜色（例如RGB）。
- 显示像素是一个逻辑概念，一个显示像素可能由 N 个物理像素组成。

以 iPHone7 为例，它的物理像素为 750px * 1334px，而它的显示像素为 375px * 667px。即物理像素的长宽都是显示像素的一倍，这种屏幕也叫做 2倍 Retina 屏。到了后期的 iPhone 也出现了 3倍屏（例如 iPhone X）。

关于像素比的问题，这里就不赘述了，详细解释可以参考[这里](https://blog.csdn.net/Allenyhy/article/details/81610244)。这里我们知道的是，「和平精英」由于在流畅画质使用了物理分辨率，导致游戏的分辨率相较之前有了成倍增长。这就是为什么很多人感觉「和平精英」种流畅画质下，草更真实了，画面更精细了。

## unreal 引擎配置

「刺激战场」和「和平精英」使用 unreal 引擎开发。参考 unreal 引擎的文档，它有一个 `MobileContentScaleFactor` 的参数，可以控制 iOS 设备在显示时，使用何种像素来显示。

具体解释如下：

- 0.0, 此时会使用设备的真实物理分辨率
- 1.0, 此时使用设备的显示分辨率
- 2.0, 对于 2倍 retina 屏幕，使用其真实物理分辨率（如 iPhone7）
- 3.0, 对于 3倍 retina 屏幕，使用其真实物理分辨率（如 iPhoneX）

参考：[https://docs.unrealengine.com/en-US/Platforms/Mobile/Performance/index.html](https://docs.unrealengine.com/en-US/Platforms/Mobile/Performance/index.html)

那么我们如果可以修改这个配置，略微降低「和平精英」的游戏画质，就能解决设备发热的问题了。

# 教程

对于 Andriod 设备，修改和平精英配置的方法很简单，网上有很多教程，这里就不赘述了。对于 iOS 设备，我们需要先用 [imazing](https://imazing.com/) 导出「和平精英」的用户数据。然后修改其中的用户配置，再将用户数据重新导回 iOS 即可。

imazing 有 Mac/PC 版本，装好之后，可以看这个[视频](https://www.bilibili.com/video/av41648532?from=search&seid=7862747127321908886)，里面有具体的导出和修改流程。

需要注意的是，这个视频里修改的是游戏的帧数设置，用于解锁低配置设备使用极限帧数。而我们需要修改的参数为：

```
[UserCustom DeviceProfile]
...
+CVars=r.MobileContentScaleFactor=1.5
...
```

在你的手机/平板上，这个值可能是 2 或者 3. 这里我使用 iPad 2018 测试时，这个配置的值为 2，将其改成 1.5 之后保存。按照视频种步骤导回 app 数据到 iOS 中后。重启游戏，游戏的分辨率就变成和「刺激战场」里的一样了。再也不会有发热的烦恼了。

> 注意：这里我只测试了 iPad 2018，此设备为 2倍 Retina 屏幕，改成 1.5 实测比较合适。对于 iPhone X，可能需要各种自行进行测试。

最后，祝大家吃鸡愉快 :)
