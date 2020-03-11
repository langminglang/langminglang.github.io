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

# Xcode工程配置和基本概念
## Workspace
+ Workspace 可以包含无数Project
+ Workspace 是为了解决平级的不同Project互相依赖的方案
+ 在一般的 App 中 Workspace 的作用是：组织 pods project 和主工程 project
	+ 使用 CocoaPods 之前只有一个 project：主工程project，引入 CocoaPods 管理pods库之后，pod又组建了一个 project

## Target、SCHEME、Configuration 
### Target
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

### Configuration
+ Configuration影响Target的产物的特性
+  不同Configuration对应不同的Build Settings
+  没有使用 CocoaPods 的 项目，Xcode 并没有生成 xcconfig 文件（当然可以自己生成xcconfig 并配置给xcode），CocoaPods 会生成Xcoconfig 文件，将 pod 相关的一些 BuildSettings 配置给 Xcode

 
### scheme
>   A scheme is a collection of settings that specify the targets to build for a project, the build configuration to use, and the executable environment to use when the product is launched. When you open an existing project (or create a new one), Xcode automatically creates a scheme for each target. The default scheme is named after your project.  
> Build is a special action that allows you to select targets to build for each of the other actions.

+ Scheme是针对一个Target的操作（Run/Build/Test/Analyze）设定
+ 上面三者的关系是 Scheme 绑定具体一个 Configuration 以及要编译的Target，还有对应的可选行为（Main Thread Checker，启动参数。Scheme是动作，比如说，NewsInHouseTest 是一个Scheme，里面配置了使用Debug Configuration，并且注入Main Thread Checker，编译的 Target 用 NewsInHouse。
+ 一个 Project 可以包含多个Target，编译链接的时候可以自由选择任意一个 Target，每个 Target 又可以设置（选择）不同的 Configuration，来影响产物。比如说头条那个NewsBackgroundFetch，编译的是 NewsInHouse 的Target，用NewsInHouseDebug Configuration，然后在里面勾选了一个Simulate Background Fetch额外参数。

![Alt Image Text](https://pic.downk.cc/item/5e68c502e83c3a1e3a82d8dc.png)

## Xcode Config File
+ Xcode Config File(xcconfig)是一个用来管理Build Settings的文本配置
+ 对同一参数的值，优先级比GUI配置要低
+ CocoPods 采用了 xcconfig
+ xcconfig 的具体分析等下面介绍 CocoaPods 时在介绍
+ 一般情况下我们可以根据 Build Settings的手册判断我们可以在 xcconfig 里面配置写入什么
+ 自己为 xcode project 配置 xcconfig
![Alt Image Text](https://pic.downk.cc/item/5e68e3cee83c3a1e3a9a58a5.png)
+ 参考
	+ 语法5分钟上手：<https://pewpewthespells.com/blog/xcconfig_guide.html>
	+ Build Settings的手册-Xcode Help：<https://help.apple.com/xcode/mac/10.2/#/itcaec37c2a6> 


## Build Phase
+ Build Phase告诉Xcode怎么构建产物
+ 几个常见标准的Phase：
	+ Compile Source：需要编译的源文件，可以指定每个文件级别的额外编译参数
	+ Linked Library：需要链接的二进制，可以是静态库或者动态库
	+ Copy Bundle Resource：需要拷贝的资源，比如Bundle，JSON，XCAssets（会编译成Assets.car）
	+ Embed Watch Content：如果含有Apple Watch需要拷贝WatchKit App 

## Build Settings

+ Build Settings：是用来管理当前Target产物生成的各项配置，Target会继承自Project级别的配置，可覆盖
	+ 编译相关参数: Header Search Path
	+ 链接相关参数: Library Search Path(链接的静态/动态库的搜索路径，-L)、OTHER_LDFLAGS（除系统库外，额外链接的库配置，-l）

# CocoaPods与包管理
## 背景概念
+ RVM：管理 Ruby 版本的工具
	+ 系统可以安装多个 Ruby 版本，用 RVM 来切换
+ Ruby 与 RubyGems
  - 每个版本的 Ruby 都有自己的一套工具工具集；RubyGems 用来管理这些 Ruby 的工具集，可以安装各种工具
  - 这些工具包含：Bundler、CocoaPods等，也可以包含自己自定义的 gem 工具
+ Bundler
  - 管理当前目录下的工具环境：GemFile Gemfile.lock
  - 例如可以控制一个工程（目录）需要的 CocoPods 版本和 Ruby 工具的版本
+ 举个例子
	+目前头条在工程目录下放置了 GemFile，开发的时候可以先执行一次 bundle insatll，一方面保证所有开发者的环境一致，另一方面方便安装一些 gem 工具集
	
## 基本使用
### Pod update、 pod install、 pod update repo update 、pod update no repo update
- pod install 按照 podfile.lock 安装所有的pod库
	- 比如把工程目录下的 pods 文件删除，由于已经执行过pod update，本地仓库已经有了需要的 pod 库的 podspec 文件（pod 库安装说明书），执行pod install 会按照 podfile.Lock 文件里面的pod库版本号，获取对应pod库的podspec，然后安装pod库
- Pod update 按照 podfile 指定的版本进行更新 pod 库
	- Pod update  会先执行 Pod update  repo update 更新 local spec repo 即更新pod 说明书，同时更改podfile.lock文
	- 然后在执行 pod install
- pod update repo update 更新 pod仓库的 pod spec 文件，其实可以指定更显哪个pod 库的 podsepc文件
	- 这个明朗存在的原因：比如本地的podspec 只到3.2，但是你更新podfile之后发现需要安装4.2版本的pod库了，这个时候必须安装心的podsepc后，才能安装pod 库

### 不同target、Configuration如何依赖不同的库
![Alt Image Text](https://sm.ms/delete/niGrDazqvQ9CEw6tSodO4Zslc2)

⚠️这里需要注意：podfile 里面怎么写直接影响 xcconfig文件的生成

### pod引入主工程的宏
![Alt Image Text](https://pic.downk.cc/item/5e68e419e83c3a1e3a9a7809.png)

### 如何创建自己的 pod 库
网上资料很多，这里不在介绍，可以参考 <https://www.jianshu.com/p/19a0b31b47a3>

## 简单原理
### CocoaPods 作用

+ pod install 的过程就是执行pod file 的过程：会有如下改动
	+ 生成一个 pod  project
		+ 该 project 有多少个target 取决于 podfile里面定义了几个target
		+ podfile 里面引入的库都体现在 pod project 的 Build phase 的 Dependecies，且每个pod库都是 pod project 下面的一个子project，即
			+  每个pod库都是一个project
				+  一个target（pod库的二进制部分）
				+  或者多个target： pod库的二进制与他的资源target。pod库的target 的dependecy 会有他的资源target，这样保证编译这个pod库的时候也编译了pod库的资源文件，但是区分成两个target，又使得二者可以拥有各自的 bulid settings
			+  每个pod都有自己的xccongig ——编译的说明书
	+  生成给主工程编译连接使用的 xcconfig 文件
		+  CocoaPods 通过生成 xcconfig 文件影响主工程编译连接过程
		+  会产生几个xcconfig
			+  podfile 里面引入了几个target会产生几个xcconfig文件夹，供每个target使用，因为每个target
在podfile里面设置可能不一样，所以会设置了几个target产生几个xcconfig文件夹
			+ 为啥说是文件夹呢？因为每个target的xccongig文件夹内有多少个xcconfig文件取决于主工程定义了多少个Configuration。podfile 里面也可以针对不同的Configuration 做设置（具体可见上面用法部分介绍）

``` 
podfile 里面引入多个target举例
target 'BDModelDynamic_Example' do
  pod 'BDModelDynamic', :path => '../'
  target 'BDModelDynamic_Tests' do
    inherit! :search_paths
  end
end 
``` 

![Alt Image Text](https://pic.downk.cc/item/5e68e63ae83c3a1e3a9b5e68.png)

![Alt Image Text](https://pic.downk.cc/item/5e68e64de83c3a1e3a9b69e9.png)

![Alt Image Text](https://pic.downk.cc/item/5e68e665e83c3a1e3a9b79a8.png)
		
+ 改变主工程设置
	+ Configuration设置变化
		+ 引入CocoaPods之后， 主工程的设置其实也会变化， 我们先看一下引入之前，主工程的Configuration设置，如下图所示:       
	![Alt Image Text](https://pic.downk.cc/item/5e68e67ee83c3a1e3a9b8304.png)
	
		+ 引入之后，如下图所示
	
	![Alt Image Text](https://pic.downk.cc/item/5e68e692e83c3a1e3a9b8b0c.png)
	
	+ Bulid phase 的 link Binary with libraries 中 增加了 libPods-XXXX.a （pod project 编译产物）

### xcconfig 文件介绍

主要是对 xcconfig 文件中过的 ```FRAMEWORK_SEARCH_PATHS``` 、``` LIBRARY_SEARCH_PATHS```、```OTHER_LDFLAGS``` 的理解。举个例子

```LIBRARY_SEARCH_PATHS = /Path/To/My/Libraries/ OTHER_LDFLAGS = -l "AFNetworking" -l "SDWebImage" ```

等价于执行

 ```ld -L/Path/To/My/Libraries -l "AFNetworking" -l "SDWebImage" main.o -o MyApp```
 
 Ld 接受参数，用-l 来指定.a/.dylib用-framework 指定.framework用-L指定这些library去哪里搜索，是绝对路径。编译连接的时候，会吧xccongig文件转换成ld命令，但是指的注意的是并不是在ld里面的库都会被link进去，还是遵循静态链接规则。
 	
### 编译并链接第三方库的原理
+ 编译顺序
	+ 主工程的main target显示指出了需要链接库 libPods-XXXXX.a，而libPods-XXXXX.a由target Pods-XXXX（pod project）产生，所以主工程的main target将会隐式依赖于target Pods-XXXX，而在target Pods-XXXXX的配置中，显示指出了依赖对第三方库对应的target的依赖。所以保证了编译顺序：第三方pod库——pod project——主工程
+ 如何编译
	+ 会参考 xcconfig文件的 ```HEADER_SEARCH_PATHS```
+ 如何链接
	+ 会参考 xcconfig文件的 ```FRAMEWORK_SEARCH_PATHS``` 、``` LIBRARY_SEARCH_PATHS```、```OTHER_LDFLAGS```
+ 小结
	+ cocopods 对编译连接的影响，最终其实体现在生产力xccongig文件，而xccongig文件是bulid settings，会影响编译连接的过程 

## CocoPods在资源上的管理
+  Pod 通过 Resources 与 Resource_bundle 引入资源的区别
	+ Resources：Assets里面的资源合并到主工程的Asset里面，其他的资源直接拷贝搭配 App 的主目录下
	+ Resource_bundle：每个pod库产生一个bundle，然后bundle被拷贝到 App 的主目录下
+ 上面说的拷贝 问题
	+ cocopods 会为 Xcode bulid phase 产生一个项目：Copy Bundle Resource：这里会执行 CocoaPods 生成的一个脚本。完成上面说的拷贝过程
	+ 具体可以自己参考自己项目工程里面该项的脚本
+  上面脚本在拷贝 XcAsset 资源的时候存在的 bug	
	+ 脚本片段	
![Alt Image Text](https://pic.downk.cc/item/5e68e6b4e83c3a1e3a9b9773.png)
	+ bug
		+ XCASSET_FILES 是一个数组，其中有几个元素取决于有多少个pod 将xcassets写到了s.resoures中，里面的内容是通过s.resoures引入资源的pod库的xcasset的路径。通过脚本可以看出，使用resource引入xcasset资源的时候会发生：
			+ 把pods中通过resource形式引入xcasset的xcasset的路径放入XCASSET_FILES数组中
			+ 如果XCASSET_FILES数组不为空，则扫描工程目录下所有包含xcasset的路径，加入到数组，以便后续copy
		+ 这样上面的流程就存在问题，会造成
			+ 不是该target的acssets被copy进去
			+ bundel引入的也会被copy，导致资源存在两份
			+ 本身通过resource引入的资源会不会存在两份？不会，因为路径一样会去重
		+ 上面两个错误，其实cocopods意识到了其中一个（第二个），即pod的xcassets会重复引入，所有他们增加了一个语句：跳过pods路径的xcassets，但是这个语句没有起作用
		+ 解决方案：替换掉脚本中的一行： ``` OTHER_XCASSETS=$(find "$PWD" -iname "*.xcassets" -type d)```(这一行会找出工程根目录下所有的xcassets)
替换为：```OTHER_XCASSETS=$(ruby get_all_xcassets ${SRCROOT} ${TARGETNAME})```
			+ get_all_xcassets 是我写的一个ruby脚本，这一行的作用是找出主工程当前target的build phase中的copy bundle resources中的所有xcassets，不在筛选pod库的——因为pod库的没必要在筛选了，应该被copy的已经在XCASSET_FILES里面了
			- Sed 命令实现替换 cocpods 生成的脚本的某一行
	
## CocoaPods 插件
