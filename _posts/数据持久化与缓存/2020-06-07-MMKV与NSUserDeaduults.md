---
layout:     post
title:      MMKV与NSUserDefaults
subtitle:   
date:       2020-05-23
author:     LML
header-img: img/post-bg-ios9-web.jpg
catalog: true
tags:
    - cache
---



# 本文文章需要解决的问题

+ MMKV
	+ MMKV 的set get default的原理总结
		+ 其实基本是内存文件都保存了一份，内存是个dic，文件是mmap，文件不足的时候需要重排列，扩大文件，并重新mmap（这里是异步还是同步需要了解，因为肯能会造成性能瓶颈——
		+ 每次第一次使用：mmap+初始化dic，这个比较耗时的，后续的文件重排列是否耗时需要看实现
	+ 如何保证线程安全的
+ NSUserDefaults
	+ 实现原理是什么？MMKV是mmap，NSUserDefaults是普通文件读写吗？
	+ 为啥存在性能瓶颈？
	+ 如何保证安全的
+ MMKV与NSUserDefaults 二者对比
	+ 为啥NSUserDefaults存储大key耗时
	+ 各自优缺点
	+ NSUserDefaults存在什么致命缺陷，导致需要MMKV 需要synchronize 来实现磁盘内存数据同步u性能不好早造成的吗？

https://xiaozhuanlan.com/topic/7851346092
https://bytedance.feishu.cn/docs/doccnNNhPHbydrMIzCfXTg5g2kc#a5fT26

 

```   
- (void)setCurrentImage:(UIImage *)currentImage {
	@synchronized(self) {
		if (_currentImage != currentImage) {
			[_currentImage release];
			_currentImage = [currentImage retain];
			// do something
		}
	}
}

- (UIImage *)currentImage {
	@synchronized(self){
		return _currentImage;
	}
}
```    


