---
layout: post
title: "Objective-C Fast Enumeration 的实现原理"
date: 2016-06-20 20:30:53 +0800
comments: true
categories: 
---

在 Objective-C 2.0 中提供了快速枚举的语法，它是我们遍历集合元素的首选方法，因为它具有以下优点：

- 比直接使用 `NSEnumerator` 更高效；
- 语法非常简洁；
- 如果你在遍历的过程中修改集合的话，它会抛出异常；
- 你可以同时执行多个枚举。

那么，问题来了，它是怎么做到的呢？我想，你应该也跟我一样，对 Objective-C 中快速枚举的底层实现非常感兴趣，事不宜迟，让我们来一探究竟吧。

## NSFastEnumeration

在 Objective-C 中，我们要实现快速枚举就必须要实现 [NSFastEnumeration](https://developer.apple.com/library/mac/#documentation/Cocoa/Reference/NSFastEnumeration_protocol/Reference/NSFastEnumeration.html) 协议，在这个协议中，只声明了一个必须实现的方法：

``` objc
/**
 Returns by reference a C array of objects over which the sender should iterate, and as the return value the number of objects in the array.

 @param state  Context information that is used in the enumeration to, in addition to other possibilities, ensure that the collection has not been mutated.
 @param buffer A C array of objects over which the sender is to iterate.
 @param len    The maximum number of objects to return in stackbuf.
 
 @discussion The state structure is assumed to be of stack local memory, so you can recast the passed in state structure to one more suitable for your iteration.

 @return The number of objects returned in stackbuf. Returns 0 when the iteration is finished.
 */
- (NSUInteger)countByEnumeratingWithState:(NSFastEnumerationState *)state
                                  objects:(id  _Nullable __unsafe_unretained [])buffer
                                    count:(NSUInteger)len
```

其中，结构体 `NSFastEnumerationState` 的定义如下：

``` objc
typedef struct {
    unsigned long state;
    id __unsafe_unretained _Nullable * _Nullable itemsPtr;
    unsigned long * _Nullable mutationsPtr;
    unsigned long extra[5];
} NSFastEnumerationState;
```

说实话，刚开始看到这个方法的时候，其实我是拒绝的，因为我觉得苹果根本就是不想让别人实现的节奏。好吧，先不吐槽了，言归正传，下面，我们将对这个方法进行全方位的剖析：

首先，我们需要理解的最重要的一点，那就是这个方法的目的是什么？概括地说，这个方法就是用于返回一系列的 C 数组，以供调用者进行遍历。为什么是一系列的 C 数组呢？因为，事实上，在一个 `for/in` 循环中，这个方法可能会被调用多次，每一次调用都会返回一个 C 数组。那为什么是 C 数组呢？是的，当然是为了提高效率了。

既然，要返回 C 数组，也就意味着我们需要返回一个数组指针 pointer 和数组长度 length 。是的，我想你已经猜到了，数组长度 length 就是通过这个方法的返回值进行提供的。而数组指针 pointer 则是通过结构体 `NSFastEnumerationState` 的 `itemsPtr` 字段进行返回的。因此，这两个值就一起定义了这个方法返回的 C 数组。

## 内部实现

## 实现 NSFastEnumeration

### 使用 stackbuf

### 使用自己的对象数组


