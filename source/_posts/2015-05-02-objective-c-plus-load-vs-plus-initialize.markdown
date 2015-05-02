---
layout: post
title: "Objective-C +load VS +initialize"
date: 2015-05-02 17:06:15 +0800
comments: true
categories: 
keywords: Objective-C, +load, +initialize, runtime, NSObject
---

在上一篇博文[《Objective-C 对象模型》](http://blog.leichunfeng.com/blog/2015/04/25/objective-c-object-model/)中，我们知道了 Objective-C 中绝大部分的类都继承自 NSObject 类。而在 NSObject 类中有两个非常特殊的类方法 +load 和 +initialize ，用于类的初始化。这两个看似非常简单的类方法在许多方面会让人感到困惑，比如：

1. 子类、父类、分类中的相应方法什么时候会被调用？
2. 需不需要在子类的实现中显式地调用父类的实现？
3. 每个方法到底会被调用多少次？  

下面，我们将结合 [runtime](http://opensource.apple.com/tarballs/objc4/)（我下载的是当前的最新版本 `objc4-646.tar.gz`) 的源码，一起来揭开它们的神秘面纱。

## +load

+load 方法是当类或分类被添加到 Objective-C runtime 时被调用的，实现这个方法可以让我们在类加载的时候执行一些类相关的行为。子类的 +load 方法会在它的所有父类的 +load 方法之后执行，而分类的 +load 方法会在它的主类的 +load 方法之后执行。但是不同的类之间的 +load 方法的调用顺序是不确定的。

打开 runtime 工程，我们接下来看看与 +load 方法相关的几个关键函数。首先是文件 `objc-runtime-new.mm` 中的 `void prepare_load_methods(header_info *hi)` 函数：

``` objc
void prepare_load_methods(header_info *hi)
{
    size_t count, i;

    rwlock_assert_writing(&runtimeLock);

    classref_t *classlist = 
        _getObjc2NonlazyClassList(hi, &count);
    for (i = 0; i < count; i++) {
        schedule_class_load(remapClass(classlist[i]));
    }

    category_t **categorylist = _getObjc2NonlazyCategoryList(hi, &count);
    for (i = 0; i < count; i++) {
        category_t *cat = categorylist[i];
        Class cls = remapClass(cat->cls);
        if (!cls) continue;  // category for ignored weak-linked class
        realizeClass(cls);
        assert(cls->ISA()->isRealized());
        add_category_to_loadable_list(cat);
    }
}
```

顾名思义，这个函数的作用就是提前准备好满足 +load 方法调用条件的类和分类，以供接下来的调用。其中，在处理类时，调用了同文件中的另外一个函数 `static void schedule_class_load(Class cls)` 来执行具体的操作。

``` objc
static void schedule_class_load(Class cls)
{
    if (!cls) return;
    assert(cls->isRealized());  // _read_images should realize

    if (cls->data()->flags & RW_LOADED) return;

    // Ensure superclass-first ordering
    schedule_class_load(cls->superclass);

    add_class_to_loadable_list(cls);
    cls->setInfo(RW_LOADED); 
}
```

其中，函数第 9 行代码对入参的父类进行了递归调用，以确保父类优先的顺序。`void prepare_load_methods(header_info *hi)` 函数执行完后，当前所有满足 +load 方法调用条件的类和分类就被分别存放在全局变量 `loadable_classes` 和 `loadable_categories` 中了。

准备好类和分类后，接下来就是对它们的 +load 方法进行调用了。打开文件 `objc-loadmethod.m` ，找到其中的 `void call_load_methods(void)` 函数。

``` objc
void call_load_methods(void)
{
    static BOOL loading = NO;
    BOOL more_categories;

    recursive_mutex_assert_locked(&loadMethodLock);

    // Re-entrant calls do nothing; the outermost call will finish the job.
    if (loading) return;
    loading = YES;

    void *pool = objc_autoreleasePoolPush();

    do {
        // 1. Repeatedly call class +loads until there aren't any more
        while (loadable_classes_used > 0) {
            call_class_loads();
        }

        // 2. Call category +loads ONCE
        more_categories = call_category_loads();

        // 3. Run more +loads if there are classes OR more untried categories
    } while (loadable_classes_used > 0  ||  more_categories);

    objc_autoreleasePoolPop(pool);

    loading = NO;
}
```

同样的，这个函数的作用就是调用上一步准备好的类和分类中的 +load 方法，并且确保类优先于分类的顺序。我们继续查看在这个函数中调用的另外两个关键函数 `static void call_class_loads(void)` 和 `static BOOL call_category_loads(void)` 。由于这两个函数的作用大同小异，下面就以篇幅较小的 `static void call_class_loads(void)` 函数为例进行探讨。

``` objc
static void call_class_loads(void)
{
    int i;
    
    // Detach current loadable list.
    struct loadable_class *classes = loadable_classes;
    int used = loadable_classes_used;
    loadable_classes = nil;
    loadable_classes_allocated = 0;
    loadable_classes_used = 0;
    
    // Call all +loads for the detached list.
    for (i = 0; i < used; i++) {
        Class cls = classes[i].cls;
        load_method_t load_method = (load_method_t)classes[i].method;
        if (!cls) continue; 

        if (PrintLoading) {
            _objc_inform("LOAD: +[%s load]\n", cls->nameForLogging());
        }
        (*load_method)(cls, SEL_load);
    }
    
    // Destroy the detached list.
    if (classes) _free_internal(classes);
}
```

这个函数的作用就是真正负责调用类的 +load 方法了。它从全局变量 `loadable_classes` 中取出所有可供调用的类，并进行清零操作。

``` objc
loadable_classes = nil;
loadable_classes_allocated = 0;
loadable_classes_used = 0;
```

其中 `loadable_classes` 指向用于保存类信息的内存的首地址，`loadable_classes_allocated` 标识已分配的内存空间大小，`loadable_classes_used` 则标识已使用的内存空间大小。

然后，循环调用所有类的 +load 方法。**注意**，这里是（调用分类的 +load 方法也是如此）直接使用函数内存地址的方式 `(*load_method)(cls, SEL_load);` 对 +load 方法进行调用的，而不是使用发送消息 `objc_msgSend` 的方式。

这样的调用方式就使得 +load 方法拥有了一个非常有趣的特性，那就是子类、父类和分类中的 +load 方法的实现是被区别对待的。也就是说如果子类没有实现 +load 方法，那么当它被加载时 runtime 是不会去调用父类的 +load 方法的。同理，当一个类和它的分类都实现了 +load 方法时，两个方法都会被调用。因此，我们常常可以利用这个特性做一些“邪恶”的事情，比如说方法混淆（[Method Swizzling](http://nshipster.com/method-swizzling/)）。

## +initialize

+initialize 方法是在类或它的子类收到第一条消息之前被调用的，这里所指的消息包括实例方法和类方法的调用。也就是说 +initialize 方法是以懒加载的方式被调用的，如果程序一直没有给某个类或它的子类发送消息，那么这个类的 +initialize 方法是永远不会被调用的。那这样设计有什么好处呢？好处是显而易见的，那就是节省系统资源，避免浪费。

同样的，我们还是结合 runtime 的源码来加深对 +initialize 方法的理解。打开文件 `objc-runtime-new.mm` ，找到以下函数：

``` objc
IMP lookUpImpOrForward(Class cls, SEL sel, id inst, 
                       bool initialize, bool cache, bool resolver)
{
    ...
        rwlock_unlock_write(&runtimeLock);
    }

    if (initialize  &&  !cls->isInitialized()) {
        _class_initialize (_class_getNonMetaClass(cls, inst));
        // If sel == initialize, _class_initialize will send +initialize and 
        // then the messenger will send +initialize again after this 
        // procedure finishes. Of course, if this is not being called 
        // from the messenger then it won't happen. 2778172
    }

    // The lock is held to make method-lookup + cache-fill atomic 
    // with respect to method addition. Otherwise, a category could 
    ...
}
```

当我们给某个类发送消息时，runtime 会调用这个函数在类中查找相应方法的实现或进行消息转发。从第 8-14 的关键代码我们可以看出，当类没有初始化时 runtime 会调用 `void _class_initialize(Class cls)` 函数对该类进行初始化。

``` objc
void _class_initialize(Class cls)
{
    ...
    Class supercls;
    BOOL reallyInitialize = NO;

    // Make sure super is done initializing BEFORE beginning to initialize cls.
    // See note about deadlock above.
    supercls = cls->superclass;
    if (supercls  &&  !supercls->isInitialized()) {
        _class_initialize(supercls);
    }
    
    // Try to atomically set CLS_INITIALIZING.
    monitor_enter(&classInitLock);
    if (!cls->isInitialized() && !cls->isInitializing()) {
        cls->setInitializing();
        reallyInitialize = YES;
    }
    monitor_exit(&classInitLock);
    
    if (reallyInitialize) {
        // We successfully set the CLS_INITIALIZING bit. Initialize the class.
        
        // Record that we're initializing this class so we can message it.
        _setThisThreadIsInitializingClass(cls);
        
        // Send the +initialize message.
        // Note that +initialize is sent to the superclass (again) if 
        // this class doesn't implement +initialize. 2157218
        if (PrintInitializing) {
            _objc_inform("INITIALIZE: calling +[%s initialize]",
                         cls->nameForLogging());
        }

        ((void(*)(Class, SEL))objc_msgSend)(cls, SEL_initialize);

        if (PrintInitializing) {
            _objc_inform("INITIALIZE: finished +[%s initialize]",
    ...
}
```

其中，第 7-12 行代码对入参的父类进行了递归调用，以确保父类优先于子类初始化。另外，最关键的是第 36 行代码（暴露了 +initialize 方法的本质），runtime 使用了发送消息 `objc_msgSend` 的方式对 +initialize 方法进行调用。也就是说 +initialize 方法的调用与普通方法的调用是一样的，走的都是发送消息的流程。换言之，如果子类没有实现 +initialize 方法，那么继承自父类的实现会被调用；如果一个类的分类实现了 +initialize 方法，那么就会对这个类中的实现造成覆盖。

因此，如果一个子类没有实现 +initialize 方法，那么父类的实现是会被执行多次的。有时候，这可能是你想要的；但如果我们想确保自己的 +initialize 方法只执行一次，避免多次执行可能带来的副作用时，我们可以使用下面的代码来实现：

``` objc
+ (void)initialize {
  if (self == [ClassName self]) {
    // ... do the initialization ...
  }
}
```

## 总结

通过阅读 runtime 的源码，我们知道了 +load 和 +initialize 方法实现的细节，明白了它们的调用机制和各自的特点。下面我们绘制一张表格，以更加直观的方式来巩固我们对它们的理解：

<p>
    <table border="1" width="100%">
        <tr>
            <th></th>        
            <th>+load</th>        
            <th>+initialize</th>        
        </tr>
        <tr>
            <td>调用时机</td>        
            <td>被添加到 runtime 时</td>        
            <td>收到第一条消息前，可能永远不调用</td>        
        </tr>
        <tr>
            <td>调用顺序</td>        
            <td>父类->子类->分类</td>        
            <td>父类->子类</td>        
        </tr>
        <tr>
            <td>调用次数</td>        
            <td>1次</td>        
            <td>多次</td>        
        </tr>
        <tr>
            <td>是否需要显式调用父类实现</td>        
            <td>否</td>        
            <td>否</td>        
        </tr>
        <tr>
            <td>是否沿用父类的实现</td>        
            <td>否</td>        
            <td>是</td>        
        </tr>
        <tr>
            <td>分类中的实现</td>        
            <td>类和分类都执行</td>        
            <td>覆盖类中的方法，只执行分类的实现</td>        
        </tr>
    </table>
</p>

## 参考链接

- [https://www.mikeash.com/pyblog/friday-qa-2009-05-22-objective-c-class-loading-and-initialization.html](https://www.mikeash.com/pyblog/friday-qa-2009-05-22-objective-c-class-loading-and-initialization.html)
- [http://blog.iderzheng.com/objective-c-load-vs-initialize/](http://blog.iderzheng.com/objective-c-load-vs-initialize/)
- [http://nshipster.com/method-swizzling/](http://nshipster.com/method-swizzling/)
