---
layout:     post
title:      Instruments 使用与原理
subtitle:   
date:       2020-03-23
author:     LML
header-img: img/post-bg-ios9-web.jpg
catalog: true
tags:
    - Xcode 工具
       -  Instuments
---

# 前言
Instruments 能够帮助我们分析一些问题，比如 xcode 对image的加载过程，什么时候解码，什么时候IO等的分析（这里详细介绍）。本文主要介绍 Instruments 的使用，以及原理

# Instruments 内存相关

从苹果的开发者文档里可以看到，一个 app 的内存分三类：其中 Leaked memory 和 Abandoned memory 是需要我们分析注意的。

> Leaked memory: Memory unreferenced by your application that cannot be used again or freed (also detectable by using the Leaks instrument).   
	是指内存被分配了，但程序中已经没有指向该内存的指针，导致该内存无法被释放，产生内存泄漏。  

> Abandoned memory: Memory still referenced by your application that has no useful purpose.  
	在已分配内存的引用，但实际上程序中不会使用，比如图片等对象加入了缓存，但缓存中的对象一直没有被使用。 

> Cached memory: Memory still referenced by your application that might be used again for better performance.  


## Allocations  

+ 对于 Abandoned memory，可以用 Instrument 的 Allocations 检测出来。检测方法是用 Mark Generation 的方式，当你每次点击 Mark Generation 时，Allocations 会生成当前 App 的内存快照，而且 Allocations 会记录从上回内存快照到这次内存快照这个时间段内，新分配的内存信息。举一个最简单的例子：  
	+ 我们可以不断重复 push 和 pop 同一个 UIViewController，理论上来说，push 之前跟 pop 之后，app 会回到相同的状态。因此，在 push 过程中新分配的内存，在 pop 之后应该被 dealloc 掉，除了前几次 push 可能有预热数据和 cache 数据的情况。如果在数次 push 跟 pop 之后，内存还不断增长，则有内存泄露。因此，我们在每回 push 之前跟 pop 之后，都 Mark Generation 一下，以此观察内存是不是无限制增长。这个方法在 WWDC 的视频里：[Session 311 - Advanced Memory Analysis with Instruments](https://developer.apple.com/videos/all-videos/)，以及苹果的开发者文档：Finding Abandoned Memory 里有介绍。

### 使用
#### 使用详解
Xcode —— product——profile——选择Allocations——点击红色按钮——开始监控整个 App的内存使用情况:
Allocations打开的界面如下，主要有四个模块需要我们关注：  

![](https://pic.downk.cc/item/5eb4d4aac2a9a83be52fcb71.jpg)

+ 模块1:显示指定内存变化曲线。对 App 进行操作，这里会显示随着操作内存的变化曲线。且显示什么内存的曲线变化是可以在2初进行选择的，比如
	+ All Heap & Anonymous VM
	+ All Heap Allocations
	+ All VM regions
	+ 其他具体类型的内存变化
+ 模块2：Detail Pane 统计信息包含的类型：
	+ Statistics
		+ 勾选不同，模块1显示不通类型内存的曲线
		![](https://pic.downk.cc/item/5eb4d634c2a9a83be5310206.jpg)
	+ Call Trees
		+ 调用堆栈中各个方法消耗的内存，方便我们定位内存消耗大的方法
		+ 且可以根据模块3中的 callTree 决定是否显示系统的堆栈，可以先选择不显示初步定为内存消耗大的方法，然后在显示系统堆栈，进行详细定位  
		![](https://pic.downk.cc/item/5eb4d689c2a9a83be53142a6.jpg) 
	+ Allocations List

		![](https://pic.downk.cc/item/5eb4d762c2a9a83be5320a38.jpg)   

	+ Generations  
		+ 两次快照之间的内存增长，一般我们可以这样用
			+ 追查某个行为的内存增长：行为前后进行快照，得到内存增长，然后分析堆栈
			+ 追查某个页面or行为是否内存泄漏：进入页面前、进入后、离开分别快照，看是否内存恢复成进入前的水平
		![](https://pic.downk.cc/item/5eb4d798c2a9a83be5323dbf.jpg) 
		
+ 模块3：一些设定
+ 模块4:显示详细堆栈

#### 使用例子
用 Allocations 分析 -[UIImage imageNamed:] 图片加载流程（详细分析，这里）,加载Assets.car 里面的图片会造成两次内存增长
![](https://pic.downk.cc/item/5eb4dd53c2a9a83be5367057.jpg)  

+ read Assets.car过程，并建立mmap  
	+ 第一次 调用imageNamed的时候 
	+ 对应 Generation B

```  
- (void)clickButton{
    self.image = [UIImage imageNamed:@"bdd_card_share_image_container_dark"];
}  
```  
![](https://pic.downk.cc/item/5eb4de31c2a9a83be5371f22.jpg)

+ 即将展示图片的解码过程：延迟解码
![](https://pic.downk.cc/item/5eb4df04c2a9a83be537c5a1.jpg)

### 原理
Allocations 检测原理，目前猜测如下：
跟踪记录代码的执行堆栈，并记录  
+ 每个malloc的内存大小、类型
+ 每个执行的代码耗费的内存
这些信息在记录的时候很容易记录，然后作为原始数据，就可以进行展示，形成Allocations

### 缺点
+ 首先，你得打开 Allocations
+ 其次，你得一个个场景去重复的操作
+ 无法及时得知泄露，得专门做一遍上述操作，十分繁琐

## Leaks
> check for leaks—memory that has been allocated to objects that are no longer referenced and reachable.  
> Search a process's memory for unreferenced malloc buffers
> 苹果官方提供的。打开你的命令行，输入 man leaks 一份详细的介绍文档就出来了。

### 使用

### 原理  

> + leaks examines a specified process's memory for values that may be pointers to malloc-allocated buffers.  
> + Any buffer reachable from a pointer in writable global memory (e.g., __DATA segments), a register, or on the stack is assumed to be memory in use.  
> + Any buffer reachable from a pointer in reachable malloc-allocated buffer is also assumed to be in use.  
> + The buffers which are not reachable are leaks; the buffers could never be freed because no pointer exists in memory to the buffer, and thus free() could never be called for these buffers.  
> + Such buffers waste memory; removing them can reduce swapping and memory usage.  Leaks are particularly dangerous for long-running programs, for eventually the leaks could fill memory and cause the application to crash.  

这个工具的基本原理就是检测malloc的内存块是否被依然被引用。非malloc出来的内存块则无能为力。leaks搜索所有可能包含指向malloc内存块指针的内存区域，比如全局数据内存块，寄存器和所有的栈。如果malloc内存块的地址被直接或者间接引用，则是reachable的，反之，则是leaks。
     
### 存在的问题  
 Leaks 工具只负责检测 Leaked memory，而不管 Abandoned memory。在 MRC 时代 Leaked memory 很常见，因为很容易忘了调用 release，但在 ARC 时代更常见的内存泄露是循环引用导致的 Abandoned memory，Leaks 工具查不出这类内存泄露，应用有限。

## Zombies

### 使用

### 原理

## Xcode Memory Graph 

### 使用

### 原理

## MLeaksFinder
一般查找内存泄露我们会使用 Xcode Instrumen，但是Xcode Instrumen 存在上面所述的问题，最为明细的是，需要人工一点点取分析，不能报警。因此，我们一般通过 MLeaksFinder 来查找内存问题。
## 使用
参考 <https://github.com/Tencent/MLeaksFinder>
## 原理
## 源码分析
# Instruments 文件相关
## File Activity
# Instruments 耗时相关
# Instruments 耗时相关
## App lan









# 相关API介绍
## UIImage API
### imageNamed相关
```
+ (nullable UIImage *)imageNamed:(NSString *)name;      // load from main bundle
+ (nullable UIImage *)imageNamed:(NSString *)name inBundle:(nullable NSBundle *)bundle withConfiguration:(nullable UIImageConfiguration *)configuration API_AVAILABLE(ios(13.0),tvos(13.0),watchos(6.0));
#if __has_include(<UIKit/UITraitCollection.h>)
+ (nullable UIImage *)imageNamed:(NSString *)name inBundle:(nullable NSBundle *)bundle compatibleWithTraitCollection:(nullable UITraitCollection *)traitCollection API_AVAILABLE(ios(8.0));  
```  
+ UIImageConfiguration 是对UITraitCollection的封装
	+ The iOS interface environment for your app, defined by traits such as horizontal and vertical size class, display scale, and user interface idiom.
	![](https://pic.downk.cc/item/5e9da07dc2a9a83be5b6a7a2.jpg)
	
### 其他
```。
+ (nullable UIImage *)imageWithContentsOfFile:(NSString *)path;
+ (nullable UIImage *)imageWithData:(NSData *)data;
+ (nullable UIImage *)imageWithData:(NSData *)data scale:(CGFloat)scale API_AVAILABLE(ios(6.0));
+ (UIImage *)imageWithCGImage:(CGImageRef)cgImage;
+ (UIImage *)imageWithCGImage:(CGImageRef)cgImage scale:(CGFloat)scale orientation:(UIImageOrientation)orientation API_AVAILABLE(ios(4.0));
#if __has_include(<CoreImage/CoreImage.h>)
+ (UIImage *)imageWithCIImage:(CIImage *)ciImage API_AVAILABLE(ios(5.0));
+ (UIImage *)imageWithCIImage:(CIImage *)ciImage scale:(CGFloat)scale orientation:(UIImageOrientation)orientation API_AVAILABLE(ios(6.0));
#endif
```  
### 属性
+  CGSize size; 
	+ 单位是points
	+ 举个例子：一个2X图的像素点事100X00——对应的imagesize是50*50
+ CGFloat scale
	+   当图片是从 Assetcatlog读取时，该属性有对应的值
	+   若不是从 Asset Catalog 里面读取的图片，默认值是1
+   上述多种初始化 UIImage的方法，我们可以通过调用带Scale参数的更改image的size

### 2x手机展示3X图和3X手机展示2X图会有什么问题？
+ 首先在图片的尺寸上不会有任何问题
	+ 因为 UiImage 的size是point，布局的单位也是Point，而即使调用sizetofit等相关方法取适应与Image的size，image的size = 像素点/Scale，所以2x手机展示3X图和3X手机展示2X图会有什么问题
+ 图片质量上
	+ 2X手机展示3X图会舍弃像素点，不会影响质量
	+ 3X手机（一个point需要3个像素点），由于2X图像素点不足，会自己补充，导致图片质量下降
+ 什么情况会影响图片大小布局，如何解决呢？
	+  影响布局
		+ 原有的图片像素为X*Y，我们更改为其他，但是仍然放入Asset Catalog 中，且使用时用了sizetofit 回导致 该图比原来有变化
		+ 改回去就好
		
		```   
		UIImage *image = [self tt_imageNamed:name inBundle:bundle compatibleWithTraitCollection:traitCollection];  
		 return [[UIImage alloc] initWithCGImage:image.CGImage scale:0.25 *[UIScreen mainScreen].scale  
		 orientation:image.imageOrientation]  
```   

## UIImageAsset API
只需关系下面几个API，具体用处后面介绍缓存的时候会讲

```  
- (UIImage *)imageWithConfiguration:(UIImageConfiguration *)configuration; // Images returned hold a strong reference to the asset that created them
- (UIImage *)imageWithTraitCollection:(UITraitCollection *)traitCollection; // Images returned hold a strong reference to the asset that created them


- (void)registerImage:(UIImage *)image withConfiguration:(UIImageConfiguration *)configuration;
- (void)unregisterImageWithConfiguration:(UIImageConfiguration *)configuration; // removes only those images added with registerImage:withConfiguration:
- (void)registerImage:(UIImage *)image withTraitCollection:(UITraitCollection *)traitCollection; // Adds a new variation to this image asset that is appropriate for the provided traits. Any traits not exposed by asset catalogs (such as forceTouchCapability) are ignored.
- (void)unregisterImageWithTraitCollection:(UITraitCollection *)traitCollection; // removes only those images added with registerImage:withTraitCollection:
@end
```   
# imageNamed相关方法加载流程

# imageWithContentsOfFile加载流程
# 图片从APP中加载到屏幕展示流程总结
# 缓存原理
