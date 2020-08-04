---
layout:     post
title:      __bridge、__bridge_retained与__bridge_transfer
subtitle:   
date:       2020-03-19
author:     LML
header-img: img/post-bg-ios9-web.jpg
catalog: true
tags:
    - OC与C++混合编程
---
# 前言
在 iOS 开发当中，难免会使用到 OC 跟 C++混编的情况，本文介绍如何在 ARC 环境下转换 C 指针跟 OC 类指针

# __bridge

__bridge ：用作于普通的 C 指针与 OC 指针的转换，不做任何操作。所谓不做任何操作，指的是不对 OC 只在进行 retian 也不进行 release

```
void *p;
NSObject *objc = [[NSObject alloc] init];
p = (__bridge void*)objc;
```
上述代码执行完毕之后，这里 p 指针直接指向了 NSObject *objc 这个 OC 类，但是 p 指针并不拥有 OC 对象，跟普通的指针指向地址无疑。所以这个出现了一个问题，OC 对象被释放，p 指针便成野指针 。比如下面的代码会crash，因为p执行的内存已经释放了
```
int main(int argc, const char * argv[]) {
    @autoreleasepool {
        void *p;
        {
            ABSClass *objc = [[ABSClass alloc]init];
            objc.name = @"我们";
            p = (__bridge void*)objc;
        }
        NSLog(@"%@", [(__bridge ABSClass *)p name]);
    }
    return 0;
}
```
# __bridge_retained

__bridge_retained：用作 C 指针与 OC 指针的转换，并且也用拥有着被转换对象的所有权，即会使得指向的 OC 对象引用计数+1。__bridge_retained 修饰之后的 C 指针强引用对象，且即使 C指针出了作用范围，指向的内存不会释放。比如下面的代码，最终 ABSClass 引用计数为1，会出现内存泄露

```
int main(int argc, const char * argv[]) {
    @autoreleasepool {
       ABSClass *objc = [[ABSClass alloc]init];
       objc.name = @"我们";
        {
            void *p;
            p = (__bridge_retained void*)objc;
        }
        NSLog(@"%@", [(__bridge ABSClass *)p name]);//这里ABSClass 引用计数为2
    }
    //这里ABSClass 引用计数为1，内存泄露
    return 0;
}
```

# __bridge_transfer

__bridge_transfer：用作 C 指针与 OC 指针的转换，并在拥有对象所有权后将原先对象所有权释放。(只支持 C 指针转换 OC 对象指针)
上面介绍了使用  ```__bridge_retained``` 进行转换的弊端，无法释放对象，那怎么解决呢。与 ``` __bridge_transfer ``` 配合使用就行，把 C 指针转换为OC，转换过程引用计数不变，然后在由 OC 指针去取释放内存


```
int main(int argc, const char * argv[]) {
    @autoreleasepool {
       ABSClass *objc = [[ABSClass alloc]init];
       objc.name = @"我们";
        {
            void *p;
            p = (__bridge_retained void*)objc;
            ABSClass *objcTemp = (__bridge_transfer id)p;
        }
        NSLog(@"%@", [(__bridge ABSClass *)p name]);//这里ABSClass 引用计数为2
    }
    //这里ABSClass 引用计数为1，内存泄露
    return 0;
}
```

```
int main(int argc, const char * argv[]) {
    @autoreleasepool {
        void *p;
        @autoreleasepool {
            ABSClass *obj = [[ABSClass alloc] init];
            obj.name = @"我们";
            p = (__bridge_retained void *)obj;
        }
        id obj = (__bridge_transfer id)p;
        NSLog(@"%@", [(__bridge ABSClass *)p name]);
        NSLog(@"%@", [(ABSClass *)obj name]);
        NSLog(@"Hello, World!");
    }
    return 0;
}
```

