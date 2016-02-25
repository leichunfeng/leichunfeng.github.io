---
layout: post
title: "MVVM with ReactiveCocoa"
date: 2016-02-01 22:17:12 +0800
comments: true
categories: 
---

[MVVM](https://en.wikipedia.org/wiki/Model%E2%80%93view%E2%80%93viewmodel) 是一种软件架构模式，它是 [Martin Fowler](https://en.wikipedia.org/wiki/Martin_Fowler) 的 [Presentation Model](http://martinfowler.com/eaaDev/PresentationModel.html) 的一种变体，最先由微软的架构师 John Gossman 在 2005 年提出，并应用在微软的 [WPF](https://en.wikipedia.org/wiki/Windows_Presentation_Foundation) 和 [Silverlight](https://en.wikipedia.org/wiki/Microsoft_Silverlight) 软件开发中。`MVVM` 衍生于 [MVC](https://en.wikipedia.org/wiki/Model%E2%80%93view%E2%80%93controller) ，是对 `MVC` 的一种演进，它促进了 `UI` 代码与业务逻辑的分离。

本文将采取理论与实践相结合的方式，介绍一个使用 `MVVM` 和 `ReactiveCocoa` 开发的 `iOS` 开源项目 [MVVMReactiveCocoa](https://github.com/leichunfeng/MVVMReactiveCocoa) 。在正式开始阅读正文之前，请你先思考以下三个问题：

- `MVC` 与 `MVVM` 有什么异同点，`MVC` 到 `MVVM` 是怎样演进的；
- `ReactiveCocoa` 在 `MVVM` 中扮演什么样的角色，`MVVM` 是否一定要用 `ReactiveCocoa` ；
- 一个现有的 `MVC` 应用如何转变成一个 `MVVM` 应用，有什么需要注意的地方。

带着以上问题，我们一起进入正文。

## MVC

`MVC` 是 `iOS` 开发中使用最普遍的架构模式，同时也是苹果官方推荐使用的架构模式。`MVC` 代表的是 `Model–view–controller` ，它们之间的关系如下：

<img src="http://localhost:4000/images/M-V-C.png" width="409" />

`Models` 代表数据，`views` 代表 `UI` ，而 `controllers` 则负责协调它们两者之间的关系。是的，`MVC` 看上去美极了，然而事实上，尽管从技术上来说 `view` 和 `controller` 是相互独立的，但是它们几乎总是结对出现，一个 `view` 只能与一个 `controller` 进行匹配，反之亦然。既然如此，我们何不将它们放在一起呢：

<img src="http://localhost:4000/images/M-VC.png" width="409" />

事实上，`M-VC` 可能是对 `iOS` 中的 `MVC` 模式更为准确的描述。在一个典型的 `MVC` 应用中，有非常多的逻辑被放置在 `controller` 中。其中，有一部分逻辑确实是属于 `controller` 的，但是也有一些逻辑不应该被放置在 `controller` 中，比如将 `model` 中的值转换成 `view` 可以展示的值，举例来说，像将 `NSDate` 转换成格式化的 `NSString` 等，而在 `MVVM` 中，我们称之为展示逻辑。

也正是因为如此，承载了过多逻辑的 `controller` 很快就会变得臃肿不堪，难以维护和扩展。所以，`iOS` 中的 `MVC` 模式还有另外一种非常形象的解读，那就是 [Massive View Controller](https://twitter.com/Colin_Campbell/status/293167951132098560) ：

> iOS architecture, where MVC stands for Massive View Controller.

## MVVM

正如前面所说的，一种可以很好地解决 `Massive View Controller` 的办法就是将 `controller` 中的展示逻辑抽取出来，放置到一个专门的地方，而这个地方就是 `viewModel` 。事实上，我们只要在上图中的 `M-VC` 之间放入 `VM` ，就可以得到 `MVVM` 模式的结构图：

<img src="http://localhost:4000/images/M-V-VM.png" width="409" />

从上图中，我们可以非常清楚地看到 `MVVM` 中三个组件之间的关系。事实上，除了 `view` 、`viewModel` 和 `model` 之外，`MVVM` 中还有另外一个非常重要的隐含组件，那就是 `binder` ：

- `view` ：由 `MVC` 中的 `view` 和 `controller` 组成，负责 `UI` 的展示，绑定 `viewModel` 中的属性，触发 `viewModel` 中的命令；
- `viewModel` ：从 `MVC` 的 `controller` 中抽取出来的展示逻辑，负责从 `model` 中获取 `view` 所需的数据，转换成 `view` 可以展示的数据，并暴露公开的属性和命令供 `view` 进行绑定；
- `model` ：与 `MVC` 中的 `model` 一致，包括数据模型、访问数据库的操作和网络请求等；
- `binder` ：在 `MVVM` 模式中，声明式的数据和命令绑定是一个隐含的约定，它可以让开发者非常方便地实现 `view` 和 `viewModel` 的同步，避免编写大量繁杂的样板化代码。在微软的 `MVVM` 实现中，使用的是一种被称为 [XAML](https://en.wikipedia.org/wiki/Extensible_Application_Markup_Language) 的标记语言。

## ReactiveCocoa

然而，在 iOS 开发中，系统并没有提供类似的框架可以让我们方便地实现 `binder` 的功能。不过值得庆幸的是，`GitHub` 开源的 `ReactiveCocoa` ，给了我们一个非常不错的选择。

`ReactiveCocoa` 是一个 `iOS` 中的函数式响应式编程框架，它受 [Functional Reactive Programming](https://en.wikipedia.org/wiki/Functional_reactive_programming) 的启发，是 [Justin Spahr-Summers](https://github.com/jspahrsummers) 和 [Josh Abernathy](https://github.com/joshaber) 在开发 [GitHub for Mac](https://desktop.github.com/) 过程中的一个副产品，它提供了一系列用来组合和转换值流的 `API` 。关于 `ReactiveCocoa` 的更多信息，可以阅读我的上一篇文章 [《ReactiveCocoa v2.5 源码解析之架构总览》](http://blog.leichunfeng.com/blog/2015/12/25/reactivecocoa-v2-dot-5-yuan-ma-jie-xi-zhi-jia-gou-zong-lan/)。

是的，在 `iOS` 的 `MVVM` 实现中，我们可以使用 `ReactiveCocoa` 来在 `view` 和 `viewModel` 之间充当 `binder` 的角色，从而避免编写大量 `KVO` 等繁复的代码。此外，我们也可以将 `ReactiveCocoa` 用在 `model` 层，使用 `RACSignal` 来代表异步的数据获取操作，比如读取文件、访问数据库和网络请求等。注意，后一个应用场景是与 `MVVM` 无关的，也就是说，我们同样可以在 `MVC` 的 `model` 层这么用。

## 小结

综上所述，`MVVM` 只是 `MVC` 的演进版本，我们只要将 `MVC` 中的 `controller` 中的展示逻辑抽取出来，放置到 `viewModel` 中，然后通过一定的技术手段，比如 `ReactiveCocoa` 来同步 `view` 和 `viewModel` ，就实现了 `MVC` 到 `MVVM` 的转变。

> Talk is cheap. Show me the code.

下面，我们直接上代码，一起来看看 `MVC` 模式的代码要怎样转变成 `MVVM` 模式的代码。假定，我们有一个简单的数据模型 `Person` 和一个相应的 `view controller` 。首先是 `Person` 的代码：

``` objc
@interface Person : NSObject

- (instancetype)initwithSalutation:(NSString *)salutation firstName:(NSString *)firstName lastName:(NSString *)lastName birthdate:(NSDate *)birthdate;

@property (nonatomic, copy, readonly) NSString *salutation;
@property (nonatomic, copy, readonly) NSString *firstName;
@property (nonatomic, copy, readonly) NSString *lastName;
@property (nonatomic, copy, readonly) NSDate *birthdate;

@end
```

接下来是 `view controller` 的代码 `PersonViewController` ，在它的 `viewDidLoad` 方法中，我们将 `Person` 中的属性进行一定的转换后，赋值给 `view` 进行展示：

``` objc
- (void)viewDidLoad {
    [super viewDidLoad];

    if (self.model.salutation.length > 0) {
        self.nameLabel.text = [NSString stringWithFormat:@"%@ %@ %@", self.model.salutation, self.model.firstName, self.model.lastName];
    } else {
        self.nameLabel.text = [NSString stringWithFormat:@"%@ %@", self.model.firstName, self.model.lastName];
    }

    NSDateFormatter *dateFormatter = [[NSDateFormatter alloc] init];
    [dateFormatter setDateFormat:@"EEEE MMMM d, yyyy"];
    self.birthdateLabel.text = [dateFormatter stringFromDate:model.birthdate];
}
```

接下来，我们引入一个 `viewModel` ，将 `PersonViewController` 中的展示逻辑抽取到这个 `PersonViewModel` 中：

``` objc
@interface PersonViewModel : NSObject

- (instancetype)initWithPerson:(Person *)person;

@property (nonatomic, strong, readonly) Person *person;
@property (nonatomic, copy, readonly) NSString *nameText;
@property (nonatomic, copy, readonly) NSString *birthdateText;

@end

@implementation PersonViewModel

- (instancetype)initWithPerson:(Person *)person {
    self = [super init];
    if (self) {
	     _person = person;
	   
	    if (person.salutation.length > 0) {
	        _nameText = [NSString stringWithFormat:@"%@ %@ %@", self.person.salutation, self.person.firstName, self.person.lastName];
	    } else {
	        _nameText = [NSString stringWithFormat:@"%@ %@", self.person.firstName, self.person.lastName];
	    }
	
	    NSDateFormatter *dateFormatter = [[NSDateFormatter alloc] init];
	    [dateFormatter setDateFormat:@"EEEE MMMM d, yyyy"];
	    _birthdateText = [dateFormatter stringFromDate:person.birthdate];

    }
    return self;
}

@end
```

最终，我们的 `PersonViewController` 中的代码将会变得非常轻量级：

``` objc
- (void)viewDidLoad {
    [super viewDidLoad];

    self.nameLabel.text = self.viewModel.nameText;
    self.birthdateLabel.text = self.viewModel.birthdateText;
}
```

怎么样，是不是非常简单呢？其实 `MVVM` 并没有我们想像中的那么难吧，而且更重要的是它也没有颠覆 `MVC` 的现有结构，只不过是移动了一些代码，仅此而已。好了，说了这么多，那么 `iOS` 中的 `MVVM` 相对于 `MVC` 来说到底有哪些杀手级的好处呢？主要可以归纳为以下三点：

- 由于展示逻辑被抽取到了 `viewModel` 中，所以 `view` 中的代码将会变得非常轻量级；
- 由于 `viewModel` 中的代码是与 UI 无关的，所以它可以非常容易被测试；
- 对于一个封装了大量业务逻辑的 `model` 来说，改变它可能会比较困难，并且存在一定的风险。在这种场景下，viewModel 则可以作为 `model` 的适配器来使用，从而避免对 `model` 进行大的改动。

对于第一点，我们通过前面的代码，已经有了一定的感触；至于第三点，可能对于一个复杂的大型应用来说，才会比较明显；那么，接下来，我们还是用前面的例子代码，来直观地感受一下 `MVVM` 的第二点好处，即 `viewModel` 的可测试性：

``` objc
SpecBegin(Person)
    NSString *salutation = @"Dr.";
    NSString *firstName = @"first";
    NSString *lastName = @"last";
    NSDate *birthdate = [NSDate dateWithTimeIntervalSince1970:0];

    it (@"should use the salutation available. ", ^{
        Person *person = [[Person alloc] initWithSalutation:salutation firstName:firstName lastName:lastName birthdate:birthdate];
        PersonViewModel *viewModel = [[PersonViewModel alloc] initWithPerson:person];
        expect(viewModel.nameText).to.equal(@"Dr. first last");
    });

    it (@"should not use an unavailable salutation. ", ^{
        Person *person = [[Person alloc] initWithSalutation:nil firstName:firstName lastName:lastName birthdate:birthdate];
        PersonViewModel *viewModel = [[PersonViewModel alloc] initWithPerson:person];
        expect(viewModel.nameText).to.equal(@"first last");
    });

    it (@"should use the correct date format. ", ^{
        Person *person = [[Person alloc] initWithSalutation:nil firstName:firstName lastName:lastName birthdate:birthdate];
        PersonViewModel *viewModel = [[PersonViewModel alloc] initWithPerson:person];
        expect(viewModel.birthdateText).to.equal(@"Thursday January 1, 1970");
    });
SpecEnd
```

对于 `MVVM` 模式来说，我们可以把 `view` 看作是 `viewModel` 的可视化形式，`viewModel` 提供了 `view` 所需的所有数据和命令。因此，`viewModel` 的可测试性可以极大地提高我们应用的正确性与质量。

## MVVMReactiveCocoa

接下来，我们进入本文的第二部分，重点介绍一个使用 `MVVM` 和 `ReactiveCocoa` 开发的真实应用，`MVVMReactiveCocoa` 。说明，本文将主要介绍这个应用的架构和设计思路，希望可以为你实践 `MVVM` 提供一个真实的参考案例，有些架构并非是 `MVVM` 模式所必须的，而是我们为了更好更顺畅地使用 `MVVM` 而引入的，尤其是 `ViewModel-Based Navigation` ，所以请你在实践的时候能够结合自身应用的实际情况做出相应的取舍，灵活处理，活学活用。最后，我们将以登录界面为具体的例子，一起探讨一下 `MVVM` 的实践过程。

**说明**，以下内容均基于 `MVVMReactiveCocoa` 的 [v2.1.1](https://github.com/leichunfeng/MVVMReactiveCocoa/tree/v2.1.1) 标签进行展开。

### 类图

为了方便我们从宏观上了解 `MVVMReactiveCocoa` 的整体结构，我们先来看看它的类图：

![MVVMReactiveCocoa-v2.1.1](http://localhost:4000/images/MVVMReactiveCocoa-v2.1.1.png)

从上面的类图中，我们可以看到，在 `MVVMReactiveCocoa` 中主要有两大继承体系：

- 用蓝色标识出来的 `viewModel` 的继承体系，基类为 `MRCViewModel` ；
- 用红色标识出来的 `view` 的继承体系，基类为 `MRCViewController` 。

除了提供与系统基类 `UIViewController` 相对应的基类 `MRCViewModel/MRCViewController` 外，还提供了与系统基类 `UITableViewController` 和 `UITabBarController` 相对应的基类 `MRCTableViewModel/MRCTableViewController` 和 `MRCTabBarViewModel/MRCTabBarController` ，其中基类 `MRCTableViewModel/MRCTableViewController` 的使用最为普遍。

**说明**，之所以通过基类的方式来组织 `MVVMReactiveCocoa` ，一方面是因为这个应用的主要开发者只有我一个人，这个方案非常容易实施；另一方面，也是因为通过基类的方式可以尽可能简单地实现代码重用，提高开发效率。

### 服务总线

经过前面的探讨，我们已经知道了 `MVVM` 中的 `viewModel` 的主要职责就是从 `model` 层获取 `view` 所需的数据，并且将这些数据转换成 `view` 能够展示的形式。因此，为了方便 `viewModel` 层调用 `model` 层中的所有服务，并且统一管理这些服务的创建，我使用抽象工厂模式将 `model` 层的所有服务集中管理起来，结构图如下所示：

<img src="http://localhost:4000/images/service-bus.png" width="652" />

从上图中，我们可以看出，在服务总线类 `MRCViewModelServices/MRCViewModelServicesImpl` 中，主要包括以下三个方面的内容：

- 应用自有的服务类，用柚黄色进行了标识，包括 `MRCAppStoreService/MRCAppStoreServiceImpl` 和 `MRCRepositoryService/MRCRepositoryServiceImpl` 两个服务类；
- 第三方 `GitHub` 提供的官方 `API` 框架，用天蓝色进行了标识，主要包括 `OCTClient` 服务类；
- 应用的导航服务，用藻绿色进行了标识，包括 `MRCNavigationProtocol` 协议、虚拟实现类 `MRCViewModelServicesImpl` 等。

其中，前面两者都是以信号的形式对 `viewModel` 层提供服务，代表异步的网络请求等数据获取操作，而我们在 `viewModel` 则可以通过订阅信号的形式获取到所需的数据。此外，服务总线还实现了 `MRCNavigationProtocol` 协议，协议的内容如下：

``` objc
@protocol MRCNavigationProtocol <NSObject>

/// Pushes the corresponding view controller.
///
/// Uses a horizontal slide transition.
/// Has no effect if the corresponding view controller is already in the stack.
///
/// viewModel - the view model
/// animated  - use animation or not
- (void)pushViewModel:(MRCViewModel *)viewModel animated:(BOOL)animated;

/// Pops the top view controller in the stack.
///
/// animated - use animation or not
- (void)popViewModelAnimated:(BOOL)animated;

/// Pops until there's only a single view controller left on the stack.
///
/// animated - use animation or not
- (void)popToRootViewModelAnimated:(BOOL)animated;

/// Present the corresponding view controller.
///
/// viewModel  - the view model
/// animated   - use animation or not
/// completion - the completion handler
- (void)presentViewModel:(MRCViewModel *)viewModel animated:(BOOL)animated completion:(VoidBlock)completion;

/// Dismiss the presented view controller.
///
/// animated   - use animation or not
/// completion - the completion handler
- (void)dismissViewModelAnimated:(BOOL)animated completion:(VoidBlock)completion;

/// Reset the corresponding view controller as the root view controller of the application's window.
///
/// viewModel - the view model
- (void)resetRootViewModel:(MRCViewModel *)viewModel;

@end
```

看上去是不是非常眼熟呢？是的，`MRCNavigationProtocol` 协议其实就是参照系统的 `UINavigationController` 中的导航操作定义出来的，用来实现以 `viewModel` 为基础的导航服务。事实上，服务总线类 `MRCViewModelServicesImpl` 并没有真正地实现 `MRCNavigationProtocol` 协议中定义的操作，只不过是实现了一些空操作而已。

``` objc
- (void)pushViewModel:(MRCViewModel *)viewModel animated:(BOOL)animated {}

- (void)popViewModelAnimated:(BOOL)animated {}

- (void)popToRootViewModelAnimated:(BOOL)animated {}

- (void)presentViewModel:(MRCViewModel *)viewModel animated:(BOOL)animated completion:(VoidBlock)completion {}

- (void)dismissViewModelAnimated:(BOOL)animated completion:(VoidBlock)completion {}

- (void)resetRootViewModel:(MRCViewModel *)viewModel {}
```

那么，我们是怎么实现以 `viewModel` 为基础的导航操作的呢？用 `MRCViewModelServicesImpl` 来实现这些空操作到底有什么用意？为什么要这么做，目的是为了什么？兄台，莫急，请看下一小节的内容。

### ViewModel-Based Navigation

### 登录界面

## 总结

## 参考链接

