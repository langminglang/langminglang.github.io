---
layout:     post
title:      Xcode 与 CocoaPods (一）
subtitle:   
date:       2020-03-10
author:     LML
header-img: img/post-bg-ios9-web.jpg
catalog: true
tags:
    - Xcode
    - CocoaPods
---

# 前言

本文简单介绍平时项目中用到的 Xcode 与 CocoaPods 相关概念以及知识。

# Xcode工程配置和基本概念

## Workspace

+ Workspace 可以包含无数 Project
+ Workspace 是为了解决平级的不同 Project 互相依赖的方案
+ 在一般的 App 中 Workspace 的作用是：组织 pods project 和主工程 project
	+ 使用 CocoaPods 之前只有一个 project：主工程 project，引入 CocoaPods 管理 pods 库之后，pod 又组建了一个 project

## Target、SCHEME、Configuration 

### Target

+ Target 是定义一个 Product（可以是App、Framework、Bundle、静态库等任何类型）的单元
+ Target 可以设置不同的 Configuration，以影响产物
	+ 设置不同的的 Configuration 实质上是指一个 Target 可以设置多组 Build Settings，选择不同的 Configuration 就是选择不同的 Build Settings
+ Target 会继承自所属的 Project 的配置，但是也可以设置自己的配置
+ 一个项目可以拥有多个 target，而每个 target 可以拥有多个且与 project 不同的配置
+ Target 可以添加 Dependency
	+ 显式依赖：Xcode 根据 Dependency 决定 Build 先后顺序
		+ pod project 就是通过显示依赖的形式管理 podfile 里面的 pod 库的
	+ 隐式依赖: 假设要得到 Target1 隐式依赖 Target2，那么至少满足其一（文档描述的）：
		+ Target1 的某一个 Build Phase 的 Input Files，来源于Target2 的某一个 Build Phase 的 Output Files
		+ Target1 的 Linked Library 项，包含了 Target2 的某个 Framework 产物
			+ 主工程就是这样依赖 pod project 的
		+ Target1 的 Build Settings 的部分参数（如 -l Foobar -framework Hello），会触发搜索找产物名为 libFoobar.a/libFoobar.dylib/Hello.framework 的 Taregt
		+ Target1 是一个 Test Target，指定的宿主 App 是 Target2 的产物 

![](https://pic.downk.cc/item/5fb3c934b18d6271131af8dd.png)
![](https://pic.downk.cc/item/5fb3c988b18d6271131b108c.png)

### Configuration

+ Configuration 影响 Target 的产物的特性
+ 不同 Configuration 对应不同的 Build Settings
+ 没有使用 CocoaPods 的项目，Xcode 并没有生成 xcconfig 文件（当然可以自己生成 xcconfig 并配置给 xcode），CocoaPods 会生成 Xcconfig 文件，将 pod 相关的一些 BuildSettings 配置给 Xcode
	+ CocoPods 为主工程生成 Xcconfig 目的是：主要是为主工程提供 HEADER_SEARCH_PATHS 等 ，比如主工程中如果引用了pod库的头文件，那编译的时候去哪里找头文件呢，依赖于 xcconfig 文件 中的 HEADER_SEARCH_PATHS 字段
+  Configuration 的设置中有个 Based on Configuration File，一般情况下是none，除非：
	+ 使用 cocopods ，且 podfile里面有针对该 target 的描述，cocopods 会为其 生成 xcconfig文件且 配置到 Based on Configuration File 中
	+ 我们自己收到配置

 
### scheme
>   A scheme is a collection of settings that specify the targets to build for a project, the build configuration to use, and the executable environment to use when the product is launched. When you open an existing project (or create a new one), Xcode automatically creates a scheme for each target. The default scheme is named after your project.  
> Build is a special action that allows you to select targets to build for each of the other actions.

+ Scheme 是针对一个 Target 的操作（Run/Build/Test/Analyze）设定
+ 上面三者的关系是 Scheme 绑定具体一个 Configuration 以及要编译的 Target，还有对应的可选行为（Main Thread Checker，启动参数）。Scheme 是动作，比如说，NewsInHouseTest 是一个 Scheme，里面配置了使用 Debug Configuration，并且注入 Main Thread Checker，编译的 Target 用 NewsInHouse。
+ 一个 Project 可以包含多个 Target，编译链接的时候可以自由选择任意一个 Target，每个 Target 又可以设置（选择）不同的 Configuration，来影响产物。比如说头条那个 NewsBackgroundFetch，编译的是 NewsInHouse 的 Target，用 NewsInHouseDebug Configuration，然后在里面勾选了一个 Simulate Background Fetch 额外参数。

<img src="https://pic.downk.cc/item/5e68c502e83c3a1e3a82d8dc.png" width = "400" height = "260" alt="Xcode Sheme 设置页面" align=center>

## Xcode Config File

+ Xcode Config File (xcconfig) 是一个用来管理 Build Settings 的文本配置
+ 一个 project 的 xcconfig 文件作用是为编译连接过程提供指导和参数（pod 库的 xcconfig、pods project 的 xcconfig、主工程的 xcconfig 等的作用均是如此）
+ 对同一参数的值，优先级比 GUI 配置要低
+ CocoPods 采用了 xcconfig
+ xcconfig 的具体分析下面介绍 CocoaPods 时在介绍
+ 一般情况下我们可以根据 Build Settings 的手册判断我们可以在 xcconfig 里面配置写入什么
+ 自己为 xcode project 配置 xcconfig
+ 参考
	+ 语法5分钟上手：<https://pewpewthespells.com/blog/xcconfig_guide.html>
	+ Build Settings的手册-Xcode Help：<https://help.apple.com/xcode/mac/10.2/#/itcaec37c2a6> 

<img src="https://pic.downk.cc/item/5e68e3cee83c3a1e3a9a58a5.png" width = "400" height = "100" alt="Xcode 配置xcconfig" align=center>



## Build Phase
+ Build Phase 告诉 Xcode 怎么构建产物
+ 几个常见标准的 Phase：
	+ Compile Source：需要编译的源文件，可以指定每个文件级别的额外编译参数
	+ Linked Library：需要链接的二进制，可以是静态库或者动态库
	+ Copy Bundle Resource：需要拷贝的资源，比如Bundle，JSON，XCAssets（会编译成Assets.car）
	+ Embed Watch Content：如果含有Apple Watch需要拷贝WatchKit App 

## Build Settings

+ Build Settings：是用来管理当前Target产物生成的各项配置，Target会继承自Project级别的配置，可覆盖
	+ 编译相关参数: Header Search Path
	+ 链接相关参数: Library Search Path(链接的静态/动态库的搜索路径，-L)、OTHER_LDFLAGS（除系统库外，额外链接的库配置，-l）

其实上文说的 Xcconfig 文件的配置，影响的也是 build settings。即 xcconfig 文件影响 Bulid Settings 配置，然后 Bulid Settings，影响编译链接。

![](https://pic.downk.cc/item/5fb4be72b18d627113525927.png)

注意上面图片，其中 build Settings 中的 Resolved 的一列是最终的 build settings，首 target GUI、 target config 、project GUI、project config 等的综合影响，几个的优先级见下面图。

Resolved 一列 和 Build Settings 的 combined 展示的是相同的。 

![](https://pic.downk.cc/item/5fb4be4cb18d62711352523a.png)



# CocoaPods与包管理
	
## 基本使用

### Pod update、 pod install、 pod update repo update 、pod update no repo update

- pod install 按照 podfile.lock 安装所有的pod库
	- 比如把工程目录下的 pods 文件删除，由于已经执行过 pod update，本地仓库已经有了需要的 pod 库的 podspec 文件（pod 库安装说明书），执行 pod install 会按照 podfile.Lock 文件里面的pod库版本号，获取对应 pod 库的podspec，然后安装 pod 库
	- 其实命令等于 pod install -no-repo-update
- pod install -repo-update
	- 按照 podfile.lock 文件中的版本号更新podsepc，然后 pod install
	- pod update 是安装最新的版本
	- 比如 podfile里面写 >= 3.5,podfilelock 里面是 3.5，如果是 pod install -repo-update，那么则是更新到3.5；如果是pod update 则是更新到 最新版本，并更改 podfile.lock
- Pod update 按照 podfile 指定的版本进行更新 podsepc 并 install pod 库
	- Pod update  会先执行 Pod update  repo update 更新 local spec repo 即更新pod 说明书，同时更改 podfile.lock
	- 然后在执行 pod install
- pod update repo update 更新 pod仓库的 pod spec 文件，其实可以指定更显哪个 pod 库的 podsepc文件

### 不同 target、Configuration 如何依赖不同的库
<img src="https://pic.downk.cc/item/5e68e759e83c3a1e3a9be388.png" width = "400" height = "200"  align=center>

⚠️这里需要注意：podfile 里面怎么写直接影响 xcconfig 文件的生成

### pod引入主工程的宏
<img src="https://pic.downk.cc/item/5e68e419e83c3a1e3a9a7809.png" width = "400" height = "200" align=center>

### 如何创建自己的 pod 库
网上资料很多，这里不在介绍，可以参考 <https://www.jianshu.com/p/19a0b31b47a3>

## 简单原理

### CocoaPods 作用

+ pod install 的过程就是执行 pod file 的过程：会有如下改动
	+ 生成一个 pod project
		+ 该 project 有多少个 target 取决于 podfile 里面定义了几个 target
		+ podfile 里面引入的库都体现在 pod project 的 Build phase 的 Dependecies，且每个 pod 库都是 pod project 下面的一个子 project，
			+  每个 pod 库都是一个 project，该 project 会拥有一个或多个 target
				+ 一个 target：podflie 里面的定义的多个 target 依赖完全相同的该库，且该库没有资源
				+ 多 target，分为几种情况：
					+ 一个二进制 target + 一个资源 target：podflie 里面的定义的多个 target 依赖完全相同的该库，且该库有一个 resource_bundle 资源
					+ 多个二进制target + 多个资源target：podflie 里面的定义的多个 target 依赖该库不同的subspec，且每个subspec都有 resource_bundle  资源
				+  pod 库的二进制 target 的 dependecy 会有他的资源 target，这样保证编译这个 pod 库的时候也编译了pod 库的资源文件，但是区分成两个target，又使得二者可以拥有各自的 bulid settings
			+  每个pod都有自己的 xccongig ——编译的说明书
	+  生成给主工程编译链接使用的 xcconfig 文件
		+  CocoaPods 通过生成 xcconfig 文件影响主工程编译连接过程
		+  会产生几个xcconfig
			+  podfile 里面引入了几个target会产生几个xcconfig文件夹，供每个target使用，因为每个 target 在 podfile 里面设置可能不一样，所以会设置了几个 target产生几个 xcconfig文件夹
			+  为啥说是文件夹呢？
				+ 因为每个 target 的 xccongig 文件夹内有多少个 xcconfig 文件取决于主工程定义了多少个 Configuration。podfile 里面也可以针对不同的 Configuration 做设置（具体可见上面用法部分介绍）
+ 改变主工程设置
	+ Configuration设置变化
		+ 引入CocoaPods之后， 主工程的设置其实也会变化， 我们先看一下引入之前，主工程的 Configuration 设置，如下图所示:
		
			<img src="https://pic.downk.cc/item/5e68e67ee83c3a1e3a9b8304.png" width = "500" height = "250" align=center>

		+ 引入之后，如下图所示，可以发现 cocopods 生成的 xcconfig 文件已经自动配置到了主工程的 Configuration设置中，用于在编译链接时候使用

			<img src="https://pic.downk.cc/item/5e68e692e83c3a1e3a9b8b0c.png" width = "500" height = "250" align=center>
	
	+ Bulid phase 的 link Binary with libraries 中 增加了 libPods-XXXX.a （pod project 编译产物），用于控制编译的顺序


<img src="https://pic.downk.cc/item/5e68e63ae83c3a1e3a9b5e68.png" width = "300" height = "300" title="主工程Configuration设置" align=center>

<img src="https://pic.downk.cc/item/5e68e64de83c3a1e3a9b69e9.png" width = "300" height = "200" title="生成的xcconfig文件夹" align=center>

<img src="https://pic.downk.cc/item/5e68e665e83c3a1e3a9b79a8.png" width = "300" height = "200" title="xcconfig文件夹内的xcconfih文件" align=center>


### cocopods 管理 pod 库的资源和二进制的原理


1. 举例子说明 cocopods 是如何管理 pod 库的二进制和资源的。
 
+ podfile 里面多 target 依赖完全相同的 pod 库，且该 pod 库没有资源
	+ 只有一个二进制 target

 ![](https://pic.downk.cc/item/5fae4c621cd1bbb86b98283e.png)

+ podfile 里面多 target 依赖完全相同的 pod 库，该 pod 库有资源
	+ 拥有资源 target 和二进制 target 各一个
		+ 资源 target 的名字是 pod target name - podspec 里面 resource_bundle 定义的名字， 且 pod 库只有一个 respurce_bundle 引入资源，所以只有一个资源 target

![](https://pic.downk.cc/item/5fae4ca31cd1bbb86b983e76.png)

+ Pod 库拥有多个 subspec，每个拥有自己的资源，且不同 target 引入的 subspec 相同
	+ 拥有一个二进制 target 和多个资源 target
		+ 由于主工程依赖的pod库完全相同，所以只有一个二进制target
		+ 由于pod 库是多个subspec组成，且每个 subspec 均有 resource_bundle，所以有多个资源 target，二进制依赖所有的资源 target

![](https://pic.downk.cc/item/5fae56b51cd1bbb86b9afa54.png)

+ 规律：
 + 对于同一个 pod 库，pod project 不同 target 依赖 pod 库的不同 subspec，依赖多少份，则：
 	+ 产生几份二进制 taget
 	+ 产生资源 target 的分数 = target1引入的subspec 中拥有资源的subpsec 个数 + target2引入的subspec 中拥有资源的subpsec 个数 + 。。。。


2. cocopods 如何保证依赖的pod 库（资源+二进制）参与到主工程的编译连接种去的？

+ 编译
	+ pod project 库会依赖引入的 pods二进制target
	+ 而 pods 二进制 target 依赖其 资源target
	+ 主工程的 Link Binary with Libraries，中依赖了 pod project
	+ 经过上述步骤后，二进制编译产物静态库文件和资源编译产物 bundle 均形成了
	+ 编译顺序
		+ 主工程的main target显示指出了需要链接库 libPods-XXXXX.a，而libPods-XXXXX.a由target Pods-XXXX（pod project）产生，所以主工程的main target将会隐式依赖于target Pods-XXXX，而在target Pods-XXXXX的配置中，显示指出了依赖对第三方库对应的target的依赖。所以保证了编译顺序：第三方pod库——pod project——主工程
+ 连接
	+ 对于静态库，根据主工程的 xcconfig 文件的 ``` LIBRARY_SEARCH_PATHS```、```OTHER_LDFLAGS``` 确定需要连接的静态库，静态靠的路径，静态库单的路径就是编译产物的路径
	+ 对于bundle 文件，其实在执行 pod install 的时候就生成了 ```Pods-XXX-resources.sh" ``` 文件，并放置在了主工程的 build phases 的 copy bundle resource 步骤执行，实现把编译生成的资源产物拷贝到的 APP 中
		+ 之所以提前写好是因为，产物的路径是固定的





3. coocpods 如何实现既可以依赖 pod 二进制 也可以依赖 pod 库的源码

上面说过一个 pod 库的资源和二进制是两个 target，且存在以来关系，所以 pod 的二进制部分是源码还是二进制依赖不影响该 pod 库的资源target被编译。

那源码与二进制引入一个 pod 库，其二进制部分在编译连接的时候有什么区别呢？

编译资源：
![](https://pic.downk.cc/item/5fae64bd1cd1bbb86b9eb56b.png)

源码编译二进制
![](https://pic.downk.cc/item/5fae64f41cd1bbb86b9ec279.png)
![](https://pic.downk.cc/item/5fae65111cd1bbb86b9ecbcf.png)

二进制编译二进制
![](https://pic.downk.cc/item/5fae653a1cd1bbb86b9ed61a.png)


可以看出，当一个库被 pod install 的时候会根据是源码引入还是二进制引入，区分配置它的 bulid phases 中的 compile Sources；同时区分生成 xcconfig 文件中关于该库的 ``` LIBRARY_SEARCH_PATHS```

![](https://pic.downk.cc/item/5fae65fa1cd1bbb86b9f0d05.png)


		

### xcconfig 文件介绍

主要是对 xcconfig 文件中的 ```FRAMEWORK_SEARCH_PATHS``` 、``` LIBRARY_SEARCH_PATHS```、```OTHER_LDFLAGS``` 的理解。举个例子

```LIBRARY_SEARCH_PATHS = /Path/To/My/Libraries/ OTHER_LDFLAGS = -l "AFNetworking" -l "SDWebImage" ```

等价于执行

 ```ld -L/Path/To/My/Libraries -l "AFNetworking" -l "SDWebImage" main.o -o MyApp```
 
 Ld 接受参数，用 -l 来指定需要连接的静态库 .a/.dylib 用 -framework 指定需要连接的动态库 .framework 用 -L 指定这些 library 去哪里搜索，是绝对路径。编译连接的时候，会把 xccongig 文件转换成 ld 命令，但是指的注意的是并不是在 ld 里面的库都会被 link 进去，还是遵循静态链接规则。

 也就是 ```FRAMEWORK_SEARCH_PATHS``` 、``` LIBRARY_SEARCH_PATHS``` 指明连接的静态库+动态库的路径；```OTHER_LDFLAGS``` 用于指明哪些库需要连接
 	
+ 如何编译
	+ 会参考 xcconfig文件的 ```HEADER_SEARCH_PATHS```
+ 如何链接
	+ 会参考 xcconfig文件的 ```FRAMEWORK_SEARCH_PATHS``` 、``` LIBRARY_SEARCH_PATHS```、```OTHER_LDFLAGS```
+ 小结
	+ cocopods 对编译连接的影响，最终其实体现在生产了 xccongig文件，而xccongig文件是bulid settings，会影响编译连接的过程 

## CocoPods 在资源上的管理

+  Pod 通过 Resources 与 Resource_bundle 引入资源的区别
	+ Resources：Assets里面的资源合并到主工程的Asset里面，其他的资源直接拷贝搭配 App 的主目录下
	+ Resource_bundle：每个pod库产生一个bundle，然后bundle被拷贝到 App 的主目录下
+ 上面说的拷贝问题
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

	
# RubyGems 工具集

## 背景概念

+ RVM：管理 Ruby 版本的工具
	+ RVM 是一个管理多个 Ruby 环境的工具，可以提供一个便捷的多版本 Ruby 环境的管理和切换。通过 RVM 我们可以轻松的切换 Ruby 的版本，从而使用不同的 Ruby 环境
+ Ruby 与 RubyGems
	+ RubyGems 简称 Gem，是一个用于对 Ruby 组件进行管理及打包的工具。通过 Gem 我们可以管理多个 Ruby 工具集 。我们日常用到的很多工具都是一个 Gem 工具，例如：Bundler、CocoaPods等，也可以包含自己自定义的 gem 工具。
	+ 每个版本的 Ruby 都有自己的一套工具工具集；RubyGems 用来管理这些 Ruby 的工具集
+ Bundler
	+ Bundler 是一个 Ruby 工具，是一个 Gem。Bundler 是用来管理维护项目中依赖的 Gem 的工具。如果我们用一个 iOS 工程来类比一个 Ruby 工程，Bundler 就相当于工程中的 CocoaPods。
	+ Bundler 这个工具中，也有对应的来描述这个工程的依赖关系的文件 -- Gemfile 。其中使用 Ruby 的语法来描述工程所需依赖、依赖的工具的版本等信息。
		+ 一个工程想使用bunlder，只需要：gem install bundler 安装bundler，然后想用的工程 里面建立一个gemfile，执行bundler install 。之后 执行任何人命令之前加上 bundle exec
		+ bundle exec 会在当前项目依赖的上下文环境中执行命令：相同的 gem、相同的依赖关系、相同的版本号
	+ CocoaPods 是一个 Ruby 的工具，所以可以使用 Bundler 来管理。通过 Bundler 中的 Gemfile 的描述文件，我们可以来管理项目中使用的 CocoaPods 及 CocoaPods 版本，以及需要安装的插件（特别是为工程而订制的插件）。
+ 举个例子
	+ 目前头条在工程目录下放置了 GemFile，开发的时候可以先执行一次 bundle update，一方面保证所有开发者的环境一致，另一方面方便安装一些 gem 工具集
	+ 我们可以为自己的工程自定义gem工具，然后用bundler 来控制版本和安装，下面会进行简单的介绍

### 自定义gem工具

#### CocoaPods Plugins
上面说到，我们可以为自己的工程定义gem工具，现在有一个为 Xcode 工具定义的 gem 工具（开发框架）：CocoaPods Plugins。我们可以在CocoaPods Plugins 基础上更便捷的开发自定义的 gem 工具，完成自定义Pod 命令，对pod 相关命令进行hook。可以参考 <https://www.jianshu.com/p/5889b25a85dd>，文档详细的说明了如何建立了一个 CocoaPods Plugin。

##### 流程
1. 开发
	+ hook
	+ add 新函数

2. 加载自定义的插件进行调试：
	+ 假设你开发的插件想作用于 A Xcode project，在A project 的 Gemfile 里面添加自定义的 插件，可以使用path来指定路径。
	+ gem 'cocopods-test', :path=>'../CocoaPods-test'
	+ 执行bundle install，后续正常秩序bundle exec pos install 等

3. 发布
	+ 发布行为与使用 CocoaPods 管理 pod 库类似，我们开发提个pod库，会把pod 库上传到公共源。而这里也要将新版本发布，并上传到对应的Rubygem源，Gemfile 里面可以设定 gem 源。具体步骤：
		+ 修改当前版本号version.rb
		+ 在CocoaPods plugins 工程的根路径上，执行rake build 
		+ build完成会看到一个pkg文件夹，里面有对应的含有版本号，并且扩展名为.gem的文件，发布到对应的gems仓库（gem源）中
		+ 如果是开源的插件，在项目路径使用 pod plugins publish，会创建PR到插件列表 

##### 如何开发

开发分为两种：
+ hook 已有的 CocoaPods 方法
	+ alias_method来进行hook方法（类似于OC的swizzle
	+ hook 的方法写在自己定义的Ruby文件中，文件内部结构（Class 和module）保持和CocoaPods原始代码文件一样的结构，hook方法名注意添加独一无二前缀，注意调用原始方法，以防止冲突（规则同OC的swizzle） 
	+ 定义的ruby 文件需要被require（CocoaPods 的入口处被require 就可以了，入口指的是plugin.ruby文件和与插件名字同名的ruby文件

<img src="https://pic.downk.cc/item/5e6b355be83c3a1e3a816d6a.jpg" width = "600" height = "300" align=center>
	
+ 增加新的命令 
	+ 定义 command 的子类，写自己的代码逻辑
	+ pod 插件名称 +你定义的command 名字即可 
+ 后续有时间在写demo，传到github上去，才能简单明了的写清楚如何自定义一个 CocoaPods plugin

#### 插件开发
+ 也可以开发其他插件，而不是基于cocopods Plugins 的
+ CocoaPods 命令其实是一个gem ，想实现类似功能（增加新的 CocoaPods command、hook已有的command ）你也可以不利用 CocoaPods plugin，完全自己开发。
	
# 参考
+ CocoaPods 源码解析 <http://www.saitjr.com/ios/cocoapods-start-with-pod-command.html>
+ CocoaPods原理：<http://www.cloudchou.com/ios/post-990.html>
+ Xcode Build Settings：<https://help.apple.com/xcode/mac/10.2/#/itcaec37c2a6> 或者 Xcode自带的Help
+ xcconfig教程: <https://pewpewthespells.com/blog/xcconfig_guide.html>
+ CocoaPods 源码,可以hook哪些方法可以参考：<https://www.rubydoc.info/github/cocoapods/cocoapods/master/Pod%2FInstaller:run_podfile_post_install_hook>
+ 语法5分钟上手Xcconfig：<https://pewpewthespells.com/blog/xcconfig_guide.html>
