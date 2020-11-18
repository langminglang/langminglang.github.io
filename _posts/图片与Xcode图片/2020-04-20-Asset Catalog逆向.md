---
layout:     post
title:      Asset Catalog 逆向
subtitle:   
date:       2020-03-23
author:     LML
header-img: img/post-bg-ios9-web.jpg
catalog: true
tags:
    - Xcode Image
---

# 前言
本系列文章主要介绍 Xcode 对图片的处理和图片相关知识：  

+ Xcode 对图片的处理 
	+ Asset Catalog 逆向
	+ UIImage 加载图片的相关方法、加载流程、缓存原理等
+ 图片相关知识
	+ 几种图片格式的对比
	+ 图片编码、无损压缩、有损压缩
	+ 图片处理相关工具

本篇主要介绍 Asset Catalo 逆向相关的知识。

# What are Asset Catalogs?  

## 简单介绍

在 Xcode Asset Catalogs 里，一个名字的 asset 可以对应多个资源。对于图片来说最常见的就是 @2x 和 @3x 这种 Scale 的区别。其他还有比如可以根据屏幕是否支持 Display P3 分成支持和不支持的两套图、根据本地语言是否是从左向右还是从右向左来分成两套图、根据是 iPhone 还是 iPad 或者其他设备分成多套图、根据 Size Class 是否是 Compact 分两套图等等。最终运行的时候，UIKit 根据当前设备的属性来寻找最适合的资源。

![](https://pic.downk.cc/item/5e9da07dc2a9a83be5b6a7a2.jpg)


## 参考  

更详尽的知识参考：

+ <https://help.apple.com/xcode/mac/current/#/dev10510b1f7>
+ <https://developer.apple.com/library/archive/documentation/Xcode/Reference/xcode_ref-Asset_Catalog_Format/index.html>

# What is a car file?  

>
+ The asset catalogs containing the various assets (images, icons, textures, …) are not simply copied to the app bundle but they are compiled as car files.  
+ **Xcode** lets you edit your asset catalogs and compile them.  
+ **Actool** lets you compile, print, update, and verify asset catalogs.    
+ **Assetutil** lets you process car files. It can remove unneeded assets from a car file but it can also parse a car file and produce a JSON output.  
	+ Running assetutil -I Assets.car will print some interesting information about the car file


Asset Catalog 的编译产物为 Assets.car。可以通过 ```  UIImage *myImage = [UIImage imageNamed:@"MyImage"];```  使用Assets.car 中的图片。当该方法被调用时，实际执行了下面：  

>The private CoreUI.framework (/System/Library/PrivateFrameworks/CoreUI.framework) is asked to give the best UIImage corresponding to the asset named MyImage. MyImage is the Asset Name, also called Facet Name. **The car file can contain multiple images for a given asset name**: @1x resolution, @2x resolution, @3x resolution, dark mode, … **These representations of the asset are called renditions**. Each rendition has a unique identifier called the rendition key. The rendition key is in fact a list of attributes describing the properties of the rendition: original facet, resolution, …

## What is a BOM file?  

BOM 文件在开头有个文件头，包含信息：  

+ magic、version等信息
+ Block Table 的偏移和长度  
+ TOC（Table of content）偏移和长度  

![](https://pic.downk.cc/item/5fae28461cd1bbb86b8d9c47.png)

根据 BOM 文件的文件头，可以找到和读取 Block Table 和 TOC

### TOC  

TOC 是一个key，value类似于map结构：  

+ key 是 Block 的名字
+ value 是 BLock 的 ID  

读取文件头——读取TOC部分——根据 Block 的 name——得到 BLOCK 的 ID

```  
//私有API：根据 Block name 获取 Block ID  
BOMBlockID BOMStorageGetNamedBlock(BOMStorage inStorage, const char *inName)   
```  

### Block Table  

维护了一个数据结构，该数据结构包含。

+ Block 的数量  
+ 每个 block 的起始位置、长度、ID  

这样可以实现了根据 ID，确定 BLOCK 存储在哪里  

```  
//私有API：把 Block ID 对应的数据拷贝到 outData  
int BOMStorageCopyFromBlock(BOMStorage inStorage, BOMBlockID inBlockID, void *outData)   
```   

使用举例  

```  
NSData *GetDataFromBomBlock(BOMStorage inBOMStorage, const char *inBlockName)
{
	NSData *outData = nil;
	
	BOMBlockID blockID = BOMStorageGetNamedBlock(inBOMStorage, inBlockName);
	size_t blockSize = BOMStorageSizeOfBlock(inBOMStorage, blockID);
	if(blockSize > 0)
	{
		void *mallocedBlock = malloc(blockSize);
		int res = BOMStorageCopyFromBlock(inBOMStorage, blockID, mallocedBlock);
		if(res == noErr)
		{
			outData = [[NSData alloc] initWithBytes:mallocedBlock length:blockSize];
		}
		
		free(mallocedBlock);
	}
	
	return outData;
}
```   
这些 Block 可以是  

+ 数据结构，存储数据  
+ Tree  
	+ Block 是一个节点，存储了该 Block 对应的 ID、是否有孩子、有孩子存储所有孩子的 Block ID（根据上述方法确定对应Block对应的起始位置和长度）、没有孩子则存储该节点的内容（节点内容是key value 对）
	+ 遍历 Tree 所有节点得到的是数组，数组里面是 key value 对

```   
// 提取 tree 类型结构的Block的key value 有相关私有API
// Accessing a BOM tree
BOMTree BOMTreeOpenWithName(BOMStorage inStorage, const char *inName, Boolean inWriting);
BOMTreeIterator BOMTreeIteratorNew(BOMTree inTree, void *, void *, void *);
Boolean BOMTreeIteratorIsAtEnd(BOMTreeIterator iterator);
void BOMTreeIteratorNext(BOMTreeIterator iterator);

// Accessing the keys and values of a BOM tree
void * BOMTreeIteratorKey(BOMTreeIterator iterator);
size_t BOMTreeIteratorKeySize(BOMTreeIterator iterator);
void * BOMTreeIteratorValue(BOMTreeIterator iterator);
size_t BOMTreeIteratorValueSize(BOMTreeIterator iterator);
```   
使用举例：  

```   
typedef void (^ParseBOMTreeCallback)(NSData *inKey, NSData *inValue);
void ParseBOMTree(BOMStorage inBOMStorage, const char *inTreeName, ParseBOMTreeCallback keyValueCallback)
{
	NSData *keyData = nil;
	NSData *keyValue = nil;
	
	// Open the BOM tree
	BOMTree bomTree = BOMTreeOpenWithName(inBOMStorage, inTreeName, false);
	if(bomTree == NULL)
		return;

	// Create a BOMTreeIterator and loop until the end
	BOMTreeIterator	bomIterator = BOMTreeIteratorNew(bomTree, NULL, NULL, NULL);
	while(!BOMTreeIteratorIsAtEnd(bomIterator))
	{
		// Get the key
		void * key = BOMTreeIteratorKey(bomIterator);
		size_t keySize = BOMTreeIteratorKeySize(bomIterator);
		keyData = [NSData dataWithBytes:key length:keySize];
		
		// Get the value associated to the key
		size_t valueSize = BOMTreeIteratorValueSize(bomIterator);
		if(valueSize > 0)
		{
			void * value = BOMTreeIteratorValue(bomIterator);
			if(value != NULL)
			{
				keyValue = [NSData dataWithBytes:value length:valueSize];
			}
		}
		
		if(keyData != nil)
		{
			keyValueCallback(keyData, keyValue);
		}
		
		// Next item in the tree
		BOMTreeIteratorNext(bomIterator);
	}
}
```   

![](https://pic.downk.cc/item/5e9da2dac2a9a83be5b8b7a5.jpg)  

## car file 的 各个部分

Aseets.car 完全遵循 BOM 格式，其中定义的 TOC name 如下：  

+ 普通数据结构
	+ CARHEADER
	+ EXTENDED_METADATA
	+ KEYFORMAT
	+ CARGLOBALS
	+ KEYFORMATWORKAROUND
	+ EXTERNAL_KEYS
+ Tree 类型的
	+  FACETKEYS
	+  RENDITIONS
	+  APPEARANCEKEYS
	+  COLORS
	+  FONTS
	+  FONTSIZES
	+  GLYPHS
	+  BEZELS
	+  BITMAPKEYS
	+  ELEMENT_INFO
	+  PART_INFO  
  
本文主要介绍几个重点的，不全部介绍
  
### CARHEADER block  

>The CARHEADER block contains information about the number of assets in the file as well as versioning information. It has a fixed size of 436 bytes.  

```   
struct carheader
{
    uint32_t tag;								// 'CTAR'
    uint32_t coreuiVersion;
    uint32_t storageVersion;
    uint32_t storageTimestamp;
    uint32_t renditionCount;
    char mainVersionString[128];
    char versionString[256];
    uuid_t uuid;
    uint32_t associatedChecksum;
    uint32_t schemaVersion;
    uint32_t colorSpaceID;
    uint32_t keySemantics;
} __attribute__((packed))
```   

### EXTENDED_METADATA block  

> The EXTENDED_METADATA block has a fixed size of 1028 bytes and contains a couple of extra information:  

```  
struct carextendedMetadata {
    uint32_t tag;								// 'META'
    char thinningArguments[256];
    char deploymentPlatformVersion[256];
    char deploymentPlatform[256];
    char authoringTool[256];
} __attribute__((packed));
```   
### APPEARANCEKEYS Tree  

存储的 Key vaule 对类似于下面  

```  
Tree APPEARANCEKEYS
	'NSAppearanceNameAccessibilityDarkAqua': 6  
	 'NSAppearanceNameAccessibilitySystem': 3  
	 'NSAppearanceNameDarkAqua': 1  
	 'NSAppearanceNameSystem': 0  
```  

### FACETKEYS Tree

+ Key：asset name  
+ Value： attributes of image in asset ，is renditionkeytoken struct. 
	+ 存储了每个Asset 的一些 catalog 属性，每个 asset 设置成什么，都在这里体现（上面 asset catalog 介绍中介绍了一个asset具体可以设置哪些属性）

```  
struct renditionkeytoken {
    struct {
    	uint16_t x;
        uint16_t y;
    } cursorHotSpot;

	uint16_t numberOfAttributes;

    struct renditionAttribute attributes[];

} __attribute__((packed));  

struct renditionAttribute {
	uint16_t name;
	uint16_t value;
} __attribute__((packed));

name:
enum RenditionAttributeType
{
	kRenditionAttributeType_ThemeLook 				= 0,
	kRenditionAttributeType_Element					= 1,
	kRenditionAttributeType_Part					= 2,
	kRenditionAttributeType_Size					= 3,
	kRenditionAttributeType_Direction				= 4,
	kRenditionAttributeType_placeholder				= 5,
	kRenditionAttributeType_Value					= 6,
	kRenditionAttributeType_ThemeAppearance			= 7,
	kRenditionAttributeType_Dimension1				= 8,
	kRenditionAttributeType_Dimension2				= 9,
	kRenditionAttributeType_State					= 10,
	kRenditionAttributeType_Layer					= 11,
	kRenditionAttributeType_Scale					= 12,
	kRenditionAttributeType_Unknown13				= 13,
	kRenditionAttributeType_PresentationState		= 14,
	kRenditionAttributeType_Idiom					= 15,
	kRenditionAttributeType_Subtype					= 16,
	kRenditionAttributeType_Identifier				= 17,
	kRenditionAttributeType_PreviousValue			= 18,
	kRenditionAttributeType_PreviousState			= 19,
	kRenditionAttributeType_HorizontalSizeClass		= 20,
	kRenditionAttributeType_VerticalSizeClass		= 21,
	kRenditionAttributeType_MemoryLevelClass		= 22,
	kRenditionAttributeType_GraphicsFeatureSetClass = 23,
	kRenditionAttributeType_DisplayGamut			= 24,
	kRenditionAttributeType_DeploymentTarget		= 25
};
```  
![](https://pic.downk.cc/item/5fae2df11cd1bbb86b8fea25.png)


### KEYFORMAT Block

+ KEYFORMAT Block 主要负责存储 Asset Catalog 使用过的 RenditionAttributeType name
	+ 上面介绍的 enum RenditionAttributeType，Asset Catalog 使用了几种，该数据结构中的 renditionKeyTokens 数组出现几种。
+ 下面要介绍的 RENDITION Tree 中的 key 是 renditionKeyTokens 数组中的 RenditionAttributeType 对应的value。
+ 可以实现：已知 asset name —— 访问 FACETKEYS Tree，确定该 asset name 对应的renditionKeyTokens 中出现的 RenditionAttributeType 的 value——遍历 RENDITION tree ——取到对应的data

``` 
struct renditionkeyfmt {
    uint32_t tag;								// 'kfmt'
    uint32_t version;
    uint32_t maximumRenditionKeyTokenCount;
    uint32_t renditionKeyTokens[];
} __attribute__((packed));
``` 
### RENDITION Tree

RENDITION Tree 的 key 上面已经介绍，其 value 部分主要分为三个部分

+ the csiheader
	+ 固定 184 bytes 大小，一些基本信息，对应的数据结构：  
	
```  
		struct csiheader {
	    uint32_t tag;								// 'CTSI'
	    uint32_t version;
	    struct renditionFlags renditionFlags;
	    uint32_t width;
	    uint32_t height;
	    uint32_t scaleFactor;
	    uint32_t pixelFormat;
		struct {
			uint32_t colorSpaceID:4;
			uint32_t reserved:28;
	    } colorSpace;
	    struct csimetadata csimetadata;
	    struct csibitmaplist csibitmaplist;
	} __attribute__((packed)); 

	struct csimetadata {
	 uint32_t modtime;  
	 uint16_t layout; 
	 uint16_t zero;  
	 char name[128]; 
	 } attribute((packed))

	struct csibitmaplist { 
	uint32_t tvlLength; // Length of all the TLV following the csiheader 
	uint32_t unknown; 
	uint32_t zero; 
	uint32_t renditionLength; 
	} attribute((packed));

```  
	
+ TLV (Type-length-value) 
	+ 长度在 csiheader 指定
	+ 包含一些基本信息，csiheader放不下的基本信息，key value 对，key 的选择  
 		
```
		 enum RenditionTLVType  
		 {
			kRenditionTLVType_Slices 				= 0x3E9,
			kRenditionTLVType_Metrics 				= 0x3EB,
			kRenditionTLVType_BlendModeAndOpacity	= 0x3EC,
			kRenditionTLVType_UTI	 				= 0x3ED,
			kRenditionTLVType_EXIFOrientation		= 0x3EE,
			kRenditionTLVType_ExternalTags			= 0x3F0,
			kRenditionTLVType_Frame					= 0x3F1,
			kRenditionTLVType_position              = 0x3F2，//如果以一张图是internal_reference类型图片，则TLV有该字段，表示了图片在packegimage 中的位置和大小
		};  
```  

+ the rendition data
	+ The rendition data can be seen after these complex structures. It contains a header specific to the type of the rendition （ type followed by the actual data either compressed or uncompressed）.The length is set in the renditionLength field of the csibitmaplist structure.  
	+ there are 21 types of renditions:  
		+ CUIRawDataRendition
		+ CUIRawPixelRendition
		+ CUIThemeColorRendition
		+ **CUIThemePixelRendition**
		+ CUIPDFRendition
		+ CUIThemeModelMeshRendition
		+ CUIMutableThemeRendition
		+ CUIThemeEffectRendition
		+ CUIThemeMultisizeImageSetRendition
		+ CUIThemeGradientRendition
		+ CUIExternalLinkRendition
		+ CUIThinningPlaceholderRendition
		+ CUIThemeTextureRendition
		+ CUIThemeTextureImageRenditio
		+ CUIInternalLinkRendition
		+ CUINameContentRendition
		+ CUIThemeSchemaRendition
		+ CUIThemeSchemaEffectRendition
		+ CUIThemeModelAssetRendition
		+ and 2 subclasses of CUIRawDataRendition:
			+ CUILayerStackRendition
			+ CUIRecognitionObjectRendition  

其中 CUIThemePixelRendition 是image 对应的 type，且比较复杂，CUIThemePixelRendition type 的 header 是个数据结构  

```  
struct CUIThemePixelRendition {
    uint32_t tag;					// 'CELM'
    uint32_t version;
    uint32_t compressionType;
    uint32_t rawDataLength;
    uint8_t rawData[];
} __attribute__((packed));

```    

> When a compression is used, the raw data is compressed and should be decoded with the corresponding algorithm. The decompression algorithms used is out of the scope of this article.  
> The rawData contains the real data - either uncompressed or compressed. If the data is compressed, you will need to decompress it using the algorithm specified in the compressionType field.

+ sizeondisk 
	+ rendition Tree 的value 的大小 = csiheader length（184）+ renditionLength + TLV length

# Xcode 对图片的处理

## 零散的图片
Xcode 工程中没有放入 Asset Catalog 中的图片仍然会以原格式存在与APP 文件中。

## Asset Catalog 中的图片
Asset Catalog 编码工具 actool 处理图片流程?

![](https://pic.downk.cc/item/5f980c7e1cd1bbb86b2c699c.jpg)


# CAR 对图片的优化和缓存

## 优化

按照上面的理解，CAR 中每张图都在 Rendition Tree 中占据 kye value 对（key:renditionKey;value:data),但是实际上 CAR 中还会额外生成 PackedImage。那么 PackedImage 是什么呢？

PackedImage 是一些小图（internal_reference 的图片）的拼接。拼接方式大概画一下是。

![](https://pic.downk.cc/item/5fae35f01cd1bbb86b92a28c.png)

PackedImage 的目的，猜测是在包大小和读取效率上有优化。

如上面介绍的，正常的图片的redition 分为三部分：csiheader、CLV、renditionData，但是有的图片没有第三部分Rendition data，他们的数据存储在 PackedImage 中。那么这种类型的图片有什么特征呢？当读取这种类型的图片是如何定位到他的真实数据呢？

这种类型的图片有什么标识？
the csiheader 的 meteddata 的 layout 的值为 0x3eb（是一些小图（internal_reference）

如何去PackedImage定位这种图的数据

tlv 是多个key value对，当key 为 kRenditionTLVType_position （0x3F2）的时候，value 标示了真实的数据所在的 PackedImage和其在 PackedImage 中的 positionx、positiony、width、heigh

![](https://pic.downk.cc/item/5fae3c891cd1bbb86b941f23.png)

##  CAR 对图片的缓存

### -[UIImage imageNamed:]对图片的缓存

使用 -[UIImage imageNamed:]方法加载的图片，会被缓存，但car 中的图片与非 car 中的图片缓存的结构是不一样的。
 
Car 中的图片被缓存在 CUIStructuredThemeStore 属性 cache 字典中，key是 reditionkey value 是 rendition data。每个 bundle 拥有一个 CUIStructuredThemeStore（bundle->_UIAssetManager->CUICatalog* _catalog->themethore）

非 Car 中的图片被缓存在 CUIMutableStructuredThemeStore 的属性 memorySore 字典中，key是 reditionkey, value 是rendition data。一个app 公用一个 CUIMutableStructuredThemeStore.

### Memory warning  + 进入后台时图片缓存的释放

CUIMutableStructuredThemeStore 清理缓存的方法

```  
-（void）clearRenditionCache
-（void）_removeRenditionInfoForImageWithName:
-（void）CUIMutableStructuredThemeStore removeImageNamed:withDescription:
```  


CUIStructuredThemeStore 清理缓存的方法

```  
-（void）clearRenditionCache
```  

模拟器 Debug 模拟memory Warning 后，发现 ：
- CUIStructuredThemeStore 调用了 clearRenditionCache 清理了全部的图片缓存
- CUIMutableStructuredThemeStore  只调用了后两个清理方法，没有调 clearRenditionCache
  - 这个后果是：没有实现全部图片被清理

### CUIStructuredThemeStore 缓存存在的问题

- CUIMutableStructuredThemeStore 会缓存两种图片
  - 一种是非 car 中通过  -[UIImage imageNamed:] 读取的图片，这些图片会在 memory warning + 进入后台的时候释放
  - 一种是 runtime image asset的图片，这类型的图片无论如何都不会被释放，进而造成内存占用暴涨

1. runtime image asset 类型的缓存不会被释放，那么这类型的图片怎么来的呢？

正常图片：<UIImage:0x7fb91cd31e70 named(main: test1) {54, 54}>

触发 CUIMUtableStructuredThemeStore 缓存 runtime image asset的图片：<UIImage:0x60000186bd50 anonymous {32, 32}>

anonymous image 调用 imageAsset 属性会导致 CUIMUtableStructuredThemeStore 缓存 runtime image asset 图片 

为什么anonymous image 调用 imageAsset 会这样还没有弄清楚，只发现：

非 anonymous 的 image 是存在 imageAsset 属性的，因为我们访问图片 imageAsset 属性前后抓取 memory graph ，内存中的 UIimageAsset 实例个数不变

而 anonymous 的 image的imageAsset 为nil，且为懒加载属性，因为我们访问图片imageAsset属性前后抓取memory graph ，内存中的 UIimageAsset 实例个数增加1

至于为什么会存在 anonymous 图片还没有搞清楚

2. 为什么没有释放呢？

没有释放的原因猜测是苹果的bug。

CUIMutableStructuredThemeStore 清理缓存的方法

``` 
-（void）clearRenditionCache
-（void）_removeRenditionInfoForImageWithName:
-（void）CUIMutableStructuredThemeStore removeImageNamed:withDescription:

```   

CUIStructuredThemeStore 清理缓存的方法

``` 
-（void）clearRenditionCache

``` 


Memory warning 或者进入后台的时候触发了 CUIStructuredThemeStore 调用  clearRenditionCache释放了全部缓存

但是 CUIMutableStructuredThemeStore 只调用了后两个方法，没有释放 runtime image asset类型图片的缓存


3. 能不能释放？有没有办法释放？

强制调用 CUIMutableStructuredThemeStore 的 clearRenditionCache 能释放其全部缓存


# 小结
 
如果想要理解 CAR 文件的结构，需要先了解 BOM 结构。CAR 文件利用了 BOM 结构，他可以很快的根据 BLOCk name 确定 BLOCKID，进而确定数据：  

+ 根据 CAR 文件头部中 TOC 结构，确定 、KEYFORMAT 的ID
+ 根据 CAR 文件头部中的 Block Table 结构，确定 FACETKEYS Tree、KEYFORMAT Block 的位置
+ 结合 FACETKEYS Tree、KEYFORMAT Block 确定要读取的 image 的 RenDition Key
+ 和上面一样的步骤，访问Rendition Tree，根据第三步确定的RenDition Key，找到 Rendition data，即拿到了图片数据

	
# 参考	
+ <https://blog.timac.org/2018/1018-reverse-engineering-the-car-file-format/>
	+ 提供了工具：以json形式看asset.car 的Block 和 Tree
+ <https://blog.timac.org/2018/1112-quicklook-plugin-to-visualize-car-files/#getting-the-asset-variations>
+ <https://developer.apple.com/videos/play/wwdc2018/227/>
+ https://ide.kaitai.io/# 可视化CAR的工具

  
