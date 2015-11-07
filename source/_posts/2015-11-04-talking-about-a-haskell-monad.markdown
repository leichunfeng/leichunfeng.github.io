---
layout: post
title: "谈谈 Haskell 中的 Monad"
date: 2015-11-04 21:35:56 +0800
comments: true
categories: 
---

在函数式编程语言中有一个非常重要的概念，那就是 Monad 。Monad 的概念本身是非常简单的，但是由于它的高度抽象化，所以如果我们只从理论的角度去理解它是非常困难的。因此，本文将采用理论结合实践的方式来诠释 Monad 的概念，相信我，它真的非常简单。

**注**：本文将一步步的带领读者从搭建 Haskell 的集成开发环境开始到最终完成一个使用 Monad 的小例子，以此来让读者充分地理解 Monad 的概念。事不宜迟，咱们开干吧。

本文的目录结构如下：

- 开始
    + 搭建环境
    + Hello world
- Type
    + Maybe
- Typeclass
    + Functor
    + Applicative
    + Monad
- 例子
- 总结

## 开始

Haskell 是一门纯函数式的编程语言，是世界上公认的语法最优美最简洁的一种语言。在命令式编程语言中，我们通过让计算机执行一系列命令让完成我们的任务；而在纯函数式编程语言中，我们只需要向计算机描述清楚我们的任务是什么。

### 搭建环境

在正式开始编写 Haskell 代码前，我们需要搭建 Haskell 的集成开发环境，只需要以下两样东西：

1. 一个文本编辑器；
2. 一个 Haskell 编译器。

文本编辑器自然不用多说，相信你已经有了自己喜欢的文本编辑器，比如我就非常钟爱 Sublime Text ；另外，在本文中，我们将使用 GHC 来作为我们的编译器，它是使用最广泛的 Haskell 编译器。安装 GHC 最好的方式是直接下载安装 [Haskell Platform](https://www.haskell.org/platform/) ，根据操作步骤安装即可。

安装好 Haskell Platform 后，打开终端，输入 `ghci` ，你将会看到类似下面这样的输出：

``` objc
➜  talkingaboutahaskellmonad git:(master) ghci
GHCi, version 7.10.2: http://www.haskell.org/ghc/  :? for help
Prelude>
```

至此，你已经成功搭建了 Haskell 的集成开发环境。

### Hello world

根据传统，在学习一门新的编程语言时，要编写的第一个程序就应该是在屏幕上打印 `Hello, world!` ，自然我们也不另外。

- 新建一个文件，命名为 hello_world.hs ，在文件中输入以下 Haskell 代码，定义一个名为 printHelloWorld 的函数：

``` objc
printHelloWorld = "Hello, world!"
```

- 使用 :l 命令加载 hello_world.hs 文件：

``` objc
Prelude> :l hello_world.hs
[1 of 1] Compiling Main             ( hello_world.hs, interpreted )
Ok, modules loaded: Main.
*Main>
```

- 加载好 hello_world.hs 后，我们就可以使用 hello_world.hs 中定义的函数了：

``` objc
*Main> printHelloWorld
"Hello, world!"
*Main>
```

恭喜你，你已经成功入门 Haskell 了。

## Type

### Maybe

接下来，我们介绍一种在 Haskell 中非常有意思的数据类型，Maybe 类型，下面是它的定义：

``` objc
data Maybe a = Nothing | Just a
```

前面的 data 是 Haskell 中的关键字，用来定义一种新的类型，具体的语法细节我们不需要太过于深究，因为这并不会影响我们对 Maybe 类型的理解和使用。

Maybe 类型封装了一个可选值。一个 Maybe a 类型的值要么包含一个 a 类型的值，用 Just a 表示；要么为空，用 Nothing 表示。那么 Maybe 类型有什么用处呢？在 Haskell 中，我们有时候可以使用 Maybe 类型来进行错误处理，用 Nothing 来表示程序出错，而不是使用更严格的方式，比如 errors 等。

另外，Maybe a 中的 a 是类型参数，表示任意的数据类型，比如当 a 为 Int 时，Maybe a 为 Maybe Int ；同样，当 a 为 String 时，Maybe a 为 Maybe String 。

## Typeclass

### Functor

下面是 Functor 的定义，它声明了一个 fmap 方法。因此，一个数据类型要实现 Functor ，则必须实现 fmap 方法：

``` objc
class Functor f where
    fmap :: (a -> b) -> f a -> f b
```

从 fmap 的方法签名，我们可以知道，它接受两个参数，第一个参数是一个从 a 到 b 的函数，第二个参数是一个 Functor 类型的值 f a ，然后返回另一个 Functor 类型的值 f b 。

### Applicative

``` objc
class Functor f => Applicative f where
    pure :: a -> f a
    (<*>) :: f (a -> b) -> f a -> f b
```

### Monad

``` objc
class Applicative m => Monad m where
    return :: a -> m a
    (>>=) :: m a -> (a -> m b) -> m b
```

## 例子

## 总结

## 参考链接

[http://learnyouahaskell.com/chapters](http://learnyouahaskell.com/chapters)
<br>
[https://downloads.haskell.org/~ghc/latest/docs/html/libraries](https://downloads.haskell.org/~ghc/latest/docs/html/libraries)
<br>
[http://adit.io/posts/2013-04-17-functors,_applicatives,_and_monads_in_pictures.html](http://adit.io/posts/2013-04-17-functors,_applicatives,_and_monads_in_pictures.html)

<img src="/images/wechat_pay.jpg" width="250" />
