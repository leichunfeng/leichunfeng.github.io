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

既然，要返回 C 数组，也就意味着我们需要返回一个数组指针 pointer 和数组长度 length 。是的，我想你应该已经猜到了，数组长度 length 就是通过这个方法的返回值进行提供的。而数组指针 pointer 则是通过结构体 `NSFastEnumerationState` 的 `itemsPtr` 字段进行返回的。因此，这两个值就一起定义了这个方法返回的 C 数组。

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

正如我们前面提到的，在快速枚举的实现中，的确没有用到结构体 `NSFastEnumerationState` 中的 `state` 和 `extra` 字段，它们只是提供给 `- countByEnumeratingWithState:objects:count:` 自由使用的。

另外，值得一提的是，我特意在 `main.m` 中加入了 `continue` 和 `break` 语句。因此，我们也有幸看到了在 `for/in` 语句中是如何使用 goto 语句来实现 `continue` 和 `break` 语句的。

## 实现 NSFastEnumeration

### 使用 stackbuf

### 使用自己的对象数组


