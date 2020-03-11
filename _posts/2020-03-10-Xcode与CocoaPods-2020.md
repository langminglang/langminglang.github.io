---
layout:     post
title:      Xcode 与 CocoaPods (一）
subtitle:   
date:       2020-03-10
author:     LML
header-img: img/post-bg-ios9-web.jpg
catalog: true
tags:
    - iOS
    - Xcode
    - CocoaPods
---

# 前言
本文简单介绍平时项目中用到的 Xcode 与 CocoaPods 的相关概念以及知识。涉及的可能不全面，更加系统的介绍会在后续展开

## Xcode工程配置和基本概念
### Workspace
+ Workspace 可以包含无数Project
+ Workspace 是为了解决平级的不同Project互相依赖的方案
+ 在一般的 App 中 Workspace 的作用是：组织 pods project 和主工程 project
	+ 使用 CocoaPods 之前只有一个 project：主工程project，引入 CocoaPods 管理pods库之后，pod又组建了一个 project

### Target、SCHEME、Configuration 
#### Target
+ Target 是定义一个 Product（可以是App、Framework、Bundle、静态库等任何类型）的单元
+ Target 可以包含不同的Configuration，以影响产物
+ Target 会继承自 所属的 Project 的配置，但是也可以设置自己的配置
+ 一个项目可以拥有多个 target，而每个 target 可以拥有多个且与 project 不同的配置
+ Target 可以添加 Dependency，Xcode 根据 Dependency 决定Build先后顺序 ——属于显式依赖
	+ 隐式Dependency ：假设要得到Target 1 隐式依赖 Target 2，那么至少满足其一（文档描述的）：
		+ Target 1的某一个Build Phase的Input Files，来源于Target 2的某一个Build Phase的Output Files
		+ Target 1的Linked Library项，包含了Target 2的某个Framework产物
		+ Target 1的Build Settings的部分参数（如 -l Foobar -framework Hello），会触发搜索找产物名为libFoobar.a/libFoobar.dylib/Hello.framework的Taregt
		+ Target 1是一个Test Target，指定的宿主App是Target 2的产物 

#### Configuration
+ Configuration影响Target的产物的特性
+  不同Configuration对应不同的Build Settings
+  没有使用 CocoaPods 的 项目，Xcode 并没有生成 xcconfig 文件（当然可以自己生成xcconfig 并配置给xcode），CocoaPods 会生成Xcoconfig 文件，将 pod 相关的一些 BuildSettings 配置给 Xcode
	+ Build Settings ：是用来管理当前Target产物生成的各项配置 
		+ 编译相关参数: Header Search Path
		+ 链接相关参数: Library Search Path(链接的静态/动态库的搜索路径，-L)、OTHER_LDFLAGS（除系统库外，额外链接的库配置，-l）
 
#### scheme
>   A scheme is a collection of settings that specify the targets to build for a project, the build configuration to use, and the executable environment to use when the product is launched. When you open an existing project (or create a new one), Xcode automatically creates a scheme for each target. The default scheme is named after your project.  
> Build is a special action that allows you to select targets to build for each of the other actions.

+ Scheme是针对一个Target的操作（Run/Build/Test/Analyze）设定
+ 上面三者的关系是 Scheme 绑定具体一个 Configuration 以及要编译的Target，还有对应的可选行为（Main Thread Checker，启动参数。Scheme是动作，比如说，NewsInHouseTest 是一个Scheme，里面配置了使用Debug Configuration，并且注入Main Thread Checker，编译的 Target 用 NewsInHouse。
+ 一个 Project 可以包含多个Target，编译链接的时候可以自由选择任意一个 Target，每个 Target 又可以设置（选择）不同的 Configuration，来影响产物。比如说头条那个NewsBackgroundFetch，编译的是 NewsInHouse 的Target，用NewsInHouseDebug Configuration，然后在里面勾选了一个Simulate Background Fetch额外参数。

![Alt Image Text](/Users/langminglang/Desktop/Lark20200310-215300.png "Xcode Sheme 设置页面")

### Xcode Config File
+ Xcode Config File(xcconfig)是一个用来管理Build Settings的文本配置
+ 对同一参数的值，优先级比GUI配置要低
+ CocoPods 采用了 xcconfig
+ 对于 xcconfig的具体分析等下面介绍 CocoaPods 的时候在介绍
+ 一般情况下我们可以根据Build Settings的手册 判断我们可以早 xcconfig 里面写入什么
+ 参考
	+ 语法5分钟上手：<https://pewpewthespells.com/blog/xcconfig_guide.html>
	+ Build Settings的手册-Xcode Help：<https://help.apple.com/xcode/mac/10.2/#/itcaec37c2a6> 
![Alt Image Text](/Users/langminglang/Desktop/Lark20200310-220728.png "自己给Xcode配置xcconfig")

### Build Phase
### Build Settings
