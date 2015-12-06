---
layout: post
title: "Objective-C Autorelease Pool 的实现原理"
date: 2015-05-31 10:46:34 +0800
comments: true
categories: 
keywords: release, autorelease, autoreleasepool, AutoreleasePoolPage, NSAutoreleasePool, 内存管理
---

内存管理一直是学习 Objective-C 的重点和难点之一，尽管现在已经是 ARC 时代了，但是了解 Objective-C 的内存管理机制仍然是十分必要的。其中，弄清楚 autorelease 的原理更是重中之重，只有理解了 autorelease 的原理，我们才算是真正了解了 Objective-C 的内存管理机制。**注**：本文使用的 [runtime](http://opensource.apple.com/tarballs/objc4/) 源码是当前的最新版本 `objc4-646.tar.gz` 。

## autoreleased 对象什么时候释放

autorelease 本质上就是延迟调用 release ，那 autoreleased 对象究竟会在什么时候释放呢？为了弄清楚这个问题，我们先来做一个小实验。这个小实验分 3 种场景进行，请你先自行思考在每种场景下的 console 输出，以加深理解。**注**：本实验的源码可以在这里 [AutoreleasePool](https://github.com/leichunfeng/AutoreleasePool) 找到。

**特别说明**：在苹果一些新的硬件设备上，本实验的结果已经不再成立，详细情况如下：

- iPad 2
- ~~iPad Air~~
- ~~iPad Air 2~~
- ~~iPad Pro~~
- iPad Retina
- iPhone 4s
- iPhone 5
- ~~iPhone 5s~~
- ~~iPhone 6~~
- ~~iPhone 6 Plus~~
- ~~iPhone 6s~~
- ~~iPhone 6s Plus~~

``` objc
__weak NSString *string_weak_ = nil;

- (void)viewDidLoad {
    [super viewDidLoad];

    // 场景 1
    NSString *string = [NSString stringWithFormat:@"leichunfeng"];
    string_weak_ = string;
    
    // 场景 2
//    @autoreleasepool {
//        NSString *string = [NSString stringWithFormat:@"leichunfeng"];
//        string_weak_ = string;
//    }
    
    // 场景 3
//    NSString *string = nil;
//    @autoreleasepool {
//        string = [NSString stringWithFormat:@"leichunfeng"];
//        string_weak_ = string;
//    }
    
    NSLog(@"string: %@", string_weak_);
}

- (void)viewWillAppear:(BOOL)animated {
    [super viewWillAppear:animated];
    NSLog(@"string: %@", string_weak_);
}

- (void)viewDidAppear:(BOOL)animated {
    [super viewDidAppear:animated];
    NSLog(@"string: %@", string_weak_);
}
```

思考得怎么样了？相信在你心中已经有答案了。那么让我们一起来看看 console 输出：

``` objc
// 场景 1
2015-05-30 10:32:20.837 AutoreleasePool[33876:1448343] string: leichunfeng
2015-05-30 10:32:20.838 AutoreleasePool[33876:1448343] string: leichunfeng
2015-05-30 10:32:20.845 AutoreleasePool[33876:1448343] string: (null)

// 场景 2
2015-05-30 10:32:50.548 AutoreleasePool[33915:1448912] string: (null)
2015-05-30 10:32:50.549 AutoreleasePool[33915:1448912] string: (null)
2015-05-30 10:32:50.555 AutoreleasePool[33915:1448912] string: (null)

// 场景 3
2015-05-30 10:33:07.075 AutoreleasePool[33984:1449418] string: leichunfeng
2015-05-30 10:33:07.075 AutoreleasePool[33984:1449418] string: (null)
2015-05-30 10:33:07.094 AutoreleasePool[33984:1449418] string: (null)
```

跟你预想的结果有出入吗？Any way ，我们一起来分析下为什么会得到这样的结果。

**分析**：3 种场景下，我们都通过 `[NSString stringWithFormat:@"leichunfeng"]` 创建了一个 autoreleased 对象，这是我们实验的前提。并且，为了能够在 `viewWillAppear` 和 `viewDidAppear` 中继续访问这个对象，我们使用了一个全局的 `__weak` 变量 `string_weak_` 来指向它。因为 `__weak` 变量有一个特性就是它不会影响所指向对象的生命周期，这里我们正是利用了这个特性。

**场景 1**：当使用 `[NSString stringWithFormat:@"leichunfeng"]` 创建一个对象时，这个对象的引用计数为 1 ，并且这个对象被系统自动添加到了当前的 autoreleasepool 中。当使用局部变量 `string` 指向这个对象时，这个对象的引用计数 +1 ，变成了 2 。因为在 ARC 下 `NSString *string` 本质上就是 `__strong NSString *string` 。所以在 `viewDidLoad` 方法返回前，这个对象是一直存在的，且引用计数为 2 。而当 `viewDidLoad` 方法返回时，局部变量 `string` 被回收，指向了 `nil` 。因此，其所指向对象的引用计数 -1 ，变成了 1 。

而在 `viewWillAppear` 方法中，我们仍然可以打印出这个对象的值，说明这个对象并没有被释放。咦，这不科学吧？我读书少，你表骗我。不是一直都说当函数返回的时候，函数内部产生的对象就会被释放的吗？如果你这样想的话，那我只能说：骚年你太年经了。开个玩笑，我们继续。前面我们提到了，这个对象是一个 autoreleased 对象，autoreleased 对象是被添加到了当前最近的 autoreleasepool 中的，只有当这个 autoreleasepool 自身 drain 的时候，autoreleasepool 中的 autoreleased 对象才会被 release 。

另外，我们注意到当在 `viewDidAppear` 中再打印这个对象的时候，对象的值变成了 `nil` ，说明此时对象已经被释放了。因此，我们可以大胆地猜测一下，这个对象一定是在 `viewWillAppear` 和 `viewDidAppear` 方法之间的某个时候被释放了，并且是由于它所在的 autoreleasepool 被 drain 的时候释放的。

你说什么就是什么咯？有本事你就证明给我看你妈是你妈。额，这个我真证明不了，不过上面的猜测我还是可以证明的，不信，你看！

在开始前，我先简单地说明一下原理，我们可以通过使用 `lldb` 的 `watchpoint` 命令来设置观察点，观察全局变量 `string_weak_` 的值的变化，`string_weak_` 变量保存的就是我们创建的 autoreleased 对象的地址。在这里，我们再次利用了 `__weak` 变量的另外一个特性，就是当它所指向的对象被释放时，`__weak` 变量的值会被置为 `nil` 。了解了基本原理后，我们开始验证上面的猜测。

我们先在第 35 行打一个断点，当程序运行到这个断点时，我们通过 `lldb` 命令 `watchpoint set v string_weak_` 设置观察点，观察 `string_weak_` 变量的值的变化。如下图所示，我们将在 console 中看到类似的输出，说明我们已经成功地设置了一个观察点：

![设置观察点](http://blog.leichunfeng.com/images/watchpoint1.jpg "设置观察点")

设置好观察点后，点击 `Continue program execution` 按钮，继续运行程序，我们将看到如下图所示的界面：

![设置观察点](http://blog.leichunfeng.com/images/watchpoint2.jpg "设置观察点")

我们先看 console 中的输出，注意到 `string_weak_` 变量的值由 `0x00007f9b886567d0` 变成了 `0x0000000000000000` ，也就是 `nil` 。说明此时它所指向的对象被释放了。另外，我们也可以注意到一个细节，那就是 console 中打印了两次对象的值，说明此时 `viewWillAppear` 也已经被调用了，而 `viewDidAppear` 还没有被调用。

接着，我们来看看左侧的线程堆栈。我们看到了一个非常敏感的方法调用 `-[NSAutoreleasePool release]` ，这个方法最终通过调用 `AutoreleasePoolPage::pop(void *)` 函数来负责对 autoreleasepool 中的 autoreleased 对象执行 release 操作。结合前面的分析，我们知道在 `viewDidLoad` 中创建的 autoreleased 对象在方法返回后引用计数为 1 ，所以经过这里的 release 操作后，这个对象的引用计数 -1 ，变成了 0 ，该 autoreleased 对象最终被释放，猜测得证。

另外，值得一提的是，我们在代码中并没有手动添加 autoreleasepool ，那这个 autoreleasepool 究竟是哪里来的呢？看完后面的章节你就明白了。

**场景 2**：同理，当通过 `[NSString stringWithFormat:@"leichunfeng"]` 创建一个对象时，这个对象的引用计数为 1 。而当使用局部变量 `string` 指向这个对象时，这个对象的引用计数 +1 ，变成了 2 。而出了当前作用域时，局部变量 `string` 变成了 `nil` ，所以其所指向对象的引用计数变成 1 。另外，我们知道当出了 `@autoreleasepool {}` 的作用域时，当前 autoreleasepool 被 drain ，其中的 autoreleased 对象被 release 。所以这个对象的引用计数变成了 0 ，对象最终被释放。

**场景 3**：同理，当出了 `@autoreleasepool {}` 的作用域时，其中的 autoreleased 对象被 release ，对象的引用计数变成 1 。当出了局部变量 `string` 的作用域，即 `viewDidLoad` 方法返回时，`string` 指向了 `nil` ，其所指向对象的引用计数变成 0 ，对象最终被释放。

理解在这 3 种场景下，autoreleased 对象什么时候释放对我们理解 Objective-C 的内存管理机制非常有帮助。其中，场景 1 出现得最多，就是不需要我们手动添加 `@autoreleasepool {}` 的情况，直接使用系统维护的 autoreleasepool ；场景 2 就是需要我们手动添加 `@autoreleasepool {}` 的情况，手动干预 autoreleased 对象的释放时机；场景 3 是为了区别场景 2 而引入的，在这种场景下并不能达到出了 `@autoreleasepool {}` 的作用域时 autoreleased 对象被释放的目的。

**PS**：请读者参考场景 1 的分析过程，使用 `lldb` 命令 `watchpoint` 自行验证下在场景 2 和场景 3 下 autoreleased 对象的释放时机，you should give it a try yourself 。

## AutoreleasePoolPage

细心的读者应该已经有所察觉，我们在上面已经提到了 `-[NSAutoreleasePool release]` 方法最终是通过调用 `AutoreleasePoolPage::pop(void *)` 函数来负责对 autoreleasepool 中的 autoreleased 对象执行 release 操作的。

那这里的 AutoreleasePoolPage 是什么东西呢？其实，autoreleasepool 是没有单独的内存结构的，它是通过以 AutoreleasePoolPage 为结点的**双向链表**来实现的。我们打开 runtime 的源码工程，在 `NSObject.mm` 文件的第 438-932 行可以找到 autoreleasepool 的实现源码。通过阅读源码，我们可以知道：

- 每一个线程的 autoreleasepool 其实就是一个指针的堆栈；
- 每一个指针代表一个需要 release 的对象或者 POOL_SENTINEL（哨兵对象，代表一个 autoreleasepool 的边界）；
- 一个 pool token 就是这个 pool 所对应的 POOL_SENTINEL 的内存地址。当这个 pool 被 pop 的时候，所有内存地址在 pool token 之后的对象都会被 release ；
- 这个堆栈被划分成了一个以 page 为结点的双向链表。pages 会在必要的时候动态地增加或删除；
- Thread-local storage（线程局部存储）指向 hot page ，即最新添加的 autoreleased 对象所在的那个 page 。

一个空的 AutoreleasePoolPage 的内存结构如下图所示：

![AutoreleasePoolPage](http://blog.leichunfeng.com/images/AutoreleasePoolPage.png "AutoreleasePoolPage")

1. `magic` 用来校验 AutoreleasePoolPage 的结构是否完整；
2. `next` 指向最新添加的 autoreleased 对象的下一个位置，初始化时指向 `begin()` ；
3. `thread` 指向当前线程；
4. `parent` 指向父结点，第一个结点的 parent 值为 `nil` ；
5. `child` 指向子结点，最后一个结点的 child 值为 `nil` ；
6. `depth` 代表深度，从 0 开始，往后递增 1；
7. `hiwat` 代表 high water mark 。 

另外，当 `next == begin()` 时，表示 AutoreleasePoolPage 为空；当 `next == end()` 时，表示 AutoreleasePoolPage 已满。

### Autorelease Pool Blocks

我们使用 `clang -rewrite-objc` 命令将下面的 Objective-C 代码重写成 C++ 代码： 

``` objc
@autoreleasepool {

}
```

将会得到以下输出结果（只保留了相关代码）：

``` objc
extern "C" __declspec(dllimport) void * objc_autoreleasePoolPush(void);
extern "C" __declspec(dllimport) void objc_autoreleasePoolPop(void *);

struct __AtAutoreleasePool {
  __AtAutoreleasePool() {atautoreleasepoolobj = objc_autoreleasePoolPush();}
  ~__AtAutoreleasePool() {objc_autoreleasePoolPop(atautoreleasepoolobj);}
  void * atautoreleasepoolobj;
};

/* @autoreleasepool */ { __AtAutoreleasePool __autoreleasepool; 

}
```

不得不说，苹果对 `@autoreleasepool {}` 的实现真的是非常巧妙，真正可以称得上是代码的艺术。苹果通过声明一个 `__AtAutoreleasePool` 类型的局部变量 `__autoreleasepool` 来实现 `@autoreleasepool {}` 。当声明 `__autoreleasepool` 变量时，构造函数 `__AtAutoreleasePool()` 被调用，即执行 `atautoreleasepoolobj = objc_autoreleasePoolPush();` ；当出了当前作用域时，析构函数 `~__AtAutoreleasePool()` 被调用，即执行 `objc_autoreleasePoolPop(atautoreleasepoolobj);` 。也就是说 `@autoreleasepool {}` 的实现代码可以进一步简化如下：

``` objc
/* @autoreleasepool */ { 
    void *atautoreleasepoolobj = objc_autoreleasePoolPush(); 
    // 用户代码，所有接收到 autorelease 消息的对象会被添加到这个 autoreleasepool 中
    objc_autoreleasePoolPop(atautoreleasepoolobj);
}
```

因此，单个 autoreleasepool 的运行过程可以简单地理解为 `objc_autoreleasePoolPush()`、`[对象 autorelease]` 和 `objc_autoreleasePoolPop(void *)` 三个过程。

### push 操作

上面提到的 `objc_autoreleasePoolPush()` 函数本质上就是调用的 AutoreleasePoolPage 的 push 函数。

``` objc
void *
objc_autoreleasePoolPush(void)
{
    if (UseGC) return nil;
    return AutoreleasePoolPage::push();
}
```

因此，我们接下来看看 AutoreleasePoolPage 的 push 函数的作用和执行过程。一个 push 操作其实就是创建一个新的 autoreleasepool ，对应 AutoreleasePoolPage 的具体实现就是往 AutoreleasePoolPage 中的 `next` 位置插入一个 POOL_SENTINEL ，并且返回插入的 POOL_SENTINEL 的内存地址。这个地址也就是我们前面提到的 pool token ，在执行 pop 操作的时候作为函数的入参。

```
static inline void *push() 
{
    id *dest = autoreleaseFast(POOL_SENTINEL);
    assert(*dest == POOL_SENTINEL);
    return dest;
}
```

push 函数通过调用 `autoreleaseFast` 函数来执行具体的插入操作。

```
static inline id *autoreleaseFast(id obj)
{
    AutoreleasePoolPage *page = hotPage();
    if (page && !page->full()) {
        return page->add(obj);
    } else if (page) {
        return autoreleaseFullPage(obj, page);
    } else {
        return autoreleaseNoPage(obj);
    }
}
```

`autoreleaseFast` 函数在执行一个具体的插入操作时，分别对三种情况进行了不同的处理：

1. 当前 page 存在且没有满时，直接将对象添加到当前 page 中，即 `next` 指向的位置；
2. 当前 page 存在且已满时，创建一个新的 page ，并将对象添加到新创建的 page 中；
3. 当前 page 不存在时，即还没有 page 时，创建第一个 page ，并将对象添加到新创建的 page 中。

每调用一次 push 操作就会创建一个新的 autoreleasepool ，即往 AutoreleasePoolPage 中插入一个 POOL_SENTINEL ，并且返回插入的 POOL_SENTINEL 的内存地址。

### autorelease 操作

通过 `NSObject.mm` 源文件，我们可以找到 `-autorelease` 方法的实现：

```
- (id)autorelease {
    return ((id)self)->rootAutorelease();
}
```

通过查看 `((id)self)->rootAutorelease()` 的方法调用，我们发现最终调用的就是 AutoreleasePoolPage 的 `autorelease` 函数。

``` objc
__attribute__((noinline,used))
id 
objc_object::rootAutorelease2()
{
    assert(!isTaggedPointer());
    return AutoreleasePoolPage::autorelease((id)this);
}
```

AutoreleasePoolPage 的 `autorelease` 函数的实现对我们来说就比较容量理解了，它跟 push 操作的实现非常相似。只不过 push 操作插入的是一个 POOL_SENTINEL ，而 autorelease 操作插入的是一个具体的 autoreleased 对象。

``` objc
static inline id autorelease(id obj)
{
    assert(obj);
    assert(!obj->isTaggedPointer());
    id *dest __unused = autoreleaseFast(obj);
    assert(!dest  ||  *dest == obj);
    return obj;
}
```

### pop 操作

同理，前面提到的 `objc_autoreleasePoolPop(void *)` 函数本质上也是调用的 AutoreleasePoolPage 的 `pop` 函数。

``` objc
void
objc_autoreleasePoolPop(void *ctxt)
{
    if (UseGC) return;

    // fixme rdar://9167170
    if (!ctxt) return;

    AutoreleasePoolPage::pop(ctxt);
}
```

pop 函数的入参就是 push 函数的返回值，也就是 POOL_SENTINEL 的内存地址，即 pool token 。当执行 pop 操作时，内存地址在 pool token 之后的所有 autoreleased 对象都会被 release 。直到 pool token 所在 page 的 `next` 指向 pool token 为止。

下面是某个线程的 autoreleasepool 堆栈的内存结构图，在这个 autoreleasepool 堆栈中总共有两个 POOL_SENTINEL ，即有两个 autoreleasepool 。该堆栈由三个 AutoreleasePoolPage 结点组成，第一个 AutoreleasePoolPage 结点为 `coldPage()` ，最后一个 AutoreleasePoolPage 结点为 `hotPage()` 。其中，前两个结点已经满了，最后一个结点中保存了最新添加的 autoreleased 对象 `objr3` 的内存地址。

![AutoreleasePoolPage1](http://blog.leichunfeng.com/images/AutoreleasePoolPage1.png "AutoreleasePoolPage1")

此时，如果执行 `pop(token1)` 操作，那么该 autoreleasepool 堆栈的内存结构将会变成如下图所示：

![AutoreleasePoolPage2](http://blog.leichunfeng.com/images/AutoreleasePoolPage2.png "AutoreleasePoolPage2")

## NSThread、NSRunLoop 和 NSAutoreleasePool

根据苹果官方文档中对 [NSRunLoop](https://developer.apple.com/library/ios/documentation/Cocoa/Reference/Foundation/Classes/NSRunLoop_Class/index.html#//apple_ref/doc/constant_group/Run_Loop_Modes) 的描述，我们可以知道每一个线程，包括主线程，都会拥有一个专属的 NSRunLoop 对象，并且会在有需要的时候自动创建。

>Each NSThread object, including the application’s main thread, has an NSRunLoop object automatically created for it as needed.

同样的，根据苹果官方文档中对 [NSAutoreleasePool](https://developer.apple.com/library/ios/documentation/Cocoa/Reference/Foundation/Classes/NSAutoreleasePool_Class/index.html#//apple_ref/doc/uid/TP40003623) 的描述，我们可知，在主线程的 NSRunLoop 对象（在系统级别的其他线程中应该也是如此，比如通过 dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0) 获取到的线程）的每个 event loop 开始前，系统会自动创建一个 autoreleasepool ，并在 event loop 结束时 drain 。我们上面提到的场景 1 中创建的 autoreleased 对象就是被系统添加到了这个自动创建的 autoreleasepool 中，并在这个 autoreleasepool 被 drain 时得到释放。

>The Application Kit creates an autorelease pool on the main thread at the beginning of every cycle of the event loop, and drains it at the end, thereby releasing any autoreleased objects generated while processing an event.

另外，NSAutoreleasePool 中还提到，每一个线程都会维护自己的 autoreleasepool 堆栈。换句话说 autoreleasepool 是与线程紧密相关的，每一个 autoreleasepool 只对应一个线程。

>Each thread (including the main thread) maintains its own stack of NSAutoreleasePool objects.

弄清楚 NSThread、NSRunLoop 和 NSAutoreleasePool 三者之间的关系可以帮助我们从整体上了解 Objective-C 的内存管理机制，清楚系统在背后到底为我们做了些什么，理解整个运行机制等。

## 总结

看到这里，相信你应该对 Objective-C 的内存管理机制有了更进一步的认识。通常情况下，我们是不需要手动添加 autoreleasepool 的，使用线程自动维护的 autoreleasepool 就好了。根据苹果官方文档中对 [Using Autorelease Pool Blocks](https://developer.apple.com/library/ios/documentation/Cocoa/Conceptual/MemoryMgmt/Articles/mmAutoreleasePools.html#//apple_ref/doc/uid/20000047-CJBFBEDI) 的描述，我们知道在下面三种情况下是需要我们手动添加 autoreleasepool 的：

1. 如果你编写的程序不是基于 UI 框架的，比如说命令行工具；
2. 如果你编写的循环中创建了大量的临时对象；
3. 如果你创建了一个辅助线程。

最后，希望本文能对你有所帮助，have fun ！

## 参考链接

[http://blog.sunnyxx.com/2014/10/15/behind-autorelease/](http://blog.sunnyxx.com/2014/10/15/behind-autorelease/)
[http://clang.llvm.org/docs/AutomaticReferenceCounting.html](http://clang.llvm.org/docs/AutomaticReferenceCounting.html)
[http://www.yifeiyang.net/development-of-the-iphone-simply-3/](http://www.yifeiyang.net/development-of-the-iphone-simply-3/)

**版权声明**：我已将本文在微信公众平台的发表权「独家代理」给 iOS 开发（iOSDevTips）微信公众号。扫下方二维码即可关注「iOS 开发」：

![iOS 开发二维码](http://blog.devtang.com/images/weixin-qr.jpg)
