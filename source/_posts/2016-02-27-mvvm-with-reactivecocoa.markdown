---
layout: post
title: "MVVM with ReactiveCocoa"
date: 2016-02-27 22:17:12 +0800
comments: true
categories: 
keywords: MVVMReactiveCocoa, MVVM, ReactiveCocoa, RAC, MVC, ViewModel
---

[MVVM](https://en.wikipedia.org/wiki/Model%E2%80%93view%E2%80%93viewmodel) 是一种软件架构模式，它是 [Martin Fowler](https://en.wikipedia.org/wiki/Martin_Fowler) 的 [Presentation Model](http://martinfowler.com/eaaDev/PresentationModel.html) 的一种变体，最先由微软的架构师 John Gossman 在 2005 年提出，并应用在微软的 [WPF](https://en.wikipedia.org/wiki/Windows_Presentation_Foundation) 和 [Silverlight](https://en.wikipedia.org/wiki/Microsoft_Silverlight) 软件开发中。`MVVM` 衍生于 [MVC](https://en.wikipedia.org/wiki/Model%E2%80%93view%E2%80%93controller) ，是对 `MVC` 的一种演进，它促进了 `UI` 代码与业务逻辑的分离。

**说明**：本文将采用理论与实践相结合的方式，重点介绍一个使用 `MVVM` 和 `RAC` 开发的 `iOS` 开源项目 [MVVMReactiveCocoa](https://github.com/leichunfeng/MVVMReactiveCocoa) ，目的是希望能为你实践 `MVVM` 提供帮助。不过，在正式开始介绍正文之前，请你先思考以下三个问题：

- `MVC` 与 `MVVM` 有什么异同点，`MVC` 到 `MVVM` 是怎样演进的；
- `RAC` 在 `MVVM` 中扮演什么样的角色，`MVVM` 是否一定要结合 `RAC` 使用；
- 如何将一个现有的 `MVC` 应用转变成一个 `MVVM` 应用，有哪些需要注意的地方。

带着以上问题，我们一起进入正文。

**名词解释**：本文中的 `RAC` 为 `ReactiveCocoa` 的缩写。

## MVC

`MVC` 是 `iOS` 开发中使用最普遍的架构模式，同时也是苹果官方推荐的架构模式。`MVC` 代表的是 `Model–view–controller` ，它们之间的关系如下：

<img src="http://localhost:4000/images/M-V-C.png" width="409" />

是的，`MVC` 看上去棒极了，`model` 代表数据，`view` 代表 `UI` ，而 `controller` 则负责协调它们两者之间的关系。然而，尽管从技术上看 `view` 和 `controller` 是相互独立的，但事实上它们几乎总是结对出现，一个 `view` 只能与一个 `controller` 进行匹配，反之亦然。既然如此，那我们为何不将它们看作一个整体呢：

<img src="http://localhost:4000/images/M-VC.png" width="409" />

因此，`M-VC` 可能是对 `iOS` 中的 `MVC` 模式更为准确的解读。在一个典型的 `MVC` 应用中，`controller` 由于承载了过多的逻辑，往往会变得臃肿不堪，所以 `MVC` 也经常被人调侃成 [Massive View Controller](https://twitter.com/Colin_Campbell/status/293167951132098560) ：

> iOS architecture, where MVC stands for Massive View Controller.

坦白说，有一部分逻辑确实是属于 `controller` 的，但是也有一部分逻辑是不应该被放置在 `controller` 中的。比如，将 `model` 中的 `NSDate` 转换成 `view` 可以展示的 `NSString` 等。在 `MVVM` 中，我们将这些逻辑统称为展示逻辑。

## MVVM

因此，一种可以很好地解决 `Massive View Controller` 问题的办法就是将 `controller` 中的展示逻辑抽取出来，放置到一个专门的地方，而这个地方就是 `viewModel` 。其实，我们只要在上图中的 `M-VC` 之间放入 `VM` ，就可以得到 `MVVM` 模式的结构图：

<img src="http://localhost:4000/images/M-V-VM.png" width="409" />

从上图中，我们可以非常清楚地看到 `MVVM` 中四个组件之间的关系。**注**：除了 `view` 、`viewModel` 和 `model` 之外，`MVVM` 中还有一个非常重要的隐含组件 `binder` ：

- `view` ：由 `MVC` 中的 `view` 和 `controller` 组成，负责 `UI` 的展示，绑定 `viewModel` 中的属性，触发 `viewModel` 中的命令；
- `viewModel` ：从 `MVC` 的 `controller` 中抽取出来的展示逻辑，负责从 `model` 中获取 `view` 所需的数据，转换成 `view` 可以展示的数据，并暴露公开的属性和命令供 `view` 进行绑定；
- `model` ：与 `MVC` 中的 `model` 一致，包括数据模型、访问数据库的操作和网络请求等；
- `binder` ：在 `MVVM` 中，声明式的数据和命令绑定是一个隐含的约定，它可以让开发者非常方便地实现 `view` 和 `viewModel` 的同步，避免编写大量繁杂的样板化代码。在微软的 `MVVM` 实现中，使用的是一种被称为 [XAML](https://en.wikipedia.org/wiki/Extensible_Application_Markup_Language) 的标记语言。

## ReactiveCocoa

尽管，在 `iOS` 开发中，系统并没有提供类似的框架可以让我们方便地实现 `binder` 功能，不过，值得庆幸的是，`GitHub` 开源的 `RAC` ，给了我们一个非常不错的选择。

`RAC` 是一个 `iOS` 中的函数式响应式编程框架，它受 [Functional Reactive Programming](https://en.wikipedia.org/wiki/Functional_reactive_programming) 的启发，是 [Justin Spahr-Summers](https://github.com/jspahrsummers) 和 [Josh Abernathy](https://github.com/joshaber) 在开发 [GitHub for Mac](https://desktop.github.com/) 过程中的一个副产品，它提供了一系列用来组合和转换值流的 `API` 。如需了解更多关于 `RAC` 的信息，可以阅读我的上一篇文章[《ReactiveCocoa v2.5 源码解析之架构总览》](http://blog.leichunfeng.com/blog/2015/12/25/reactivecocoa-v2-dot-5-yuan-ma-jie-xi-zhi-jia-gou-zong-lan/)。

在 `iOS` 的 `MVVM` 实现中，我们可以使用 `RAC` 来在 `view` 和 `viewModel` 之间充当 `binder` 的角色，优雅地实现两者之间的同步。此外，我们还可以把 `RAC` 用在 `model` 层，使用 `Signal` 来代表异步的数据获取操作，比如读取文件、访问数据库和网络请求等。**说明**，`RAC` 的后一个应用场景是与 `MVVM` 无关的，也就是说，我们同样可以在 `MVC` 的 `model` 层这么用。

## 小结

综上所述，我们只要将 `MVC` 中的 `controller` 中的展示逻辑抽取出来，放置到 `viewModel` 中，然后通过一定的技术手段，比如 `RAC` 来同步 `view` 和 `viewModel` ，就完成了 `MVC` 到 `MVVM` 的转变。

> Talk is cheap. Show me the code.

下面，我们直接上代码，一起来看一个 `MVC` 模式转换成 `MVVM` 模式的示例。首先是 `model` 层的代码 `Person` ：

``` objc
@interface Person : NSObject

- (instancetype)initwithSalutation:(NSString *)salutation firstName:(NSString *)firstName lastName:(NSString *)lastName birthdate:(NSDate *)birthdate;

@property (nonatomic, copy, readonly) NSString *salutation;
@property (nonatomic, copy, readonly) NSString *firstName;
@property (nonatomic, copy, readonly) NSString *lastName;
@property (nonatomic, copy, readonly) NSDate *birthdate;

@end
```

然后是 `view` 层的代码 `PersonViewController` ，在 `viewDidLoad` 方法中，我们将 `Person` 中的属性进行一定的转换后，赋值给相应的 `view` 进行展示：

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

最终，`PersonViewController` 将会变得非常轻量级：

``` objc
- (void)viewDidLoad {
    [super viewDidLoad];

    self.nameLabel.text = self.viewModel.nameText;
    self.birthdateLabel.text = self.viewModel.birthdateText;
}
```

怎么样？其实 `MVVM` 并没有想像中的那么难吧，而且更重要的是它也没有破坏 `MVC` 的现有结构，只不过是移动了一些代码，仅此而已。好了，说了这么多，那 `MVVM` 相比 `MVC` 到底有哪些好处呢？我想，主要可以归纳为以下三点：

- 由于展示逻辑被抽取到了 `viewModel` 中，所以 `view` 中的代码将会变得非常轻量级；
- 由于 `viewModel` 中的代码是与 `UI` 无关的，所以它具有良好的可测试性；
- 对于一个封装了大量业务逻辑的 `model` 来说，改变它可能会比较困难，并且存在一定的风险。在这种场景下，`viewModel` 可以作为 `model` 的适配器使用，从而避免对 `model` 进行较大的改动。

通过前面的示例，我们对第一点已经有了一定的感触；至于第三点，可能对于一个复杂的大型应用来说，才会比较明显；下面，我们还是使用前面的示例，来直观地感受下第二点好处：

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

对于 `MVVM` 来说，我们可以把 `view` 看作是 `viewModel` 的可视化形式，`viewModel` 提供了 `view` 所需的数据和命令。因此，`viewModel` 的可测试性可以帮助我们极大地提高应用的质量。

## MVVMReactiveCocoa

接下来，我们进入本文的第二部分，重点介绍一个使用 `MVVM` 和 `RAC` 开发的真实应用，`MVVMReactiveCocoa` 。说明，本文将主要介绍这个应用的架构和设计思路，希望可以为你实践 `MVVM` 提供一个真实的参考案例，有些架构并非是 `MVVM` 模式所必须的，而是我们为了更好更顺畅地使用 `MVVM` 而引入的，尤其是 `ViewModel-Based Navigation` ，所以请你在实践的时候能够结合自身应用的实际情况做出相应的取舍，灵活处理，活学活用。最后，我们将以登录界面为具体的例子，一起探讨一下 `MVVM` 的实践过程。

**说明**，以下内容均基于 `MVVMReactiveCocoa` 的 [v2.1.1](https://github.com/leichunfeng/MVVMReactiveCocoa/tree/v2.1.1) 标签进行展开，部分代码有删减。

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

<img src="http://localhost:4000/images/service-bus.png" width="558" />

从上图中，我们可以看出，在服务总线类 `MRCViewModelServices/MRCViewModelServicesImpl` 中，主要包括以下三个方面的内容：

- 应用自有的服务类，用柚黄色进行了标识，包括 `MRCAppStoreService/MRCAppStoreServiceImpl` 和 `MRCRepositoryService/MRCRepositoryServiceImpl` 两个服务类；
- 第三方 `GitHub` 提供的官方 `API` 框架，用天蓝色进行了标识，主要包括 `OCTClient` 服务类；
- 应用的导航服务，用藻绿色进行了标识，包括 `MRCNavigationProtocol` 协议、虚拟实现类 `MRCViewModelServicesImpl` 等。

其中，前面两者都是以信号的形式对 `viewModel` 层提供服务，代表异步的网络请求等数据获取操作，而我们在 `viewModel` 则可以通过订阅信号的形式获取到所需的数据。此外，服务总线还实现了 `MRCNavigationProtocol` 协议，协议的内容如下：

``` objc
@protocol MRCNavigationProtocol <NSObject>

- (void)pushViewModel:(MRCViewModel *)viewModel animated:(BOOL)animated;

- (void)popViewModelAnimated:(BOOL)animated;

- (void)popToRootViewModelAnimated:(BOOL)animated;

- (void)presentViewModel:(MRCViewModel *)viewModel animated:(BOOL)animated completion:(VoidBlock)completion;

- (void)dismissViewModelAnimated:(BOOL)animated completion:(VoidBlock)completion;

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

那么，我们是怎么实现以 `viewModel` 为基础的导航操作的呢？用 `MRCViewModelServicesImpl` 来实现这些空操作到底有什么用意？为什么要这么做，目的是为了什么？兄台，莫急，请接着看下一小节的内容。

### ViewModel-Based Navigation

首先，我们先来思考一个问题，就是我们为什么要实现 `ViewModel-Based` 的导航操作呢？直接在 `view` 层使用系统的 `push/present` 等操作来完成导航不就好了么？我总结了一下这么做的理由，主要有以下三点：

- 从理论上来说，`MVVM` 模式的应用应该是以 `viewModel` 为驱动来运转的；
- 根据我们前面对 `MVVM` 的探讨，`viewModel` 提供了 `view` 所需的全部数据和命令。因此，通常来说，我们可以直接在命令执行成功后使用 `doNext` 顺带就把导航操作给做了，一气呵成；
- 使 `view` 更加轻量级，只需要绑定 `viewModel` 提供的数据和命令即可。

既然如此，那我们究竟要如何实现 `ViewModel-Based` 的导航操作呢？我们都知道 `iOS` 中的导航操作无外乎两种，`push/pop` 和 `present/dismiss` ，前者是 `UINavigationController` 特有的功能，而后者是所有 `UIViewController` 都具备的功能。另外，值得一提的是 `UINavigationController` 也是 `UIViewController` 的子类，所以它也同样具备 `present/dismiss` 的功能。因此，从本质上来说，不管我们要实现什么样的导航操作，最终都是离不开 `push/pop` 和 `present/dismiss` 的。

事实上，目前的做法是在 `view` 层维护了一个 `NavigationController` 的堆栈 `MRCNavigationControllerStack` ，不管是 `push/pop` 还是 `present/dismiss` ，都是使用栈顶的 `NavigationController` 来执行操作的。

接下来，我们一起来看看 `MVVMReactiveCocoa` 在执行了 `push/pop` 或 `present/dismiss` 操作后视图层次结构的变化过程。首先，我们来看看用户在登录成功后进入到首页时的视图层次结构图：

<img src="http://localhost:4000/images/view-model-based1.png" width="671" />

此时，应用展示的界面是 `NewsViewController` 。同时，在 `MRCNavigationControllerStack` 堆栈中只有 `NavigationController0` 一个元素；而 `NavigationController1` 并没有在 `MRCNavigationControllerStack` 堆栈中，这是因为需要支持 `TabBarController` 的滑动切换而设计的视图层次结构，是首页比较特殊的一个地方。更多信息可以查看 `GitHub` 开源库 [WXTabBarController](https://github.com/leichunfeng/WXTabBarController) ，在这里，我们并不需要太过于关心这个问题，因为真正重要的是原理。

接下来，当用户在 `NewsViewController` 界面，点击了某一个 `cell` ，通过 `push` 的方式，进入到仓库详情界面时，应用的视图层次结构图如下：

<img src="http://localhost:4000/images/view-model-based2.png" width="184" />

应用通过 `MRCNavigationControllerStack` 栈顶的元素 `NavigationController0` ，将仓库详情界面 `push` 到自身的内部堆栈中。此时，应用展示的界面是被 `push` 进来的仓库详情界面 `RepoDetailViewController` 。最后，当用户在仓库详情界面，点击左下角的分支按钮，通过 `present` 的方式，弹出分支选择界面时，应用的视图层次结构图如下：

<img src="http://localhost:4000/images/view-model-based3.png" width="454" />

另外，`pop` 和 `dismiss` 为 `push` 和 `present` 的逆操作，只要反过来看上面的视图层次构图变化过程即可，这里不再赘述。

等等，如果我没有记错的话，你现在说的 `MRCNavigationControllerStack` 堆栈是在 `view` 层，而服务总线类 `MRCViewModelServicesImpl` 是在 `viewModel` 的。据我所知，`MVVM` 中的 `viewModel` 层是不能引入 `view` 层的任何东西的，更严格的说，是不能引入任何 `UIKit` 中的东西的，不然就违背了 `MVVM` 模式的基本原则，并且 `viewModel` 的可测试性也将随之散失。

是的，这里就是 `MRCViewModelServicesImpl` 中之所以实现那些空的导航操作的真正目的所在了。`viewModel` 通过调用 `MRCViewModelServicesImpl` 中的空操作来表明需要执行相应的导航操作，而 `MRCNavigationControllerStack` 中则通过 `Hook` 来捕获这些空操作，然后使用栈顶的 `NavigationController` 来执行真正的导航操作：

``` objc
- (void)registerNavigationHooks {
    @weakify(self)
    [[(NSObject *)self.services
        rac_signalForSelector:@selector(pushViewModel:animated:)]
        subscribeNext:^(RACTuple *tuple) {
            @strongify(self)
            UIViewController *viewController = (UIViewController *)[MRCRouter.sharedInstance viewControllerForViewModel:tuple.first];
            [self.navigationControllers.lastObject pushViewController:viewController animated:[tuple.second boolValue]];
        }];

    [[(NSObject *)self.services
        rac_signalForSelector:@selector(popViewModelAnimated:)]
        subscribeNext:^(RACTuple *tuple) {
        	@strongify(self)
            [self.navigationControllers.lastObject popViewControllerAnimated:[tuple.first boolValue]];
        }];

    [[(NSObject *)self.services
        rac_signalForSelector:@selector(popToRootViewModelAnimated:)]
        subscribeNext:^(RACTuple *tuple) {
            @strongify(self)
            [self.navigationControllers.lastObject popToRootViewControllerAnimated:[tuple.first boolValue]];
        }];

    [[(NSObject *)self.services
        rac_signalForSelector:@selector(presentViewModel:animated:completion:)]
        subscribeNext:^(RACTuple *tuple) {
        	@strongify(self)
            UIViewController *viewController = (UIViewController *)[MRCRouter.sharedInstance viewControllerForViewModel:tuple.first];

            UINavigationController *presentingViewController = self.navigationControllers.lastObject;
            if (![viewController isKindOfClass:UINavigationController.class]) {
                viewController = [[MRCNavigationController alloc] initWithRootViewController:viewController];
            }
            [self pushNavigationController:(UINavigationController *)viewController];

            [presentingViewController presentViewController:viewController animated:[tuple.second boolValue] completion:tuple.third];
        }];

    [[(NSObject *)self.services
        rac_signalForSelector:@selector(dismissViewModelAnimated:completion:)]
        subscribeNext:^(RACTuple *tuple) {
            @strongify(self)
            [self popNavigationController];
            [self.navigationControllers.lastObject dismissViewControllerAnimated:[tuple.first boolValue] completion:tuple.second];
        }];

    [[(NSObject *)self.services
        rac_signalForSelector:@selector(resetRootViewModel:)]
        subscribeNext:^(RACTuple *tuple) {
            @strongify(self)
            [self.navigationControllers removeAllObjects];

            UIViewController *viewController = (UIViewController *)[MRCRouter.sharedInstance viewControllerForViewModel:tuple.first];

            if (![viewController isKindOfClass:[UINavigationController class]]) {
                viewController = [[MRCNavigationController alloc] initWithRootViewController:viewController];
                ((UINavigationController *)viewController).delegate = self;
                [self pushNavigationController:(UINavigationController *)viewController];
            }

            MRCSharedAppDelegate.window.rootViewController = viewController;
        }];
}
```

通过 `Hook` 的方式，我们最终实现了 `viewModel-based` 的导航操作，并且在 `viewModel` 中也没有引入 `view` 层的任意东西，实现了解耦合。

#### Router

另外，还有一点值得一提的是，我们在 `viewModel` 层调用导航操作的时候，只传入了 `viewModel` 的实例作为参数，那么在 `MRCNavigationControllerStack` 堆栈中去真正执行导航操作的时候，我们怎么才能知道要跳转到哪个界面呢？因此，我们就需要配置 `viewModel` 到 `view` 的映射了，并且约定好一个统一的初始化 `view` 的方法 `initWithViewModel:` ，这些也是必不可少的条件：

``` objc
- (MRCViewController *)viewControllerForViewModel:(MRCViewModel *)viewModel {
    NSString *viewController = self.viewModelViewMappings[NSStringFromClass(viewModel.class)];
    
    NSParameterAssert([NSClassFromString(viewController) isSubclassOfClass:[MRCViewController class]]);
    NSParameterAssert([NSClassFromString(viewController) instancesRespondToSelector:@selector(initWithViewModel:)]);
    
    return [[NSClassFromString(viewController) alloc] initWithViewModel:viewModel];
}

- (NSDictionary *)viewModelViewMappings {
    return @{
    	@"MRCLoginViewModel": @"MRCLoginViewController",
        @"MRCHomepageViewModel": @"MRCHomepageViewController",
        @"MRCRepoDetailViewModel": @"MRCRepoDetailViewController",
        ...
    };
}
```

### 登录界面

接下来，我们一起来看一下 `MVVMReactiveCocoa` 中登录界面的 `viewModel` 和 `view` 的部分关键代码，探讨一下 `MVVM` 模式的具体实践过程。**说明**，我们将会尽可能地避开具体的业务逻辑，重点关注 `MVVM` 模式的实践思路。下面是登录界面的截图：

<img src="http://localhost:4000/images/login.jpg" width="320" />

其中，主要的界面元素有：

- 一个用于展示用户头像的按钮 `avatarButton` ，因为需要点击查看大图，所以直接用的按钮；
- 用于输入账号和密码的输入框 `usernameTextField` 和 `passwordTextField` ；
- 一个直接登录的按钮 `loginButton` 和一个跳转到浏览器授权登录的按钮 `browserLoginButton` 。

**分析**：根据我们前面对 `MVVM` 的探讨，`viewModel` 需要提供 `view` 所需的全部数据和命令。因此，`MRCLoginViewModel.h` 头文件的内容大致如下：

``` objc
@interface MRCLoginViewModel : MRCViewModel

@property (nonatomic, copy, readonly) NSURL *avatarURL;
@property (nonatomic, copy) NSString *username;
@property (nonatomic, copy) NSString *password;

@property (nonatomic, strong, readonly) RACSignal *validLoginSignal;
@property (nonatomic, strong, readonly) RACCommand *loginCommand;
@property (nonatomic, strong, readonly) RACCommand *browserLoginCommand;

@end
```

非常直观，其中需要特别说明的是 `validLoginSignal` 属性代表的是登录按钮是否可用，它将会与 `view` 中登录按钮的 `enabled` 属性进行绑定。 接着，我们再来看看 `MRCLoginViewModel` 的实现文件中的部分关键代码：

``` objc
@implementation MRCLoginViewModel

- (void)initialize {
    [super initialize];
    
    RAC(self, avatarURL) = [[RACObserve(self, username)
        map:^(NSString *username) {
            return [[OCTUser mrc_fetchUserWithRawLogin:username] avatarURL];
        }]
        distinctUntilChanged];
    
    self.validLoginSignal = [[RACSignal
    	combineLatest:@[ RACObserve(self, username), RACObserve(self, password) ]
        reduce:^(NSString *username, NSString *password) {
        	return @(username.length > 0 && password.length > 0);
        }]
        distinctUntilChanged];
    
    @weakify(self)
    void (^doNext)(OCTClient *) = ^(OCTClient *authenticatedClient) {
        @strongify(self)
        MRCHomepageViewModel *viewModel = [[MRCHomepageViewModel alloc] initWithServices:self.services params:nil];
        dispatch_async(dispatch_get_main_queue(), ^{
            [self.services resetRootViewModel:viewModel];
        });
    };
    
    [OCTClient setClientID:MRC_CLIENT_ID clientSecret:MRC_CLIENT_SECRET];
    
    self.loginCommand = [[RACCommand alloc] initWithSignalBlock:^(NSString *oneTimePassword) {
    	@strongify(self)
        OCTUser *user = [OCTUser userWithRawLogin:self.username server:OCTServer.dotComServer];
        return [[OCTClient
        	signInAsUser:user password:self.password oneTimePassword:oneTimePassword scopes:OCTClientAuthorizationScopesUser | OCTClientAuthorizationScopesRepository note:nil noteURL:nil fingerprint:nil]
            doNext:doNext];
    }];

    self.browserLoginCommand = [[RACCommand alloc] initWithSignalBlock:^(id input) {
        return [[OCTClient
        	signInToServerUsingWebBrowser:OCTServer.dotComServer scopes:OCTClientAuthorizationScopesUser | OCTClientAuthorizationScopesRepository]
            doNext:doNext];
    }];    
}

@end
```

- 当用户输入的用户名发生变化时，调用 `model` 层的方法查询本地数据库中缓存的用户表数据，然后返回 `avatarURL` 属性;
- 当用户输入的用户名或者密码发生变化时，判断用户名和密码的长度是否均大于 `0` ，如果是则登录按钮可用，否则不可用;
- 当 `loginCommand` 或者 `browserLoginCommand` 命令执行成功时，调用 `doNext` 代码块，调用服务总线中的方法 `resetRootViewModel`: 进入首页。

接下来，我们来看看 `MRCLoginViewController` 中的部分关键代码：

``` objc
@implementation MRCLoginViewController

- (void)bindViewModel {
    [super bindViewModel];
    
    @weakify(self)
    [RACObserve(self.viewModel, avatarURL) subscribeNext:^(NSURL *avatarURL) {
    	@strongify(self)
        [self.avatarButton sd_setImageWithURL:avatarURL forState:UIControlStateNormal placeholderImage:[UIImage imageNamed:@"default-avatar"]];
    }];
    
    RAC(self.viewModel, username)  = self.usernameTextField.rac_textSignal;
    RAC(self.viewModel, password)  = self.passwordTextField.rac_textSignal;
    RAC(self.loginButton, enabled) = self.viewModel.validLoginSignal;
    
    [[self.loginButton
        rac_signalForControlEvents:UIControlEventTouchUpInside]
        subscribeNext:^(id x) {
            @strongify(self)
            [self.viewModel.loginCommand execute:nil];
        }];
    
    [[self.browserLoginButton
        rac_signalForControlEvents:UIControlEventTouchUpInside]
        subscribeNext:^(id x) {
            @strongify(self)
            NSString *message = [NSString stringWithFormat:@"“%@” wants to open “Safari”", MRC_APP_NAME];
            
            UIAlertController *alertController = [UIAlertController alertControllerWithTitle:nil
                                                                                     message:message
                                                                              preferredStyle:UIAlertControllerStyleAlert];
            
            [alertController addAction:[UIAlertAction actionWithTitle:@"Cancel" style:UIAlertActionStyleCancel handler:NULL]];
            [alertController addAction:[UIAlertAction actionWithTitle:@"Open" style:UIAlertActionStyleDefault handler:^(UIAlertAction *action) {
                @strongify(self)
                [self.viewModel.browserLoginCommand execute:nil];
            }]];
            
            [self presentViewController:alertController animated:YES completion:NULL];
        }];
}

@end
```

- 观察 `viewModel` 中 `avatarURL` 属性的变化，然后设置 `avatarButton` 中的图片；
- 将 `viewModel` 中的 `username` 和 `password` 属性分别与 `usernameTextField` 和 `passwordTextField` 输入框中的内容进行绑定；
- 将 `loginButton` 的 `enabled` 属性与 `viewModel` 的 `validLoginSignal` 属性进行绑定；
- 在 `loginButton` 和 `browserLoginButton` 按钮被点击时分别执行 `loginCommand` 和 `browserLoginCommand` 命令。

综上所述，我们将 `MRCLoginViewController` 中的展示逻辑抽取到 `MRCLoginViewModel` 中后，使得 `MRCLoginViewController` 中的代码更加简洁和清晰。其中，实践 `MVVM` 的关键点在于，我们需要分析清楚 `viewModel` 需要暴露给 `view` 的数据和命令，这些数据和命令要能足够用来表示 `view` 当前的状态。

## 总结

我们从理论出发介绍了 `MVC` 和 `MVVM` 各自的概念以及从 `MVC` 到 `MVVM` 的演进过程，接着介绍了 `RAC` 在 `MVVM` 中的两个使用场景，最后，我们从实践的角度，重点介绍了一个使用 `MVVM` 和 `RAC` 开发的开源项目 `MVVMReactiveCocoa` 。总得来说，我认为 `iOS` 中的 `MVVM` 模式可以分为以下三种不同的实践程度，它们分别对应不同的适用场景：

- `MVVM + KVO` ，适用于现有的 `MVC` 项目，想转换成 `MVVM` 但是不打算引入 `RAC` 作为 `bivder` 的团队；
- `MVVM + RAC` ，适用于现有的 `MVC` 项目，想转换成 `MVVM` 并且打算引入 `RAC` 作为 `binder` 的团队；
- `MVVM + RAC + ViewModel-Based Navigation` ，适用于全新的项目，想实践 `MVVM` 并且打算引入 `RAC` 作为 `binder` ，然后也想实践 `ViewModel-Based Navigation` 的团队。

最后，希望这篇文章能够打消你对 `MVVM` 模式的顾虑，还等什么，赶紧行动起来吧。

## 参考链接

[https://www.objc.io/issues/13-architecture/mvvm/](https://www.objc.io/issues/13-architecture/mvvm/)
[https://msdn.microsoft.com/en-us/library/hh848246.aspx](https://msdn.microsoft.com/en-us/library/hh848246.aspx)
[https://en.wikipedia.org/wiki/Model%E2%80%93view%E2%80%93viewmodel](https://en.wikipedia.org/wiki/Model%E2%80%93view%E2%80%93viewmodel)
[https://medium.com/ios-os-x-development/ios-architecture-patterns-ecba4c38de52#.p6n56kyc4](https://medium.com/ios-os-x-development/ios-architecture-patterns-ecba4c38de52#.p6n56kyc4)
[http://cocoasamurai.blogspot.ru/2013/03/basic-mvvm-with-reactivecocoa.html](http://cocoasamurai.blogspot.ru/2013/03/basic-mvvm-with-reactivecocoa.html)
[http://www.sprynthesis.com/2014/12/06/reactivecocoa-mvvm-introduction/](http://www.sprynthesis.com/2014/12/06/reactivecocoa-mvvm-introduction/)

<img src="http://blog.leichunfeng.com/images/wechat_pay.jpg" width="260" />


