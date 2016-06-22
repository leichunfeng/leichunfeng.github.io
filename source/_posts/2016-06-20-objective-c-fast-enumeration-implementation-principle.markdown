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
- 如果你在遍历集合的过程中修改集合的话，它会抛出异常；
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
                                  objects:(id __unsafe_unretained [])stackbuf
                                    count:(NSUInteger)len
```

其中，结构体 `NSFastEnumerationState` 的定义如下：

``` objc
typedef struct {
    unsigned long state;
    id __unsafe_unretained __nullable * __nullable itemsPtr;
    unsigned long * __nullable mutationsPtr;
    unsigned long extra[5];
} NSFastEnumerationState;
```

说实话，刚开始看到这个方法的时候，其实我是拒绝的，因为我觉得苹果根本就是不想让别人实现的节奏。好吧，先不吐槽了，言归正传，下面，我们将对这个方法进行全方位的剖析：

首先，我们需要理解的最重要的一点，那就是这个方法的目的是什么？概括地说，这个方法就是用于返回一系列的 C 数组，以供调用者进行遍历。为什么是一系列的 C 数组呢？因为，事实上，在一个 `for/in` 循环中，这个方法可能会被调用多次，每一次调用都会返回一个 C 数组。那为什么是 C 数组呢？是的，当然是为了提高效率了。

既然要返回 C 数组，也就意味着我们需要返回一个数组的指针和数组的长度。是的，我想你应该已经猜到了，数组长度就是通过这个方法的返回值进行提供的。而数组指针则是通过结构体 `NSFastEnumerationState` 的 `itemsPtr` 字段进行返回的。因此，这两个值就一起定义了这个方法返回的 C 数组。

通常来说，`NSFastEnumeration` 允许我们直接返回一个指向内部存储的指针，但是并非所有的数据结构都能够满足内存连续的要求。因此，`NSFastEnumeration` 为我们提供了另外一种方案，我们可以将对象拷贝到调用者提供的一个 C 数组上，这个数组就是 `stackbuf` ，它的长度由参数 `len` 指定。

在本文的开头，我们提到了如果在遍历集合的过程中修改集合的话，`NSFastEnumeration` 就会抛出异常。事实上，这个功能就是通过 `mutationsPtr` 字段来实现的，它指向一个这样的值，这个值在集合被修改时会发现改变。因此，我们就可以在遍历集合的过程中，使用它来判断集合是否被修改。

截至目前，我们只剩下 `NSFastEnumerationState` 中的 `state` 和 `extra` 字段还没有进行介绍。事实上，它们是调用者提供给被调用者自由使用的两个字段，调用者根本就不关心这两个字段的值，我们可以用它们来存储任何有用的值。

## 内部实现

说了这么多，说得好像 `NSFastEnumeration` 是你设计的一样，你到底是怎么知道的呢？额，我说我是瞎猜的，你信么？好了，不开玩笑了。下面，我们就一起来探究一下快速枚举的内部实现。假设，我们有一个 `main.m` 文件，其中的代码如下：

``` objc
#import <Foundation/Foundation.h>

int main(int argc, char * argv[]) {
    NSArray *array = @[ @1, @2, @3 ];
    for (NSNumber *number in array) {
        if ([number isEqualToNumber:@1]) {
            continue;
        }
        NSLog(@"%@", number);
        break;
    }
}
```

接着，我们使用下面的 clang 命令将 `main.m` 文件重写成 C++ 代码：

```
clang -rewrite-objc main.m
```

得到 `main.cpp` 文件，其中 `main` 函数的代码如下：

``` objc
int main(int argc, char * argv[]) {
    // 创建数组 @[ @1, @2, @3 ]
    NSArray *array = ((NSArray *(*)(Class, SEL, const ObjectType *, NSUInteger))(void *)objc_msgSend)(objc_getClass("NSArray"), sel_registerName("arrayWithObjects:count:"), (const id *)__NSContainer_literal(3U, ((NSNumber *(*)(Class, SEL, int))(void *)objc_msgSend)(objc_getClass("NSNumber"), sel_registerName("numberWithInt:"), 1), ((NSNumber *(*)(Class, SEL, int))(void *)objc_msgSend)(objc_getClass("NSNumber"), sel_registerName("numberWithInt:"), 2), ((NSNumber *(*)(Class, SEL, int))(void *)objc_msgSend)(objc_getClass("NSNumber"), sel_registerName("numberWithInt:"), 3)).arr, 3U);
    
    {
        NSNumber * number;
        
        // 初始化结构体 NSFastEnumerationState
        struct __objcFastEnumerationState enumState = { 0 };

        // 初始化数组 stackbuf
        id __rw_items[16];

        id l_collection = (id) array;

        // 第一次调用 - countByEnumeratingWithState:objects:count: 方法，形参和实参的对应关系如下：
        // state -> &enumState
        // stackbuf -> __rw_items
        // len -> 16
        _WIN_NSUInteger limit =
            ((_WIN_NSUInteger (*) (id, SEL, struct __objcFastEnumerationState *, id *, _WIN_NSUInteger))(void *)objc_msgSend)
            ((id)l_collection,
            sel_registerName("countByEnumeratingWithState:objects:count:"),
            &enumState, (id *)__rw_items, (_WIN_NSUInteger)16);
        
        if (limit) {
            // 获取 mutationsPtr 的初始值
            unsigned long startMutations = *enumState.mutationsPtr;
            
            // 外层的 do/while 循环，用于循环调用 - countByEnumeratingWithState:objects:count: 方法，获取 C 数组
            do {
                unsigned long counter = 0;

                // 内层的 do/while 循环，用于遍历获取到的 C 数组
                do {
                    // 判断 mutationsPtr 的值是否有发生变化，如果有则使用 objc_enumerationMutation 抛出异常
                    if (startMutations != *enumState.mutationsPtr) objc_enumerationMutation(l_collection);
                    
                    // 使用指针的算术运算获取相应的集合元素，这是快速枚举之所以高效的关键所在
                    number = (NSNumber *)enumState.itemsPtr[counter++]; 
                    
                    {
                        if (((BOOL (*)(id, SEL, NSNumber *))(void *)objc_msgSend)((id)number, sel_registerName("isEqualToNumber:"), ((NSNumber *(*)(Class, SEL, int))(void *)objc_msgSend)(objc_getClass("NSNumber"), sel_registerName("numberWithInt:"), 1))) {
                            // continue 语句的实现，使用 goto 语句无条件转移到内层 do 语句的末尾，跳过中间的所有代码
                            goto __continue_label_1;
                        }
                       
                        NSLog((NSString *)&__NSConstantStringImpl__var_folders_cr_xxw2w3rd5_n493ggz9_l4bcw0000gn_T_main_fc7b79_mi_0, number);
                       
                        // break 语句的实现，使用 goto 语句无条件转移到最外层 if 语句的末尾，跳出嵌套的两层循环
                        goto __break_label_1;
                    };

                    // goto 语句标号，用来实现 continue 语句
                    __continue_label_1: ;
                } while (counter < limit);
            } while ((limit = 
                ((_WIN_NSUInteger (*) (id, SEL, struct __objcFastEnumerationState *, id *, _WIN_NSUInteger))(void *)objc_msgSend)
                ((id)l_collection,
                sel_registerName("countByEnumeratingWithState:objects:count:"),
                &enumState, (id *)__rw_items, (_WIN_NSUInteger)16)));

            number = ((NSNumber *)0);

            // goto 语句标号，用来实现 break 语句
            __break_label_1: ;
        } else {
            number = ((NSNumber *)0);
        }
    }
}
```

如上代码所示，快速枚举其实是用两层 `do/while` 循环来实现的，外层循环负责调用 `- countByEnumeratingWithState:objects:count:` 方法，获取 C 数组，而内层循环则负责遍历获取到的 C 数组。同时，我想你也应该注意到了它是如何利用 `mutationsPtr` 来检测集合在遍历过程中的突变的，以及怎样使用 [objc_enumerationMutation](https://developer.apple.com/reference/objectivec/1418744-objc_enumerationmutation?language=objc) 函数来抛出异常。

正如我们前面提到的，在快速枚举的实现中，的确没有用到结构体 `NSFastEnumerationState` 中的 `state` 和 `extra` 字段，它们只是提供给 `- countByEnumeratingWithState:objects:count:` 自由使用的字段。

另外，值得一提的是，我特意在 `main.m` 中加入了 `continue` 和 `break` 语句。因此，我们也有机会看到了在 `for/in` 语句中是如何使用 goto 语句来实现 `continue` 和 `break` 语句的。

## 实现 NSFastEnumeration

看到这里，我相信你对 Objective-C 中的快速枚举的实现原理已经有了一个比较清晰地认识。接下来，我们就一起来动手实现一下 `NSFastEnumeration` 。

正如我们前面所说的，`NSFastEnumeration` 在设计上允许我们使用两种不同的方式来实现它。如果集合中的元素是内存连续的，那么我们可以直接返回这段内存的首地址；而如果内存不连续，比如链表，就只能使用调用者提供的 C 数组 `stackbuf` 了，将我们的元素拷贝到这个 C 数组上。

下面，我们将自定义一个集合类 `Array` ，演示如何通过以上两种不同的方式来实现 `NSFastEnumeration` 。**注**：这个工程的代码可以在这里找到。

``` objc
@interface Array : NSObject <NSFastEnumeration>

- (instancetype)initWithCapacity:(NSUInteger)numItems;

@end

@implementation Array {
    std::vector<NSNumber *> _list;
}

- (instancetype)initWithCapacity:(NSUInteger)numItems {
    self = [super init];
    if (self) {
        for (NSUInteger i = 0; i < numItems; i++) {
            _list.push_back(@(random()));
        }
    }
    return self;
}

#define USE_STACKBUF 1

- (NSUInteger)countByEnumeratingWithState:(NSFastEnumerationState *)state objects:(id __unsafe_unretained [])stackbuf count:(NSUInteger)len {
    // 这个方法的返回值，即我们需要返回的 C 数组的长度
    NSUInteger count = 0;
    
    // 我们前面已经知道了，这个方法是会被多次调用的
    // 因此，我们需要使用 state->state 来保存当前遍历到了 _list 的什么位置
    unsigned long countOfItemsAlreadyEnumerated = state->state;
    
    // 当 countOfItemsAlreadyEnumerated 为 0 时，表示第一次调用这个方法
    // 我们可以在这里做一些初始化的设置
    if (countOfItemsAlreadyEnumerated == 0) {
        // 我们前面已经提到了，state->mutationsPtr 是用来追踪集合在遍历过程中的突变的
        // 它不能为 NULL ，并且也不应该指向 self
        //
        // 这里，因为我们的 Array 类是不可变的，所以我们不需要追踪它的突变
        // 因此，我们的做法是将它指向 state->extra 的其中一个值
        // 因为我们知道 NSFastEnumeration 协议本身并没有用到 state->extra
        //
        // 但是，如果你的集合是可变的，那么你可以考虑将 state->mutationsPtr 指向一个内部变量
        // 而这个内部变量的值会在你的集合突变时发生变化
        state->mutationsPtr = &state->extra[0];
    }
    
#if USE_STACKBUF
    
    // 判断我们是否已经遍历完 _list
    if (countOfItemsAlreadyEnumerated < _list.size()) {
        // 我们知道 state->itemsPtr 就是这个方法返回的 C 数组指针，它不能为 NULL
        // 这里，我们将 state->itemsPtr 指向调用者提供的 C 数组 stackbuf
        state->itemsPtr = stackbuf;
        
        // 将 _list 中的值填充到 stackbuf 中，直到以下两个条件中的任意一个满足时为止
        // 1. 已经遍历完 _list 中的元素
        // 2. 已经填充满 stackbuf
        while (countOfItemsAlreadyEnumerated < _list.size() && count < len) {
            // 取出 _list 中的元素填充到 stackbuf 中
            stackbuf[count] = _list[countOfItemsAlreadyEnumerated];
            
            // 更新我们的遍历位置
            countOfItemsAlreadyEnumerated++;
            
            // 更新我们返回的 C 数组的长度，使之与 state->itemsPtr 中的元素个数相匹配
            count++;
        }
    }
    
#else
    
    // 判断我们是否已经遍历完 _list
    if (countOfItemsAlreadyEnumerated < _list.size()) {
        // 直接将 state->itemsPtr 指向内部的 C 数组指针，因为它的内存地址是连续的
        __unsafe_unretained const id * const_array = _list.data();
        state->itemsPtr = (__typeof__(state->itemsPtr))const_array;
        
        // 因为我们一次性返回了 _list 中的所有元素
        // 所以，countOfItemsAlreadyEnumerated 和 count 的值均为 _list 中的元素个数
        countOfItemsAlreadyEnumerated = _list.size();
        count = _list.size();
    }
    
#endif
    
    // 将本次调用得到的 countOfItemsAlreadyEnumerated 保存到 state->state 中
    // 因为 NSFastEnumeration 协议本身并没有用到 state->state
    // 所以，我们可以将这个值一直保留到下一次调用
    state->state = countOfItemsAlreadyEnumerated;
    
    // 返回 C 数组的长度
    return count;
}

@end
```

我已经在上面的代码中添加了必要的注释，相信你理解起来应该没有什么难度。不过，值得一提的是，在第二种方式的实现中，用到了 ARC 下不同所有权对象之间的相互转换，代码如下：

``` objc
__unsafe_unretained const id * const_array = _list.data();
state->itemsPtr = (__typeof__(state->itemsPtr))const_array;
```

其实，这里面有两次类型转换，第一次是从 `__strong NSNumber *` 类型转换到 `__unsafe_unretained const id *` 类型，第二次是从 `__unsafe_unretained const id *` 类型转换到 `id __unsafe_unretained *` 类型，更多信息可以查看 [AutomaticReferenceCounting](http://clang.llvm.org/docs/AutomaticReferenceCounting.html) 中的 4.3.3 小节。

## 总结

本文从 `NSFastEnumeration` 的定义出发，解释了 `- countByEnumeratingWithState:objects:count:` 方法中的返回值以及各个参数的含义；接着，使用 `clang -rewrite-objc` 命令探究了快速枚举的内部实现细节；最后，通过一个自定义的集合类 `Array` 演示了两种实现 `NSFastEnumeration` 的方式。

## 参考链接

[https://www.mikeash.com/pyblog/friday-qa-2010-04-16-implementing-fast-enumeration.html](https://www.mikeash.com/pyblog/friday-qa-2010-04-16-implementing-fast-enumeration.html)
[https://zh.wikipedia.org/wiki/Objective-C#.E5.BF.AB.E9.80.9F.E6.9E.9A.E4.B8.BE](https://zh.wikipedia.org/wiki/Objective-C#.E5.BF.AB.E9.80.9F.E6.9E.9A.E4.B8.BE)
[https://developer.apple.com/library/mac/documentation/Cocoa/Conceptual/Collections/Articles/Enumerators.html#//apple_ref/doc/uid/20000135-SW1](https://developer.apple.com/library/mac/documentation/Cocoa/Conceptual/Collections/Articles/Enumerators.html#//apple_ref/doc/uid/20000135-SW1)

<img src="http://blog.leichunfeng.com/images/wechat_pay.jpg" width="260" />
