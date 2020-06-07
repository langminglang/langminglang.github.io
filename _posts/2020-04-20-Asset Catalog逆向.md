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
本系列文章主要介绍 Xcode 对图片的处理 和图片相关的介绍：  

+ Xcode 对图片的处理 
	+ Asset Catalog 逆向
	+ UIImage 加载图片的相关方法、加载流程、缓存原理等
+ 图片相关知识
	+ 几种图片格式的对比
	+ 图片编码、无损压缩与有损压缩
	+ 图片处理相关工具
本篇主要介绍 Asset Catalo 逆向相关。

# What are Asset Catalogs?
## 简单介绍

在 Xcode  Asset Catalogs 里，一个名字的 asset 可以对应多个资源。对于图片来说最常见的就是 @2x 和 @3x 这种 Scale 的区别。其他还有比如可以根据屏幕是否支持 Display P3 来分成支持和不支持的两套图、根据本地话语言是否是从左向右还是从右向左来分成两套图、根据时 iPhone 还是 iPad 或者其他设备分成多套图、根据 Size Class 是否是 Compact 分两套图等等。最终运行的时候，UIKit 根据当前设备的属性来寻找最适合的资源。

![](https://pic.downk.cc/item/5e9da07dc2a9a83be5b6a7a2.jpg)


## 参考
更详尽的知识参考：

+ <https://help.apple.com/xcode/mac/current/#/dev10510b1f7>
+ <https://developer.apple.com/library/archive/documentation/Xcode/Reference/xcode_ref-Asset_Catalog_Format/index.html>

# What is a car file?
>
+ the asset catalogs containing the various assets (images, icons, textures, …) are not simply copied to the app bundle but they are compiled as car files.  
+ **Xcode** lets you edit your asset catalogs and compile them.  
+ **actool** lets you compile, print, update, and verify asset catalogs.    
+ **assetutil** lets you process car files. It can remove unneeded assets from a car file but it can also parse a car file and produce a JSON output.  
	+ Running assetutil -I Assets.car will print some interesting information about the car file

包含 Assets 的 Asset Catalog 的边缘产物为 Assets.car 
可以通过 ```  UIImage *myImage = [UIImage imageNamed:@"MyImage"];```  使用Assets.car 中的图片。当改方法被调用时，实际执行了下面：
>The private CoreUI.framework (/System/Library/PrivateFrameworks/CoreUI.framework) is asked to give the best UIImage corresponding to the asset named MyImage. MyImage is the Asset Name, also called Facet Name. **The car file can contain multiple images for a given asset name**: @1x resolution, @2x resolution, @3x resolution, dark mode, … **These representations of the asset are called renditions**. Each rendition has a unique identifier called the rendition key. The rendition key is in fact a list of attributes describing the properties of the rendition: original facet, resolution, …

## What is a BOM file?
BOM 文件在开头有个文件头，包含信息：  

+ Block Table 的偏移和长度  
+ TOC（Table of content）偏移和长度  

根据 BOM文件的文件头，可以找到和读取 Block Table 和TOC
### TOC
TOC 是一个key，value的 类似于map结构：  

+ key 是Block的名字
+ value是BLock的ID  

可以根据 读取文件头——读取TOC部分——根据Block的name——得到BLOCK的ID

```  
//私有API：跟好Block name 获取Block ID  
BOMBlockID BOMStorageGetNamedBlock(BOMStorage inStorage, const char *inName)   
```  

### Block Table
首先维护了一个数据结构对应的数据，该数据结构包含。

+ Block的数量  
+ 每个block的起始位置、长度、ID  

这样可以实现了根据ID，确定BLOCK存储在哪里  

```  
//私有API：把 块 ID 对应的数据拷贝到 outData  
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
	+ Block 是一个节点，存储了改Block对应的ID、是否有孩子、有孩子存储所有孩子的Block ID（根据上述方法确定对应Block对应的起始位置和长度）、没有孩子则存储该节点的内容（节点内容是key value 对）
	+ Tree 遍历所有节点得到的是数组，数组里面是 key value对

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

![](https://pic.downk.cc/item/5e9da2dac2a9a83be5b8b7a5.jpg)
## car file 的 各个部分
Aseets.car 完全遵循 BOM的格式，其中定义的TOC name 如下：  

+ 普通数据结构
	+ CARHEADER
	+ EXTENDED_METADATA
	+ KEYFORMAT
	+ CARGLOBALS
	+ KEYFORMATWORKAROUND
	+ EXTERNAL_KEYS
+ Tree 类型的
	+ 	FACETKEYS
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
	+ 存储了每个Asset 的一些 catalog 属性，每个asset设置成什么，都在这里体现（上午 asset catalog 介绍中介绍了一个asset具体可以设置哪些属性）

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
### KEYFORMAT Block
+ KEYFORMAT Block 主要负责存储 Asset Catalog 使用过的 renditionKeyTokens，
	+ renditionKeyTokens 有哪几种选择：上面介绍的enum RenditionAttributeType，Asset Catalog 使用了几种，该数据结构中的renditionKeyTokens 数组出现几种。
+ 下面要介绍的 RENDITION Tree 中的 key 是renditionKeyTokens 数组中的 renditionKeyTokens 对应的value。
	+ RENDITION Tree 存储了image 等对应的数据，每个image 都对应 Tree 中的一对 key value。key就是该image的renditionKeyTokens值（catalog设置值）
+ 可以实现：已知 asset name——访问 FACETKEYS Tree，确定该 asset name 对应的几种 renditionKeyTokens 的value——遍历RENDITION tree的value——取到对应的data

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

其中 CUIThemePixelRendition 是image 对应的type，且比较复杂，CUIThemePixelRendition type 的header 是个数据结构  

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

+ sizeondisk = rendition Tree 的value 的大小 = csiheader length（184）+ renditionLength + TLV length

	
# 关于 Assets.car 文件结构的一些思考
为什么 Assets.car 要使用如上所属的结构呢？这个需要从Xcdoe使用Asset.car里面资源的时候分析
+ 加载
	+ 比如对图片的加载，可以先加载进 rendition Data csiheader，实现懒解码，读取图片更快，图片将要显示的时候在读取
	+ 减少IO，读取整个Asset.car文件，而不是一个image一个image的读取？
+ rendition keys 的应用
	+ asset catalog   有很多选择因子，BOM结构可以快速找到与因子对应的图片

本节的内容将在分析xcode对图片的读取一文详细介绍

	
# 参考	
+ <https://blog.timac.org/2018/1018-reverse-engineering-the-car-file-format/>
	+ 提供了工具：以json形式看asset.car 的Block 和 Tree
+ <https://blog.timac.org/2018/1112-quicklook-plugin-to-visualize-car-files/#getting-the-asset-variations>
+ <https://developer.apple.com/videos/play/wwdc2018/227/>

# 疑问
+ 执行 xcrun assetutil --info Assets.car 命令得到 Assets.car 的信息，回包含packimage
	+ packimage 回对同类型近似图片处理，相同的保存一份
	+ 那么问题：现在分析Assets.car的结构发现，每个asset对应一个 rendition data （key：value），那上面的packimage怎么实现呢？
  
