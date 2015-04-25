---
layout: post
title: "Objective-C 对象模型"
date: 2015-04-25 19:38:02 +0800
comments: true
categories: 
keywords: Objective-C, Object Model, runtime, objc_object, objc_class, id, Class, objc, isa, object, class, metaclass, NSObject, rootclass, 对象模型, 对象, 类, 元类
---

Objective-C 是一门面向对象的程序设计语言，它的对象模型是基于类来建立的。我们可以在苹果开源的 [runtime](http://opensource.apple.com/tarballs/objc4/)（我下载的是当前的最新版本 `objc4-646.tar.gz` ）中发现 Objective-C 对象模型的实现细节。

## 对象

在 Objective-C 中，每一个对象都是某个类的实例，且这个对象的 `isa`（在 64 位 CPU 下，`isa` 已经不再是一个简单的指针，在本文中我们暂且把它当作普通指针来理解，后面我会单独写一篇博文来详细介绍 `Non-pointer isa` ）指针指向它所属的类。

打开刚下载的 runtime 工程，在文件 `objc-private.h` 的第 127-232 行我们可以找到 Objective-C 中的对象的定义 `struct objc_object` 。是的，Objective-C 中的对象本质上是结构体对象，其中 `isa` 是它唯一的私有成员变量。

{% img /images/objc_object.jpg 'Objective-C 对象的定义' 'Objective-C 对象的定义' %}

在同一个文件的第 51-52 行我们可以找到 `Class` 和 `id` 类型的定义，它们分别是 `struct objc_class` 和 `struct objc_object` 类型的指针。这也是为什么 `id` 类型可以指向任意对象的原因。其中 `struct objc_class` 就是 Objective-C 中的类的定义，在下一节将会详细介绍。

``` objc
typedef struct objc_class *Class;
typedef struct objc_object *id;
```

## 类

对象的类不仅描述了对象的数据：对象占用的内存大小、成员变量的类型和布局等，而且也描述了对象的行为：对象能够响应的消息、实现的实例方法等。因此，当我们调用实例方法 `[receiver message]` 给一个对象发送消息时，这个对象能否响应这个消息就需要通过 `isa` 找到它所属的类（当然还有 superclass，本文主要内容不是这个，所以不展开）才能知道。

打开文件 `objc-runtime-new.h` ，在第 687-902 行我们可以找到 Objective-C 中的类的定义 `struct objc_class` 。同样的，Objective-C 中类也是一个结构体对象，并且继承了 `struct objc_object` 。

{% img /images/objc_class.jpg 'Objective-C 类的定义' 'Objective-C 类的定义' %}

所以，Objective-C 中的类本质上也是对象，我们称之为类对象。按照我们前面所说的所有的对象都是某个类的实例，那么类对象又是什么类的实例呢？答案就是我们将在下一节介绍的元类。

在 Objective-C 中有一个非常特殊的类 `NSObject` ，绝大部分的类都继承自它。它是 Objective-C 中的两个根类（rootclass）之一，另外一个是 NSProxy（本文不讨论）。同样的，我们打开文件 `NSObject.h` ，可以看到 `NSObject` 类其实就只有一个成员变量 `isa` ，所有继承自 `NSObject` 的类也都会有这个成员变量。

{% img /images/nsobject.jpg 'NSObject 类' 'NSObject 类' %}

## 元类

我们上面提到，本质上 Objective-C 中的类也是对象，它也是某个类的实例，这个类我们称之为元类（metaclass）。

因此，我们也可以通过调用类方法，比如 `[NSObject new]`，给类对象发送消息。同样的，类对象能否响应这个消息也要通过 `isa` 找到类对象所属的类（元类）才能知道。也就是说，实例方法是保存在类中的，而类方法是保存在元类中的。

那元类也是对象吗？是的话那它又是什么类的实例呢？是的，没错，元类也是对象（元类对象），元类也是某个类的实例，这个类我们称之为根元类（root metaclass）。不过，有一点比较特殊，那就是所有的元类所属的类都是同一个根元类（当然根元类也是元类，所以它所属的类也是根元类，即它本身）。根元类指的就是根类的元类，具体来说就是根类 `NSObject` 对应的元类。

因此，理论上我们也可以给元类发送消息，但是 Objective-C 倾向于隐藏元类，不想让大家知道元类的存在。元类是为了保持 Objective-C 对象模型在设计上的完整性而引入的，比如用来保存类方法等，它主要是用来给编译器使用的。

说了这么多，大家可能已经有点绕迷糊了，下面我们看一张图，一切自会明了。

{% img /images/object_model.png 'Objective-C 对象模型' 'Objective-C 对象模型' %}

## 参考链接

- [http://www.devtang.com/blog/2013/10/15/objective-c-object-model/](http://www.devtang.com/blog/2013/10/15/objective-c-object-model/)
- [http://husbandman.diandian.com/post/2012-08-17/40036035008](http://husbandman.diandian.com/post/2012-08-17/40036035008)
- [http://www.sealiesoftware.com/blog/archive/2009/04/14/objc_explain_Classes_and_metaclasses.html](http://www.sealiesoftware.com/blog/archive/2009/04/14/objc_explain_Classes_and_metaclasses.html)
