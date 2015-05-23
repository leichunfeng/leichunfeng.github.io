---
layout: post
title: "结合 Reveal 谈谈 MBProgressHUD 的用法"
date: 2015-03-16 12:32:51 +0800
comments: true
categories: 
keywords: Reveal, MBProgressHUD, self.view.window, self.navigationController.view, self.navigationController.navigationBar, self.view
---

我的博客已经好几个月没有更新了，想想还真是有些难过。究其原因主要是工作占用了我绝大部分的时间，包含工作日的晚上和大部分周末。剩余的很小部分空余时间，我都用来开发个人 iOS 应用 GitBucket 了。

GitBucket 是一个 GitHub 的 iOS 客户端，使用 [MVVM](http://en.wikipedia.org/wiki/Model_View_ViewModel) 模式和 [RAC](https://github.com/ReactiveCocoa/ReactiveCocoa) 框架进行开发。目前 GitBucket 的项目代码 [MVVMReactiveCocoa](https://github.com/leichunfeng/MVVMReactiveCocoa) 已经在 GitHub 上开源了出来。开发这个应用的主要目的是希望提供一个使用 MVVM 模式和 RAC 框架开发的完整应用，能够对学习 MVVM 模式和 RAC 框架的 iOS 开发者有所帮助。昨天，我已经向 App Store 提交了 GitBucket v1.0 版本，相信很快大家就可以下载使用了。

言归正传，接下来我们结合 [Reveal](http://revealapp.com/) 来谈谈 [MBProgressHUD](https://github.com/jdg/MBProgressHUD) 的用法。这里主要讨论的是什么情况下导航栏上的按钮可用，什么情况下不可用，及其原因。

**注**：本博文的程序代码可以在 [leichunfeng/MBProgressHUD](https://github.com/leichunfeng/MBProgressHUD) 上找到，读者可以克隆下来亲自运行下看看效果。

## 关于 self.navigationController.view

相信看过 MBProgressHUD 官方例子 `HudDemo` 代码的同学应该看到过下述代码：

``` objc
HUD = [[MBProgressHUD alloc] initWithView:self.navigationController.view];
```

当时，你可能会对 `self.navigationController.view` 有些疑惑，这是什么玩意？其实，如果我们查看下 `UINavigationController.h` 文件就会发现，`UINavigationController` 其实是继承自 `UIViewController` 的，那么它拥有 `view` 属性也就不奇怪了。

``` objc
NS_CLASS_AVAILABLE_IOS(2_0) @interface UINavigationController : UIViewController
```

下面，我们会结合 Reveal 清楚地看到 `self.navigationController.view` 到底是什么东西，稍安勿躁。

## 显示 MBProgressHUD

初始化 MBProgressHUD 时需要我们传入一个 `UIView` 类型的参数 `view`，而显示 MBProgressHUD 的原理其实就是用 `addSubview` 方法将 MBProgressHUD 添加为这个 `view` 的子视图。

我们先来看看未显示 MBProgressHUD 时，应用的视图层次结构。其中 1 为 `UIWindow` ，即 `self.view.window`，2 是 `UINavigationController` 的 `view` ，即我们前面提到的 `self.navigationController.view` ，3 为 `self.view` ，4 为导航栏 `UINavigationBar` ，即 `self.navigationController.navigationBar` 。

{% img /images/show-mbprogresshud.jpg '应用的视图层次结构' '应用的视图层次结构' %}

通过这张图，我们清楚地看到了 `self.view.window` 、`self.navigationController.view` 、`self.view` 和 `self.navigationController.navigationBar` 在应用的视图层次中所处的位置，以及它们之间的层次关系。

下面，我们就对比一下 MBProgressHUD 分别在 `self.view.window` 、`self.navigationController.view` 和 `self.view` 上显示时应用的视图层次结构，以及导航栏上按钮的可用情况。

### 方式 1 - On self.view.window

使用这种方式时，MBProgressHUD 被添加到了 `self.view.window` 上，它与 `self.navigationController.view` 在视图层次上是平级的，同为 `self.view.window` 的子视图。但是由于 MBProgressHUD 是后添加的，所以它处于 `self.navigationController.view` 的上方，因此导航栏上的按钮均不可点击。

{% img /images/on-self.view.window.jpg '应用的视图层次结构' '应用的视图层次结构' %}

### 方式 2 - On self.navigationController.view

使用这种方式时，MBProgressHUD 被添加到了 `self.navigationController.view` 上，它与 `self.navigationController.navigationBar` 在视图层次上是平级的，同为 `self.navigationController.view` 的子视图。但是由于 MBProgressHUD 是后添加的，所以它处于 `self.navigationController.navigationBar` 的上方，因此导航栏上的按钮也均不可点击。

{% img /images/on-self.navigationController.view.jpg '应用的视图层次结构' '应用的视图层次结构' %}

### 方式 3 - On self.view

使用这种方式时，MBProgressHUD 被添加到了 `self.view` 上，不管 `self.view` 或 MBProgressHUD 是否占满整个屏幕，`self.navigationController.navigationBar` 永远处于 MBProgressHUD 的上方。因此，导航栏上的按钮一直是可点击的。

{% img /images/on-self.view.jpg '应用的视图层次结构' '应用的视图层次结构' %}

## 总结

当你需要让导航栏上的按钮不可点击的时候，可以选择使用 `方式 1` 或 `方式 2` 显示 MBProgressHUD 。反之，可以选择 `方式 3` 。
