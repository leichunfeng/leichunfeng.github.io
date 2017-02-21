---
layout: post
title: "谈谈 iOS 中图片的解压缩"
date: 2017-02-20 10:47:56 +0800
comments: true
categories: 
---

对于大多数 iOS 应用来说，图片往往是最占用手机内存的资源之一，同时也是不可或缺的组成部分。将一张图片从磁盘中加载出来，并最终显示到屏幕上，中间经过了一系列复杂的处理过程，其中就包括了对图片的解压缩。

## 图片加载的工作流

概括来说，从磁盘中加载一张图片，并将它显示到屏幕上，中间的[主要工作流](https://github.com/path/FastImageCache#the-problem)如下：

1. 假设我们使用 `+imageWithContentsOfFile:` 方法从磁盘中加载一张图片，这个时候的图片并没有解压缩；
2. 然后将生成的 `UIImage` 赋值给 `UIImageView` ；
3. 接着一个隐式的 `CATransaction` 捕获到了 `UIImageView` 图层树的变化；
4. 在主线程的下一个 run loop 到来时，Core Animation 提交了这个隐式的 transaction ，这个过程可能会对图片进行 copy 操作，而受图片是否**字节对齐**等因素的影响，这个 copy 操作可能会涉及以下部分或全部步骤：
	5. 分配内存缓冲区用于管理文件 IO 和解压缩操作；
	6. 将文件数据从磁盘读到内存中；
	7. 将压缩的图片数据解码成未压缩的位图形式，这是一个非常耗时的 CPU 操作；
	8. 最后 Core Animation 使用未压缩的位图数据渲染 `UIImageView` 的图层。

在上面的步骤中，我们提到了图片的解压缩是一个非常耗时的 CPU 操作，并且它默认是在主线程中执行的。那么当需要加载的图片比较多时，就会对我们应用的响应性造成严重的影响，尤其是在快速滑动的列表上，这个问题会表现得更加突出。

## 为什么需要解压缩

既然图片的解压缩需要消耗大量的 CPU 时间，那么我们为什么还要对图片进行解压缩呢？是否可以不经过解压缩，而直接将图片显示到屏幕上呢？答案是否定的。要想弄明白这个问题，我们首先需要知道什么是[位图](https://developer.apple.com/library/content/documentation/GraphicsImaging/Conceptual/drawingwithquartz2d/dq_images/dq_images.html#//apple_ref/doc/uid/TP30001066-CH212-SW3)：

> A bitmap image (or sampled image) is an array of pixels (or samples). Each pixel represents a single point in the image. JPEG, TIFF, and PNG graphics files are examples of bitmap images.

其实，位图就是一个像素数组，数组中的每个像素就代表着图片中的一个点。我们在应用中经常用到的 JPEG 和 PNG 图片就是位图。下面，我们来看一个具体的例子，这是一张 PNG 图片，像素为 30 × 30 ，文件大小为 843B ：

![位图](http://localhost:4000/images/check_green.png)

我们使用[下面的代码](https://developer.apple.com/library/content/qa/qa1509/_index.html)：

``` objc
UIImage *image = [UIImage imageNamed:@"check_green"];    
CFDataRef rawData = CGDataProviderCopyData(CGImageGetDataProvider(image.CGImage));
```

就可以获取到这个图片的原始像素数据，大小为 3600B ：

```
00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000
01020102 032c023c 0567048c 078d06bf 08a006d9 09b307f3 09b307f3 08a006d9 078d06bf
0567048c 032c023c 01020102 00000000 00000000 00000000 00000000 00000000 00000000
00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000
00000000 01060108 05570476 09ab07e9 09bb07ff 09bb07ff 09bb07ff 09bb07ff 09bb07ff
09bb07ff 09bb07ff 09bb07ff 09bb07ff 09bb07ff 09ab07e9 05570476 01060108 00000000
00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000
00000000 00000000 00000000 033d0353 08a607e2 09bb07ff 09bb07ff 09bb07ff 09bb07ff
... 
09bb07ff 09bb07ff 09bb07ff 09bb07ff 08a607e2 033d0353 00000000 00000000 00000000
00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000
00000000 01060108 05570476 09ab07e9 09bb07ff 09bb07ff 09bb07ff 09bb07ff 09bb07ff
09bb07ff 09bb07ff 09bb07ff 09bb07ff 09bb07ff 09ab07e9 05570476 01060108 00000000
00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000
00000000 00000000 00000000 00000000 00000000 00000000 01020102 032c023c 0567048c
078d06bf 08a006d9 09b307f3 09b307f3 08a006d9 078d06bf 0567048c 032c023c 01020102
00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000
```

也就是说，这张文件大小为 843B 的 PNG 图片解压缩后的大小是 3600B ，是原始文件大小的 4.27 倍。那么这个 3600B 是怎么得来的呢？与图片的文件大小或者像素有什么必然的联系吗？事实上，解压缩后的图片大小与原始文件大小之间没有任何关系，而只与图片的像素有关：

``` objc
解压缩后的图片大小 = 图片的像素宽 30 * 图片的像素高 30 * 每个像素所占的字节数 4
```

至于这个公式是怎么得来的，我们后面会有详细的说明，现在只需要知道即可。

至此，我们已经知道了什么是位图，并且直观地看到了它的原始像素数据，那么它与我们经常提到的图片的二进制数据有什么联系吗？是同一个东西吗？事实上，这二者是完全独立的两个东西，它们之间没有必然的联系。为了加深理解，我把这个图片拖进 Sublime Text 2 中，得到了这个图片的二进制数据，大小与原始文件大小一致，为 843B ：

```
8950 4e47 0d0a 1a0a 0000 000d 4948 4452 0000 001e 0000 001e 0806 0000 003b 30ae a200
0000 0173 5247 4200 aece 1ce9 0000 0305 4944 4154 480d c557 4d68 1341 149e 3709 da4d
09c6 8a56 2385 9e14 f458 4fa2 d092 f4a6 28d8 2222 de04 3d09 a1d0 7a50 0954 8bad 2d05
4fde 3c89 482b 2ad6 8334 d183 e049 ef9e 4a41 48b0 42eb a549 6893 1ddf 9bcd b4d9 d9d9
4dd8 a43a b0d9 9d79 3fdf bc79 3ff3 02ac 8591 1559 3e97 9b3e 5b05 fb32 6330 c098 48a2
183d 340a b886 8ff8 1e15 fced 587a e26b 16b2 b643 f2ff 057f 1263 fd9f fbbb 7ed7 7edd
1142 8c09 268e 04f1 2a1a 3058 0380 b9c3 91de a7ab 43ab 15b5 aebf 7d81 ad65 eb0a 5a31
8f4f 9f2e d4da 1c7e e249 64ca c3e5 d726 7eae 2fa2 7510 cb75 3d62 cc5e 0c0f 4a5a 69c3
...
36ac b11e 7006 f71b 5386 a2b7 1e48 ad82 a26a 2880 95db 3f8b f525 b880 e0ed 7221 75f1
fa02 2cd4 1af7 1d0e 546a 98e5 d4ae 342a 337e 6b96 134f 1ba0 0c0b c83b a0f2 3593 7b5c
6ca9 b541 cb4f 254e df58 d958 8955 a0fc 2638 658c 2660 f986 b5f1 f4dd 63f2 5aec ce59
e3b6 b0a7 cdac ee55 145c c7dc 8f60 f53f e0a6 b436 e3c0 27b0 8ecf 5054 336a ccd0 e1d8
2335 1f78 323d 6141 09c3 c1aa 5f8b 4e37 0899 e6b0 ed72 4046 759e d262 5247 9d01 1689
a976 55fb c993 6ed5 7d10 8ff4 b162 fe6f cd1e ee4a d4bb c18e 594e 96ea 1da6 c762 6539
bdff 7943 afc0 c91f bdd1 a327 28fc 29f7 d47a b337 f192 0cc9 36fa 5497 73f9 5827 aa39
1599 4eff 69fb 0b0d 1f7a 96cd 3eb0 7800 0000 0049 454e 44ae 4260 82
```

事实上，不管是 JPEG 还是 PNG 图片，都是一种压缩的位图图形格式。PNG 图片是无损压缩，并且支持 Alpha 通道，而 JPEG 图片则是有损压缩，可以指定 0-100% 的压缩比。值得一提的是，在苹果的 SDK 中专门提供了两个函数用来生成 PNG 和 JPEG 图片：

``` objc
// return image as PNG. May return nil if image has no CGImageRef or invalid bitmap format
UIKIT_EXTERN NSData * __nullable UIImagePNGRepresentation(UIImage * __nonnull image);

// return image as JPEG. May return nil if image has no CGImageRef or invalid bitmap format. compression is 0(most)..1(least)                           
UIKIT_EXTERN NSData * __nullable UIImageJPEGRepresentation(UIImage * __nonnull image, CGFloat compressionQuality);
```

因此，在将磁盘中的图片渲染到屏幕之前，必须先要得到图片的原始像素数据，才能执行后续的绘制操作，这就是为什么需要对图片解压缩的原因。

## 强制解压缩

## 开源库的实现

## 性能对比

## 总结

## 参考链接


