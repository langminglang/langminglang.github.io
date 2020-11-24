---
layout:     post
title:      KVC + KVO + NSNotificationCenter + 关联对象原理
subtitle:   
date:       2020-03-10
author:     LML
header-img: img/post-bg-ios9-web.jpg
catalog: true
tags:
    - OC 基础
---

# 前言

本文主要介绍 KVC、KVO、NSNotificationCenter、关联对象、weak 四个的原理。了解原理有助于我们更好的使用

# 关联对象原理

## 使用

```
- (NSString *)categoryProperty {
    return objc_getAssociatedObject(self, _cmd);
}

- (void)setCategoryProperty:(NSString *)categoryProperty {
    objc_setAssociatedObject(self, @selector(categoryProperty), categoryProperty, OBJC_ASSOCIATION_RETAIN_NONATOMIC);
}

```  
陌生的同学理解上面使用的一些参数，可能需要下面的知识：

```   
void objc_setAssociatedObject(id _Nonnull object, const void * _Nonnull key, id _Nullable value, objc_AssociationPolicy policy)  

id _Nullable objc_getAssociatedObject(id _Nonnull object, const void * _Nonnull key) OBJC_AVAILABLE(10.6, 3.1, 9.0, 1.0, 2.0);  

typedef struct objc_method *Method;

struct objc_method {
	SEL _Nonnull method_name                                 OBJC2_UNAVAILABLE;
	char * _Nullable method_types                            OBJC2_UNAVAILABLE;
	IMP _Nonnull method_imp                                  OBJC2_UNAVAILABLE;
}

相关方法:  
method_getImplementation——获取IMP  
method_getTypeEncoding——获取method_types  
method_getName——获取SEL  
SEL：方法的名字，字符串，可以通过@selecto获取  
_cmd: 在 Objective-C 的方法中表示当前方法的 selector  

```  

## 数据结构以及原理

注意三个方法：  

```   
void objc_setAssociatedObject(id object, const void *key, id value, objc_AssociationPolicy policy);  
id objc_getAssociatedObject(id object, const void *key);  
void objc_removeAssociatedObjects(id object);

```    

堆栈和相关数据结构：  

```  
void objc_setAssociatedObject(id object, const void *key, id value, objc_AssociationPolicy policy)
└── void objc_setAssociatedObject_non_gc(id object, const void *key, id value, objc_AssociationPolicy policy)
    └── void _object_set_associative_reference(id object, void *key, id value, uintptr_t policy)
```  
![](https://pic.downk.cc/item/5fbc83b6b18d6271132513b8.jpg)

可以发现全局单例 AssociationsManager 维护来一个 map，map 的 key 是指针转换成的数字，这个指针即为需要设置关联对象的实例的地址，能保证不使其计数加1还能保证唯一性，value 是一个 map，因一个实例可能关联多个对象。里层的 map 的 key 是属性名字，value 是 objcAssocation，下面有具体介绍  

Is it correct to use objc_setAssociatedObject for class object?(https://stackoverflow.com/questions/15609149/is-it-correct-to-use-objc-setassociatedobject-for-class-object)  

是的，我们可以给class 添加关联对象，只不过需要注意:  
- class 关联对象
  - 必须通过 class 不能是实例获取
  - 子类不能获取，只能是 set 的 class 本身获取。
- 实例的关联对象
  - 只能通过实例获取，不能通过 class 获取
  - 且子类可以获取
    - 比如父类init里面add ，子类实例可以获取  

其实很容易理解这是为什么，关联对象依托与全局对象存在（map）。key 是 DISGUISE（地址）得到 key，class 本身也有地址，且不同 class以及 class与其 superclass 地址不同，则 key 不同，所以取不到 class 或 实例的关联对象。  

## 关键点 

网上的大多数文章只介绍到这里：通过介绍关联对象的数据结构讲解原理，但是大家有没有思考过：  
- 关联对象支持设置assign、retain、copy 这个是如何实现的呢？  
- 一个设置过关联对象的实例 dealloc 的时候是如何高效的保证关联对象的释放呢？  
下面简单解答下这两个问题的原理

### 关联对象设置assign、retain、copy的原理

下面代码是 ObjcAssociation 的结构，其中有个方法是 acquireValue，该方法会根据设置的 valuue 是哪种类型进行不同的操作  
- Assign
  - 不做操作，相当于一个C++指针，这个容易导致一个问题，当关联的对象已经释放，在获取关联对象会造成 crash
- Cony
  - 调用关联对象copy方法
- Retain
  - 调用 retain 方法，是的计数加1  

```  
class ObjcAssociation {
    uintptr_t _policy;
    id _value;
public:
    ObjcAssociation(uintptr_t policy, id value) : _policy(policy), _value(value) {}
    ObjcAssociation() : _policy(0), _value(nil) {}
    ObjcAssociation(const ObjcAssociation &other) = default;
    ObjcAssociation &operator=(const ObjcAssociation &other) = default;
    ObjcAssociation(ObjcAssociation &&other) : ObjcAssociation() {
        swap(other);
    }

    inline void swap(ObjcAssociation &other) {
        std::swap(_policy, other._policy);
        std::swap(_value, other._value);
    }

    inline uintptr_t policy() const { return _policy; }
    inline id value() const { return _value; }

    inline void acquireValue() {
        if (_value) {
            switch (_policy & 0xFF) {
            case OBJC_ASSOCIATION_SETTER_RETAIN:
                _value = objc_retain(_value);
                break;
            case OBJC_ASSOCIATION_SETTER_COPY:
                _value = ((id(*)(id, SEL))objc_msgSend)(_value, @selector(copy));
                break;
            }
        }
    }
    inline void releaseHeldValue() {
        if (_value && (_policy & OBJC_ASSOCIATION_SETTER_RETAIN)) {
            objc_release(_value);
        }
    }

    inline void retainReturnedValue() {
        if (_value && (_policy & OBJC_ASSOCIATION_GETTER_RETAIN)) {
            objc_retain(_value);
        }
    }

    inline id autoreleaseReturnedValue() {
        if (slowpath(_value && (_policy & OBJC_ASSOCIATION_GETTER_AUTORELEASE))) {
            return objc_autorelease(_value);
        }
        return _value;
    }
};

```  
acquireValue 方法在 ```_object_set_associative_reference``` 最开始的时候调用：  

```  
void objc_setAssociatedObject(id object, const void *key, id value, objc_AssociationPolicy policy)
└── void objc_setAssociatedObject_non_gc(id object, const void *key, id value, objc_AssociationPolicy policy)
    └── void _object_set_associative_reference(id object, void *key, id value, uintptr_t policy)
        └──ObjcAssociation association{policy, value};
        	└──association.acquireValue();
```  

那释放的时候如何解决的呢？  
在 ```_object_remove_assocations``` 和 ```_object_set_associative_reference```  设置 nil 时调用了 releaseHeldValue 方法


### 实例 dealloc 时如何保证关联对象的释放   

实例对象的 isa 指针，其中某一位标示了该实例对象是否存在关联对象，如果存在关联对象，则调用_object_remove_assocations 方法

那么这个标志位是什么时候设置的呢？  

当第一次调用 ```_object_set_associative_reference``` 时会调用 setHasAssociatedObjects 方法设置这个标志位


# NSNotificationCenter 原理

## 使用
如何使用不再赘述，可以直接参考头文件

## 原理和数据结构  

- 通知

```  
@interface NSNotification : NSObject <NSCopying, NSCoding>
@property (readonly, copy) NSNotificationName name;
@property (nullable, readonly, retain) id object;
@property (nullable, readonly, copy) NSDictionary *userInfo;
- (instancetype)initWithName:(NSNotificationName)name object:(nullable id)object userInfo:(nullable NSDictionary *)userInfo API_AVAILABLE(macos(10.6), ios(4.0), watchos(2.0), tvos(9.0)) NS_DESIGNATED_INITIALIZER;
- (nullable instancetype)initWithCoder:(NSCoder *)coder NS_DESIGNATED_INITIALIZER;
@end  
```  

值得注意的是，通知会强持有 object，为的是注册观察通知的对象，如果限定来只接受指定对象的通知，如果没有强持有，那么如果 object 释放了会影响观察者接受通知  

- 通知中心  

通知中心是个全局单例，维护了一个字典属性，类似于如下结构  

```  
@property (class, strong) YBNotificationCenter *defaultCenter;
@property (strong) NSMutableDictionary *observersDic;
```  

字典的 key 为通知的名字 NSNotificationName；value 是一个数组，因为一个通知可能对象多个观察者，数组的成员是 model，需要存储一些信息，类似于如下：  

```  
@interface YBObserverInfoModel : NSObject
@property (assign) id observer;
@property (strong) id observer_strong; //addObserverForName注册blaock没有obsever的才需要
@property (strong) NSString *observerId;
@property (assign) SEL selector;
@property (weak) id object;
@property (copy) NSString *name;
@property (strong) NSOperationQueue *queue;
@property (copy) void(^block)(YBNotification *noti)
@end
```  
- 注意：
	+ 一般（指除去 addObserverForName 注册 blaock 没有 obsever 的情况）。不会强持有 observe，系统的实现是 assign，其实用 weak 更合适，系统用 assign，导致了某种情况不移除会造成 crash  
	+ 不会强持有object  
	+ 通过 addObserverForName 注册通知时，YBNotificationCenter 的 YBObserverInfoModel 会强持有注册的 block，不使用的时候我们需要调用 [返回值 removeObserver]，否则 block 一直不释放，Block 捕获的变量也不释放。   
	+ 发送通知
	  - 根据 name 确定需要通知的数组，然后在根据 object限制确定数组里面需要通知的对象，根据 queue 决定 block 或者 selectot 执行的线程，queue 为空在发送通知的线程执行

## 关键点

### 强持有问题

+ 以这个方法为例子，通知中心并没有强持有observe和object
	```。
	- (void)addObserver:(id)observer selector:(SEL)aSelector name:(nullable NSNotificationName)aName object:(nullable id)anObject;
	```  

+ block 被 NSNotificationCenter 强持有
	+ 只有调用方通过 removeObserver 方法移除该方法返回的 id 类型的对象，才会移除 NSNotificationCenter 对该 block 的持有，否则 则 block 一直不释放，block 持有的对象也不会释放

	```  
	- (id <NSObject>)addObserverForName:(nullable NSNotificationName)name object:(nullable id)obj queue:(nullable NSOperationQueue *)queue usingBlock:(void (^)(NSNotification *note))block API_AVAILABLE(macos(10.6), ios(4.0), watchos(2.0), tvos(9.0));
	```  


### 执行的线程 

设置了 queue 在 queue，否则在发送通知的线程执行

### 移除通知问题 

某个版本（iOS9以前）必须在被添加来通知的对象 dealloc 之前移除通知。因为通知中心关联的是类似于 C++ 的指针，不会自动置 nil，会导致 dealloc 后再次发送通知 crash。  

高版本的 iOS 是如何做到的不需要手动调用移除通知呢？因为系统在 dealloc 的时候帮我们做了，如果是我们自己实现自动调用，该如何做？  
 
实现自动移除通知，思路是在响应者 observer 走 dealloc 的时候移除对应的通知，难点就是在 ARC 中是不允许对 dealloc 做 swizzle 等操作的，所以我使用了一个缓兵之计——动态给 observer 添加一个属性，我们监听这个属性的 dealloc 方法移除对应的通知，代码如下：  

NSNotificationCenter 报漏给我们的 api 实际上会调用类似于如下一样的方法，我们可以对于么每个添加通知的实例添加自定义类类型的关联对象（retain类型的），这个实例 dealloc 的时候，关联对象也会 dealloc，然后重写这个自定义类的 dealloc，保证移除所属关联实例的所有通知

```  
- (void)addObserverInfo:(YBObserverInfoModel *)observerInfo {
    
    //为observer关联一个释放监听器
    id resultObserver = observerInfo.observer?observerInfo.observer:observerInfo.observer_strong;
    if (!resultObserver) {
        return;
    }
    YBObserverMonitor *monitor = [YBObserverMonitor new];
    monitor.observerId = observerInfo.observerId;
    const char *keyOfmonitor = [[NSString stringWithFormat:@"%@", monitor] UTF8String];
    objc_setAssociatedObject(resultObserver, keyOfmonitor, monitor, OBJC_ASSOCIATION_RETAIN_NONATOMIC);
    
    //添加进observersDic
    NSMutableDictionary *observersDic = YBNotificationCenter.defaultCenter.observersDic;
    @synchronized(observersDic) {
        NSString *key = (observerInfo.name && [observerInfo.name isKindOfClass:NSString.class]) ? observerInfo.name : key_observersDic_noContent;
        if ([observersDic objectForKey:key]) {
            NSMutableArray *tempArr = [observersDic objectForKey:key];
            [tempArr addObject:observerInfo];
        } else {
            NSMutableArray *tempArr = [NSMutableArray array];
            [tempArr addObject:observerInfo];
            [observersDic setObject:tempArr forKey:key];
        }
    }
}

//监听响应者释放类
@interface YBObserverMonitor : NSObject
@property (strong) NSString *observerId;
@end
@implementation YBObserverMonitor
- (void)dealloc {
    NSLog(@"%@ dealloc", self);
    [YBNotificationCenter.defaultCenter removeObserverId:self.observerId];
}
@end

```  

# KVO 原理 

## 原理

- 当某个类的对象第一次被观察时，系统就会在运行期动态地创建该类的一个派生类（类名就是在该类的前面加上NSKVONotifying_前缀），在这个派生类中重写基类中任何被观察属性的 setter 和 get 方法。保证 set 前后可以发出通知
- 系统将这个对象的 isa 指针指向这个新诞生的派生类
- 怎么知道把变化通知给谁呢?
  - 每个 NSobject 都有属性:observationInfo
  - observationInfo
  	- 有个属性指向 observer（不是strong，是assign，使用不当会产生crash），重写的 set被调用的时候会调用 observationInfo 中 observer 的 observeValueForKeyPath方法
  - observationInfo 还包含了 keypath 等信息

### 添加多个观察对象时候原理

```  
- (instancetype)init {
    if (self = [super init]) {
        [self addObserver:self forKeyPath:@"vc.studant" options:NSKeyValueObservingOptionNew context:nil];
        [self addObserver:self forKeyPath:@"vc" options:NSKeyValueObservingOptionNew context:nil];
    }
    return self;
}  
```  
![](https://pic.downk.cc/item/5fbcafeab18d627113326228.jpg)

### 路径 kvo 原理:self addobservr self forkeypath A.B

```  
- (instancetype)init {
    if (self = [super init]) {
        [self addObserver:self forKeyPath:@"vc.studant" options:NSKeyValueObservingOptionNew|NSKeyValueObservingOptionOld context:nil];
    }
    return self;
}
```  
![](https://pic.downk.cc/item/5fbcb039b18d6271133279e0.jpg)


- self 的 isa 的指向被更改成指向 NSKVONotifying_ 前缀的类(重写set get指针,self.observationInfo被赋值,存储观察者信息)
  - self.vc 被更改的时候也会触发kvo,虽然监听的是 vc.student
    - 但是也合理,因为vc变了.vc.student肯定也变了
- 当self.vc 不为空的时候(被赋值的时候),会触发
  - self.vc 的 isa指向被更改成指向 NSKVONotifying_前缀的类. self.vc,observationInfo 被赋值,存储观察者信息:观察者是 self.observationInfo的 NSKeyValueObservance;
  - 当 student 被更改的时候,触发 self.vc 的被重写的 set 方法,然后调用观察者 self.observationInfo 的 observe 实例的 observeValueForKeyPath方法(见上面堆栈)
    - observationInfo 的 NSKeyValueObservance 的 observeValueForKeyPath 内部会调用 observationInfo 的 observe的 observeValueForKeyPath方法
- 实现了监听属性的属性,或者更深层的嵌套属性

### 路径kvo原理:self.A keypathB

self addobserve self forkepath: A.B 与 self.A addobserve self for keypath B 两种方法实现效果不一样，前一种已经说过了,后面一种如下:  
![](https://pic.downk.cc/item/5fbcb4f7b18d62711333cb15.jpg)
![](https://pic.downk.cc/item/5fbcb512b18d62711333d283.jpg)

## 关键点

### 什么时候添加KVO  

self.A addobserve self forkeypath c 这种一定要保证A不为空,否则添加kvo失败

### KVO不移除造成的crash问题

#### 存在的问题 

为什么KVO不移除会造成crash，还存在其他问题吗？  
1. crash的原因  
	因为被观察者没有strong或者weak持有观察者，导致观察者释放后，再继续想其发通知会crash,因此需要在观察这释放前，移除观察者  
2. 其他的问题  
	添加多次需要移除多次:因为self.obserinfo没有去重机制;不能移除多次,否则数组找不到crash

### 如何解决 

经典做法：FBKVO(https://github.com/facebook/KVOController)   

# weak 原理

##  原理

为了管理所有对象的引用计数和 weak 指针，苹果创建了一个全局的 SideTables，虽然名字后面有个"s"不过他其实是一个全局的 Hash 表，里面的内容装的都是 SideTable 结构体而已。它使用对象的内存地址当它的 key。管理引用计数和 weak 指针就靠它了。  

当我们通过 SideTables[key] 来得到 SideTable 的时候，SideTable 的结构如下:  

```
struct SideTable {
    
	spinlock_t slock;
	RefcountMap refcnts;
	    
	weak_table_t weak_table;

	SideTable() {
	        
		memset(&weak_table, 0, 
		sizeof(weak_table));
	    
	}

	~SideTable() {
	    _objc_fatal("Do not delete SideTable.");
	}

	    
	void lock() { slock.lock(); }
	    
	void unlock() { slock.unlock(); }
	    
	void forceReset() { slock.forceReset(); }

	    
	// Address-ordered lock discipline for a pair of side tables.

	    
	template<HaveOld, HaveNew>
	    
	static void lockTwo(SideTable *lock1, SideTable *lock2);
	    
	template<HaveOld, HaveNew>static void unlockTwo(SideTable *lock1, SideTable *lock2);
};

``` 

SideTable 主要分为 3 部分：  

- weak_table_t: weak 引用的全局 hash 表
- RefcountMap : 引用计数的 hash 表
- slock: 保证原子操作的自旋锁

1. 引用计数器 RefcountMap：refcnts  
    
对象具体的引用计数数量是记录在这里的。  
    
这里注意 RefcountMap 其实是个C++的Map。为什么Hash以后还需要个Map？其实苹果采用的是分块化的方法。举个例子:  

假设现在内存中有16个对象。  
    
0x0000、0x0001、...... 0x000e、0x000f  

咱们创建一个SideTables[8]来存放这16个对象，那么查找的时候发生Hash冲突的概率就是八分之一。 假设SideTables[0x0000]和SideTables[0x0x000f]冲突,映射到相同的结果:  
SideTables[0x0000] == SideTables[0x0x000f]  ==> 都指向同一个SideTable  

苹果把两个对象的内存管理都放到里同一个SideTable中。你在这个SideTable中需要再次调用table.refcnts.find(0x0000)或者table.refcnts.find(0x000f)找到他们真正的引用计数器。这里是一个分流。内存中对象的数量实在是太庞大了我们通过第一个Hash表只是过滤了第一次，然后我们还需要再通过这个Map才能精确的定位到我们要找的对象的引用计数器。  

2. 维护weak指针的结构体 weak_table_t weak_table  

这里是一个两层结构:  
+ 第一层结构体中包含两个元素:
	+ 第一个元素 weak_entry_t *weak_entries;是一个数组,上面的 RefcountMap是要通过find(key)来找到精确的元素的。weak_entries则是通过循环遍历来找到对应的entry。
	+ 第二个元素 num_entries 是用来维护保证数组始终有一个合适的size。比如数组中元素的数量超过3/4的时候将数组的大小乘以2。  
+ 第二层 weak_entry_t 的结构包含3个部分:  
	+ referent:被指对象的地址。前面循环遍历查找的时候就是判断目标地址是否和他相等。
	+ referrers:可变数组,里面保存着所有指向这个对象的弱引用的地址。当这个对象被释放的时候，referrers里的所有指针都会被设置成nil。
 	+ inline_referrers:只有4个元素的数组，默认情况下用它来存储弱引用的指针。当大于4个的时候使用referrers来存储指针。


 ![](https://pic.downk.cc/item/5fbcb955b18d627113350ffa.jpg)

# KVC 原理

## 简单使用

KVC 全称是 Key Value Coding ，定义在 NSKeyValueCoding.h 文件中，是一个非正式协议。KVC 提供了一种间接访问其属性方法或成员变量的机制，可以通过字符串来访问对应的属性方法或成员变量。常用的几组函数：  

```  
- (nullable id)valueForKey:(NSString *)key; //直接通过Key来取值
- (void)setValue:(nullable id)value forKey:(NSString *)key; //通过Key来设值
- (nullable id)valueForKeyPath:(NSString *)keyPath; //通过KeyPath来取值
- (void)setValue:(nullable id)value forKeyPath:(NSString *)keyPath; //通过KeyPath来设值

//异常处理：  
- (nullable id)valueForUndefinedKey:(NSString *)key; //如果Key不存在，且没有KVC无法搜索到任何和Key有关的字段或者属性，则会调用这个方法，默认是抛出异常。
- (void)setValue:(nullable id)value forUndefinedKey:(NSString *)key; //和上一个方法一样，但这个方法是设值。
- (void)setNilValueForKey:(NSString *)key;
//针对集合类的特殊函数：需要注意的是，虽然看到dictionary的字样，下面两个方法并不是字典的方法。  
- (NSDictionary<NSString *, id> *)dictionaryWithValuesForKeys:(NSArray<NSString *> *)keys;
- (void)setValuesForKeysWithDictionary:(NSDictionary<NSString *, id> *)keyedValues;

```  


## 原理  

### 搜索路径
1. value for key(之所以是如下的顺序,猜测原因是如果属性实现了set get,这样不影响kvo)  
	方法：getKey key iskey——属性：key, isKey key isKey
2. setvaluefoekey(同样为了不耽误KVO)    
	方法 setKey ——属性：```_key, isKey,key,isKey```  

+ [obj valueForKey:key]

	第一步，首先会搜索getter方法（3个方法）。优先级 -getKey > Key > isKey。

	第二步，若找不到找getter 方法 ，则寻找+(BOOL)accessInstanceVariablesDirectly;方法。如果类方法返回NO,跳过第三步。

	第三步，寻找相关的成员变量（4个）。优先级 ```_key > _isKey > key > isKey```。

	第四步，还没找到的话，查找countOfKey、enumeratorOfKey、memberOfKey格式的方法。如果这三个方法都找到，那么就返回一个可以响应NSSet所有方法的代理集合。

	如果以上方法都拿不到，调用valueForUndefinedKey。
+ [obj setValueForKey:key]的搜索路径
	第一步，搜索setter相关方法（2个方法）。优先级setKey > setIsKey。

	第二步，寻找+(BOOL)accessInstanceVariablesDirectly;方法如果返回NO，跳过第三步

	第三步，寻找相关的成员变量（4个）。优先级```_key > _isKey > key > isKey```。

	如果没有找到成员变量，调用setValue:forUnderfinedKey:


###  KVC处理非对象和自定义对象 

valueForKey：总是返回一个id对象，如果原本的变量类型是值类型或者结构体，返回值会封装成 NSNumber 或者 NSValue 对象。这两个类会处理从数字，布尔值到指针和结构体任何类型。然后开以者需要手动转换成原来的类型。尽管 valueForKey：会自动将值类型封装成对象，但是 setValue：forKey：却不行。你必须手动将值类型转换成 NSNumber 或者 NSValue 类型，才能传递过去。对于自定义对象，KVC也会正确地设值和取值。因为传递进去和取出来的都是id类型，所以需要开发者自己担保类型的正确性，运行时 Objective-C 在发送消息的会检查类型，如果错误会直接抛出异常。


### KVC性能

根据上面 KVC 的实现原理，我们可以看出 KVC 的性能并不如直接访问属性快，虽然这个性能消耗是微乎其微的。所以在使用 KVC 的时候，建议最好不要手动设置属性的setter、getter，这样会导致搜索步骤变长。
而且尽量不要用 KVC 进行集合操作，例如 NSArray、NSSet 之类的，集合操作的性能消耗更大，而且还会创建不必要的对象  

### 私有访问

根据上面的实现原理我们知道，KVC 本质上是操作方法列表以及在内存中查找实例变量。我们可以利用这个特性访问类的私有变量，例如下面在.m中定义的私有成员变量和属性，都可以通过 KVC 的方式访问。
这个操作对 readonly 的属性，@protected 的成员变量，都可以正常访问。如果不想让外界访问类的成员变量，则可以将 accessInstanceVariablesDirectly属 性赋值为NO。


### 通过KVC实现集合的KVO

iOS 中 KVO (key-value-observing) 的原理，简单来说就是重写了被观察属性的 set 方法。自然一般情况下只有通过调用 set 方法对值进行改变才会触发 KVO，直接访问实例变量修改值是不会触发 KVO 的。  

对于 NSMutableArray 内容的变化进行观察，是我们比较常见的一个需求。但是在调用它的 addObject、removeObject 系列方法时，并不会触发它自己的 set 方法。所以，对一个可变数组进行观察，在它加减元素时不会收到期望的消息。  

那么，该如何实现对 NSMutableArray 的内容的 KVO 呢？官方为我们提供了这个方法。  


```  
- (NSMutableArray *)mutableArrayValueForKey:(NSString *)key
```  

像之前一样，为可变数组添加 KVO。在加减元素时，使用这个方法来获取我们要进行操作的可变数组，便可以像普通的属性一样，收到它变化的消息。  

举个例子，myItems 是我们要进行 KVO 的一个属性，它的定义如下：  

```
@property (nonatomic, strong) NSMutableArray *myItems;  
```  

在它进行添加元素时，使用如下方法：  

```  
[[self mutableArrayValueForKey:@"myItems"] addObject:item];
```  

这样，我们便实现了对 NSMutableArray 的内容的 KVO。这里解释一下,为什么这样就实现 KVO 了。  

这和 mutableArrayValueForKey 的搜索路径相关：  
一个 class 有个 NSMutableArray 属性 myItems ,我们 addObject 其实调用的是 myItems 的get函数,没有调用 set 函数所以没有触发 kvo。mutableArrayValueForKey 可以实现KVO的原因:  
![](https://pic.downk.cc/item/5fbcbdf1b18d627113365886.jpg)


# 参考

<https://juejin.cn/post/6844904158366007310>



