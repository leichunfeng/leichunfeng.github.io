---
layout: post
title: "ReactiveCocoa v2.5 源码解析之架构总览"
date: 2015-12-25 20:44:27 +0800
comments: true
categories: 
---

`ReactiveCocoa` 是一个 `iOS` 中的函数式响应式编程框架，它受 [Functional Reactive Programming](https://en.wikipedia.org/wiki/Functional_reactive_programming) 的启发，是 [Justin Spahr-Summers](https://github.com/jspahrsummers) 和 [Josh Abernathy](https://github.com/joshaber) 在开发 [GitHub for Mac](https://desktop.github.com/) 过程中的一个副产品，它提供了一系列用于组合和转换值流的 `API` 。

[Mattt Thompson](http://nshipster.com/reactivecocoa/) 大神是这样评价 `ReactiveCocoa` 的：

> Breaking from a tradition of covering Apple APIs exclusively, this edition of NSHipster will look at an open source project that exemplifies this brave new era for Objective-C.

他认为 `ReactiveCocoa` 打破了苹果 `API` 排他性的束缚，勇敢地开创了 `Objective-C` 的新纪元，具有划时代的意义。不得不说，这对于一个第三方框架来说，已经是非常高的评价了。

关于 `ReactiveCocoa` 的版本演进历程，简单介绍如下：

- `<= v2.5` ：`Objective-C` ；
- `v3.x` ：`Swift 1.2` ；
- `v4.x` ：`Swift 2.x` 。

**注**：本文所介绍的均为 `ReactiveCocoa v2.5` 版本中的内容，这是 `Objective-C` 最新的稳定版本。另外，本文的目录结构如下：

- 简介
- 信号源
	- RACStream
	- RACSignal
	- RACSubject
	- RACSequence
- 订阅者
	- RACSubscriber
	- RACMulticastConnection
- 调度器
	- RACScheduler
- 清洁工
	- RACDisposable
- 总结

**PS**：[MVVMReactiveCocoa](https://github.com/leichunfeng/MVVMReactiveCocoa) 是我用 `MVVM` + `RAC` 编写的一个开源应用，如果你有兴趣的话不妨 `clone` 下来看看 `ReactiveCocoa` 的具体实践吧。

## 简介

`ReactiveCocoa` 是一个非常复杂的框架，在正式开始介绍它的核心组件前，我们先来看看它的类图，以便从宏观上了解它的层次结构：

![ReactiveCocoa v2.5](http://localhost:4000/images/ReactiveCocoa v2.5.png)

从上面的类图中，我们可以看出，`ReactiveCocoa` 主要由以下四大核心组件构成：

- 信号源：`RACStream` 及其子类；
- 订阅者：`RACSubscriber` 的实现类及其子类；
- 调度器：`RACScheduler` 及其子类；
- 清洁工：`RACDisposable` 及其子类。

其中，信号源又是最核心的部分，其他组件都是围绕它运作的。

对于一个应用来说，绝大部分的时间都是在等待某些事件的发生或响应某些状态的变化，比如用户的触摸事件、应用进入前台、网络请求成功刷新界面等等，而维护这些状态的变化，常常会使代码变得非常复杂，难以扩展。而 `ReactiveCocoa` 给出了一种非常好的解决方案，它使用信号来代表这些异步事件，提供了一种统一的方式来处理异步的行为，包括代理方法、`block` 回调、`target-action` 机制、通知、`KVO` 等：

``` objc
// 代理方法
[[self
    rac_signalForSelector:@selector(webViewDidStartLoad:)
    fromProtocol:@protocol(UIWebViewDelegate)]
    subscribeNext:^(id x) {
        // 实现 webViewDidStartLoad: 代理方法
    }];
    
// target-action
[[self.avatarButton
    rac_signalForControlEvents:UIControlEventTouchUpInside]
    subscribeNext:^(UIButton *avatarButton) {
        // avatarButton 被点击了
    }
    
// 通知
[[[NSNotificationCenter defaultCenter]
    rac_addObserverForName:kReachabilityChangedNotification object:nil]
    subscribeNext:^(NSNotification *notification) {
        // 收到 kReachabilityChangedNotification 通知
    }];
    
// KVO
[RACObserve(self, username) subscribeNext:^(NSString *username) {
    // 用户名发生了变化
}];
```

这些只是 `ReactiveCocoa` 的冰山一角，它真正强大的地方在于我们可以对这些信号进行任意地组合和链式操作，从最原始的输入开始直至得到最终的输出结果为止：

``` objc
[[[RACSignal
    combineLatest:@[ RACObserve(self, username), RACObserve(self, password) ]
    reduce:^(NSString *username, NSString *password) {
    	return @(username.length > 0 && password.length > 0);
    }]
    distinctUntilChanged]
    subscribeNext:^(NSNumber *valid) {
        if (valid.boolValue) {
            // 用户名和密码合法，登录按钮可用
        } else {
            // 用户名或密码不合法，登录按钮不可用
        }
    }];
```

因此，对于 `ReactiveCocoa` 而言，我们可以毫不夸张地说，阻碍它发挥的瓶颈就只剩下你的想象力了。

## 信号源

在 `ReactiveCocoa` 中，信号源代表的是随着时间而改变的值流，订阅者可以通过订阅信号源来获取这些值：

``` objc
Streams of values over time.
```

你可以把它想象成水龙头中的水，当你打开水龙头时，水源源不断地流出来；你也可以把它想象成电，当你插上插头时，电静静地充到你的手机上；你还可以把它想象成运送玻璃珠的管道，当你打开阀门时，珠子一个接一个地到达。这里的水、电、玻璃珠就是我们所需要的值，而打开水龙头、插上插头、打开阀门就是订阅它们的过程。

### RACStream

`RACStream` 是 `ReactiveCocoa` 中最核心的类，代表的是任意的值流，它是整个 `ReactiveCocoa` 得以建立的基石，下面是它的继承结构图：

<img src="http://blog.leichunfeng.com/images/RACStream.png" width="650" />

事实上，`RACStream` 是一个抽象类，通常情况下，我们并不会去实例化它，而是直接使用它的子类 `RACSignal` 和 `RACSequence` 。那么，问题来了，为什么 `RACStream` 会被设计成一个抽象类？或者说它是依据什么概念进行设计的呢？

是的，没错。看过上一篇文章 [《Functor、Applicative 和 Monad》](http://blog.leichunfeng.com/blog/2015/11/08/functor-applicative-and-monad/) 的同学，应该已经知道了，`RACStream` 就是根据 `Monad` 的概念进行设计的，它代表的就是一个 `Monad` ：

``` objc
/// An abstract class representing any stream of values.
///
/// This class represents a monad, upon which many stream-based operations can
/// be built.
///
/// When subclassing RACStream, only the methods in the main @interface body need
/// to be overridden.
@interface RACStream : NSObject

/// Lifts `value` into the stream monad.
///
/// Returns a stream containing only the given value.
+ (instancetype)return:(id)value;

/// Lazily binds a block to the values in the receiver.
///
/// This should only be used if you need to terminate the bind early, or close
/// over some state. -flattenMap: is more appropriate for all other cases.
///
/// block - A block returning a RACStreamBindBlock. This block will be invoked
///         each time the bound stream is re-evaluated. This block must not be
///         nil or return nil.
///
/// Returns a new stream which represents the combined result of all lazy
/// applications of `block`.
- (instancetype)bind:(RACStreamBindBlock (^)(void))block;

@end
```

有了 `Monad` 作为基石后，许多基于流的操作就可以被建立起来了，比如 `map` 、`filter` 、`zip` 等。

### RACSignal

`RACSignal` 通常代表的是未来将会被传送的值，它是一种 `push-driven` 的流。`RACSignal` 可以向订阅者发送三种不同类型的事件：

- `next` ：`RACSignal` 通过 `next` 事件向订阅者传送新的值，并且这个值可以为 `nil` ；
- `error` ：`RACSignal` 通过 `error` 事件向订阅者表明信号在正常结束前发生了错误；
- `completed` ：`RACSignal` 通过 `completed` 事件向订阅者表明信号已经正常结束，不会再有后续的值传送给订阅者。

**注意**，我们所说的值流只包含正常的值，即通过 `next` 事件传送的值，并不包括 `error` 和 `completed` 事件，它们需要被特殊处理。通常情况下，一个信号的生命周期是由任意个 `next` 事件和一个 `error` 事件或一个 `completed` 事件组成的。

从上面的类图中，我们可以看出，`RACSignal` 并非只有一个类，事实上，它的一系列功能是用一个类簇来实现的。除去我们将在下节介绍的 `RACSubject` 及其子类外，`RACSignal` 还有五个用来实现不同功能的私有子类：

- `RACEmptySignal` ：空信号，用来实现 `RACSignal` 的 `+empty` 方法；
- `RACReturnSignal` ：一元信号，用来实现 `RACSignal` 的 `+return:` 方法；
- `RACDynamicSignal` ：动态信号，使用一个 `block` 来实现订阅行为，我们使用 `RACSignal` 的 `+createSignal:` 方法创建的就是该类的实例；
- `RACErrorSignal` ：错误信号，用来实现 `RACSignal` 的 `+error` 方法；
- `RACChannelTerminal` ：通道终端，代表 `RACChannel` 的一个终端。

`RACSignal` 继承自 `RACStream` ，并且实现了 `+return:` 和 `-bind:` 方法，所以它也是 `Monad` ，具体的实现细节我们将在后续的文章中进行介绍。

对于 `RACSignal` 类簇而言，最核心的方法莫过于 `-subscribe:` 了，这个方法封装了订阅者对信号源的一次订阅过程，它是订阅者与信号源产生联系的唯一入口。因此，对于 `RACSignal` 的所有子类而言，这个方法的实现逻辑就代表了该子类的具体订阅行为，是区分不同子类的关键所在。同时，这也是为什么 `RACSignal` 中的 `-subscribe:` 方法是一个抽象方法，并且必须要让子类实现的原因：

``` objc
- (RACDisposable *)subscribe:(id<RACSubscriber>)subscriber {
	NSCAssert(NO, @"This method must be overridden by subclasses");
	return nil;
}
```

### RACSubject

`RACSubject` 代表的是可以手动控制的信号，我们可以把它看作是 `RACSignal` 的可变版本，就好比 `NSMutableArray` 是 `NSArray` 的可变版本一样。`RACSubject` 继承自 `RACSignal` ，所以它可以作为信号源被订阅者订阅，同时，它也实现了 `RACSubscriber` 协议，所以它又可以作为订阅者订阅其他信号源，这个正是它为什么可以手动控制的原因：

<img src="http://localhost:4000/images/RACSubject.png" width="600" />

根据官方的 [Design Guidelines](https://github.com/ReactiveCocoa/ReactiveCocoa/blob/v2.5/Documentation/DesignGuidelines.md#avoid-using-subjects-when-possible) 中的说法，我们应该尽可能少地使用它。因为它太过灵活，我们可以在任何时候任何地方操作它，所以一旦过度使用，就会使代码变得非常复杂。

根据我的实际使用经验，在 `MVVM` 中使用 `RACSubject` 可以非常方便地实现统一的错误处理逻辑。比如，我们首先在 `viewModel` 的基类中声明一个 `RACSubject` 类型的属性 `errors` ，然后在 `viewController` 的基类中编写统一的错误处理逻辑：

```
[self.viewModel.errors subscribeNext:^(NSError *error) {
    // 错误处理逻辑
}
```

此时，假设在某个界面的 `viewModel` 中有三个用于请求远程数据的命令，分别是 `requestReadmeMarkdownCommand` 、`requestBlobCommand` 、`requestReadmeHTMLCommand` ，那么这个界面的错误处理逻辑就非常简单了，只需要编写一行代码：

``` objc
[[RACSignal
     	merge:@[
            self.requestReadmeMarkdownCommand.errors,
            self.requestBlobCommand.errors,
            self.requestReadmeHTMLCommand.errors
        ]]
     	subscribe:self.errors];
```

另外，`RACSubject` 也有三个用于实现不同功能的子类：

- `RACGroupedSignal` ：分组信号，用来实现 `RACSignal` 的分组功能；
- `RACBehaviorSubject` ：重演最后值的信号，当被订阅时，向订阅者发送它最后接收到的值；
- `RACReplaySubject` ：重演信号，保存发送过的值，当被订阅时，向订阅者重新发送这些值。

`RACSubject` 的功能非常强大，也正因为如此，我们在使用它时也需要慎重考虑。

### RACSequence

`RACSequence` 代表的是一个不可变的值的序列，与 `RACSignal` 不同，它是 `pull-driven` 类型的流。从严格意义上讲，`RACSequence` 并不能算作是信号源，因为它并不能像 `RACSignal` 那样，可以被订阅者订阅，但是它与 `RACSignal` 之间可以非常方便地进行转换。

从理论上看，一个 `RACSequence` 由两部分组成：

- `head` ：指的是序列中的第一个对象，如果序列为空，则为 `nil` ；
- `tail` ：指的是序列中除第一个对象外的其它所有对象，如果序列为空，则为 `nil` 。

**注**，一个序列的 `tail` 也是序列。如果我们将序列看作是一条毛毛虫，那么 `head` 和 `tail` 可表示如下：

![listmonster](http://localhost:4000/images/listmonster.png)

同样的，一个序列的 `tail` 也可以看作是由 `head` 和 `tail` 组成，而这个新的 `tail` 又可以继续看作是由 `head` 和 `tail` 组成，这个过程可以一直进行下去。而这个正是 `RACSequence` 得以建立的理论基础，所以一个 `RACSequence` 子类的最小实现就是 `head` 和 `tail` ：

``` objc
/// Represents an immutable sequence of values. Unless otherwise specified, the
/// sequences' values are evaluated lazily on demand. Like Cocoa collections,
/// sequences cannot contain nil.
///
/// Most inherited RACStream methods that accept a block will execute the block
/// _at most_ once for each value that is evaluated in the returned sequence.
/// Side effects are subject to the behavior described in
/// +sequenceWithHeadBlock:tailBlock:.
///
/// Implemented as a class cluster. A minimal implementation for a subclass
/// consists simply of -head and -tail.
@interface RACSequence : RACStream <NSCoding, NSCopying, NSFastEnumeration>

/// The first object in the sequence, or nil if the sequence is empty.
///
/// Subclasses must provide an implementation of this method.
@property (nonatomic, strong, readonly) id head;

/// All but the first object in the sequence, or nil if the sequence is empty.
///
/// Subclasses must provide an implementation of this method.
@property (nonatomic, strong, readonly) RACSequence *tail;

@end
```

总的来说，`RACSequence` 存在的最大意义就是为了简化 `Objective-C` 中的集合操作：

``` objc
Higher-order functions like map, filter, fold/reduce are sorely missing from Foundation.
```

比如下面的代码：

``` objc
NSMutableArray *results = [NSMutableArray array];
for (NSString *str in strings) {
    if (str.length < 2) {
        continue;
    }

    NSString *newString = [str stringByAppendingString:@"foobar"];
    [results addObject:newString];
}
```

可以用 `RACSequence` 来优雅地实现：

``` objc
RACSequence *results = [[strings.rac_sequence
    filter:^ BOOL (NSString *str) {
        return str.length >= 2;
    }]
    map:^(NSString *str) {
        return [str stringByAppendingString:@"foobar"];
    }];
```

我们用 `RACSequence` 可以非常方便地实现集合的链式操作，直到得到你想要的最终结果为止。除此之外，使用 `RACSequence` 的另外一个好处是，`RACSequence` 中的值在默认情况下是懒计算的，即只有在用到的时候才会进行计算，且只计算一次。也就是说，如果我们只用到了 `RACSequence` 中的部分值的话，它可以提高我们应用的性能。

同样的，`RACSequence` 的一系列功能也是通过类簇来实现的，它共有九个用来实现不同功能的私有子类：

- `RACUnarySequence` ：一元序列，用来实现 `RACSequence` 的 `+return:` 方法；
- `RACIndexSetSequence` ：用来遍历索引集；
- `RACEmptySequence` ：空序列，用来实现 `RACSequence` 的 `+empty` 方法；
- `RACDynamicSequence` ：动态序列，使用 `blocks` 来动态地实现一个序列；
- `RACSignalSequence` ：用来遍历信号中的值；
- `RACArraySequence` ：用来遍历数组中的元素；
- `RACEagerSequence` ：非懒计算的序列，在初始化时立即计算所有的值；
- `RACStringSequence` ：用来遍历字符串中的字符；
- `RACTupleSequence` ：用来遍历元组中的元素。

`RACSequence` 为类簇提供了统一的对外接口，对于使用它的客户端代码来说，完全不需要知道私有子类的存在，非常好的隐藏了实现细节。另外，值得一提的是，`RACSequence` 类实现了快速枚举的协议 `NSFastEnumeration` ，在这个协议中只声明了下面这个看上去非常抽筋的方法：

``` objc
- (NSUInteger)countByEnumeratingWithState:(NSFastEnumerationState *)state objects:(id __unsafe_unretained [])buffer count:(NSUInteger)len;
```

有兴趣的同学，可以看看 `RACSequence` 中的具体实现，我们将会在后续的文章中进行介绍。因此，我们可以直接使用 `for in` 来遍历 `RACSequence` 。

## 订阅者

现在，我们已经知道信号源是什么了，为了获取信号源中的值，我们需要对信号源进行订阅。在 `ReactiveCocoa` 中，订阅者是一个抽象的概念，所有实现了 `RACSubscriber` 协议的类都可以作为信号源的订阅者。

### RACSubscriber

在 `RACSubscriber` 协议中，声明了四个必须实现的方法：

``` objc
/// Represents any object which can directly receive values from a RACSignal.
///
/// You generally shouldn't need to implement this protocol. +[RACSignal
/// createSignal:], RACSignal's subscription methods, or RACSubject should work
/// for most uses.
///
/// Implementors of this protocol may receive messages and values from multiple
/// threads simultaneously, and so should be thread-safe. Subscribers will also
/// be weakly referenced so implementations must allow that.
@protocol RACSubscriber <NSObject>

@required

/// Sends the next value to subscribers.
///
/// value - The value to send. This can be `nil`.
- (void)sendNext:(id)value;

/// Sends the error to subscribers.
///
/// error - The error to send. This can be `nil`.
///
/// This terminates the subscription, and invalidates the subscriber (such that
/// it cannot subscribe to anything else in the future).
- (void)sendError:(NSError *)error;

/// Sends completed to subscribers.
///
/// This terminates the subscription, and invalidates the subscriber (such that
/// it cannot subscribe to anything else in the future).
- (void)sendCompleted;

/// Sends the subscriber a disposable that represents one of its subscriptions.
///
/// A subscriber may receive multiple disposables if it gets subscribed to
/// multiple signals; however, any error or completed events must terminate _all_
/// subscriptions.
- (void)didSubscribeWithDisposable:(RACCompoundDisposable *)disposable;

@end
```

其中 `-sendNext:` 、`-sendError:` 和 `-sendCompleted` 分别用于从 `RACSignal` 接收 `next` 、`error` 和 `completed` 事件，而 `-didSubscribeWithDisposable:` 则用于接收代表某次订阅的 `disposable` 对象。

订阅者对信号源的一次订阅过程可以抽象为：通过 `RACSignal` 的 `-subscribe:` 方法传入一个订阅者，并最终返回一个 `RACDisposable` 对象的过程：

<img src="http://localhost:4000/images/subscribe.png" width="450" />

**注意**：在 `ReactiveCocoa` 中并没有专门的类 `RACSubscription` 来表示一次订阅，而间接地使用 `RACDisposable` 来充当这一角色。因此，一个 `RACDisposable` 对象就代表着一次订阅，并且我们可以用它来取消这次订阅，详细内容将会在下面的章节中进行介绍。

除了 `RACSignal` 的子类外，还有两个实现了 `RACSubscriber` 协议的类，类图如下：

<img src="http://localhost:4000/images/RACSubscriber.png" width="450" />

其中，`RACSubscriber` 类的名字与 `RACSubscriber` 协议的名字相同，这跟 `Objective-C` 中的 `NSObject` 类的名字与 `NSObject` 协议的名字相同是一样，除了名字相同外，然并卵。通常来说，`RACSubscriber` 类充当的角色就是信号源的真正订阅者，它非常“老实地”实现了 `RACSubscriber` 协议。

既然 `RACSubscriber` 类就是真正的订阅者，那么 `RACPassthroughSubscriber` 类又是干嘛用的呢？原来在 `ReactiveCocoa` 中，一个订阅者是可以订阅多个信号源的，也就是说它会拥有多个 `RACDisposable` 对象，并且它可以随时取消其中任何一个订阅。为了实现这个功能，`ReactiveCocoa` 就引入了 `RACPassthroughSubscriber` 类，它是 `RACSubscriber` 类的一个装饰器，封装了一个真正的订阅者 `RACSubscriber` 对象，负责传递所有事件给这个真正的订阅者，而当此次订阅被取消时，它就会停止传递所有事件：

``` objc
- (void)sendNext:(id)value {
	if (self.disposable.disposed) return;
	[self.innerSubscriber sendNext:value];
}

- (void)sendError:(NSError *)error {
	if (self.disposable.disposed) return;
	[self.innerSubscriber sendError:error];
}

- (void)sendCompleted {
	if (self.disposable.disposed) return;
	[self.innerSubscriber sendCompleted];
}
```

在 `ReactiveCocoa` 的实现中，我们倾向于隐藏订阅者，外部根本不需要知道订阅者的存在，因为这是 `ReactiveCocoa` 内部的实现细节。这样做的主要目的是进一步简化信号源的订阅逻辑，客户端代码只需要关心它所需要的值就可以了，根本不需要关心内部的订阅过程。

### RACMulticastConnection

有的时候，我们在订阅一个信号源的过程中可能会产生副作用或者消耗比较大的资源，比如修改全局变量、发送网络请求等。这个时候，我们往往需要让多个订阅者之间共享一次订阅，就好比我们读高中时，多个好朋友一起订阅一份英文周报，然后只要出一份钱，是一个道理。这就是 `ReactiveCocoa` 中引入 `RACMulticastConnection` 类的原因。

`RACMulticastConnection` 通过一个标志 `_hasConnected` 来保证只对 `sourceSignal` 订阅一次，然后对外暴露一个 `RACSubject` 类型的 `signal` 供外部订阅者订阅。这样一来，不管外部订阅者对 `signal` 订阅多少次，我们对 `sourceSignal` 的订阅至多只会有一次：

![RACMulticastConnection](http://localhost:4000/images/RACMulticastConnection.png)

了解 `RACMulticastConnection` 的实现原理，对于我们后面理解 `-replay` 、`replayLast` 和 `replayLazily` 等方法非常有帮助。

## 调度器

有了信号源和订阅者后，我们还需要有调度器来统一调度订阅者订阅信号源的过程中所涉及到的任务，这样才能保证所有的任务都能够合理有序地执行。

### RACScheduler

`RACScheduler` 在 `ReactiveCocoa` 就是扮演着调度器的角色，本质上，它就是用 GCD 的串行队列来实现的，并且支持取消操作。是的，在 `ReactiveCocoa` 中，并没有使用到 `NSOperationQueue` 和 `NSRunloop` 等技术，`RACScheduler` 只是对 `GCD` 的简单封装而已。

同样的，`RACScheduler` 的一系列功能也是通过类簇来实现的，除了用于测试的子类外，总共还有四个私有子类：

![RACScheduler](http://localhost:4000/images/RACScheduler.png)

咋看之下，`RACScheduler` 的儿子貌似还不少，但是真正出力干活的还真不多，主要就是 `RACTargetQueueScheduler` ：

- `RACImmediateScheduler` ：立即执行调度的任务，这是唯一一个支持同步执行的调度器；
- `RACQueueScheduler` ：一个抽象的队列调度器，在一个 `GCD` 串行列队中异步调度所有任务；
- `RACTargetQueueScheduler` ：继承自 `RACQueueScheduler` ，在一个以一个任意的 `GCD` 队列为 `target` 的串行队列中异步调度所有任务；
- `RACSubscriptionScheduler` ：一个只用于调度订阅的调度器。

值得一提的是，在 `RACScheduler` 中有一个非常特殊的方法：

``` objc
- (RACDisposable *)scheduleRecursiveBlock:(RACSchedulerRecursiveBlock)recursiveBlock;
```

这个方法的作用非常有意思，他可以将递归调用转换成迭代调用，这样做的目的是为了解决深层次的递归调用可能会出现的堆栈溢出问题。

## 清洁工

正如我们前面所说的，在订阅者订阅信号源的过程中，可能会产生副作用或者消耗一定的资源，所以当我们取消订阅或者完成订阅时，我们就需要做一些资源回收和垃圾清理的工作。

### RACDisposable

`RACDisposable` 在 `ReactiveCocoa` 中就充当着清洁工的角色，它封装了取消和清理一次订阅所必需的工作。它有一个最核心的方法 `-dispose` ，调用这个方法就会执行相应的清理工作，这有点类似于 `NSObject` 的 `-dealloc` 方法。`RACDisposable` 总共有四个子类，它的继承结构图如下：

<img src="http://localhost:4000/images/RACDisposable.png" width="450" />

- `RACSerialDisposable` ：作为 `disposable` 的容器使用，可以包含一个 `disposable` 对象，并且允许将这个 `disposable` 对象通过原子操作交换出来；
- `RACKVOTrampoline` ：代表一次 `KVO` 观察，用于停止观察；
- `RACCompoundDisposable` ：跟 `RACSerialDisposable` 一样，`RACCompoundDisposable` 也是作为 `disposable` 的容器使用。不同的是，它可以包含多个 `disposable` 对象，可以手动添加和移除 `disposable` 对象，有点类似于可变数组 `NSMutableArray` 。而当一个 `RACCompoundDisposable` 对象被 `disposed` 时，它会调用其所包含的所有 `disposable` 对象的 `-dispose` 方法，有点类似于 `autoreleasepool` 的作用;
- `RACScopedDisposable` ：当它被 `dealloc` 的时候调用本身的 `-dispose` 方法。

咋看之下，`RACDisposable` 的逻辑似乎有些复杂，不过换汤不换药，不管怎么换着花样玩，最终都只是为了在合适的时机调用 `disposable` 对象的 `-dispose` 方法，执行清理工作而已。

## 总结

至此，我们介绍完了 `ReactiveCocoa` 的四大核心组件，对它的架构有了宏观上的认识，它建立于 `Monad` 的概念之后，并且围绕其搭建了一系列完整的配套组件，它们一起支撑了 `ReactiveCocoa` 的强大功能。尽管，`ReactiveCocoa` 是一个重型的函数式响应式框架，但是它并不会对我们的现有代码造成侵略性，我们完全可以在一个单独的类中使用它，哪怕只是用到一行代码，也是完全可以的。所以，如果你对 `ReactiveCocoa` 感兴趣的话，不妨就从现在开始尝试吧，Let's go ！

## 参考链接

[https://github.com/ReactiveCocoa/ReactiveCocoa/tree/v2.5](https://github.com/ReactiveCocoa/ReactiveCocoa/tree/v2.5)
[https://github.com/ReactiveCocoa/ReactiveCocoa/blob/v2.5/Documentation/FrameworkOverview.md](https://github.com/ReactiveCocoa/ReactiveCocoa/blob/v2.5/Documentation/FrameworkOverview.md)
[https://github.com/ReactiveCocoa/ReactiveCocoa/blob/v2.5/Documentation/DesignGuidelines.md#avoid-using-subjects-when-possible](https://github.com/ReactiveCocoa/ReactiveCocoa/blob/v2.5/Documentation/DesignGuidelines.md#avoid-using-subjects-when-possible)
[http://nshipster.com/reactivecocoa/](http://nshipster.com/reactivecocoa/)
[http://nathanli.cn/2015/08/27/reactivecocoa2-%E6%BA%90%E7%A0%81%E6%B5%85%E6%9E%90/](http://nathanli.cn/2015/08/27/reactivecocoa2-%E6%BA%90%E7%A0%81%E6%B5%85%E6%9E%90/)
[http://blog.devtang.com/blog/2014/02/11/reactivecocoa-introduction/](http://blog.devtang.com/blog/2014/02/11/reactivecocoa-introduction/)
[http://m.oschina.net/blog/294178](http://m.oschina.net/blog/294178)

<img src="http://blog.leichunfeng.com/images/wechat_pay.jpg" width="260" />


