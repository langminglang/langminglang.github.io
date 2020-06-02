---
layout:     post
title:      python 中的 unicode和编码
subtitle:   
date:       2018-01-04
author:     BY
header-img: img/post-bg-BJJ.jpg
catalog: true
tags:
    - python
    - 编码
    
---

### 目录

- unicode和编码
- unicode 和string
- python中判断相等
- 总结


## unicode和编码
> UTF-8 and Unicode cannot be compared. UTF-8 is an encoding used to translate numbers into binary data. Unicode is a character set used to translate characters into numbers.

UTF-8 是一种编码方式， 它主要是用来解决序号到二进制数据之间的转化问题。这里的序号怎么理解呢？ 比如那 ascii 编码为例， 数字 0 在 ascii 码表中的序号是 60，那么 ascii编码要解决的问题就是将 60 转为为二进制，而后进行存储和传输。因为 ascii 所表示的字符集非常有限， 不到 256 个， 如果需要用序号来标识一个字符的话，0 - 255 就够用了，将小于 256 的数字转化为二进制最节约的方式就是采用一个 byte， 所以 ascii 编码方式下每个字符最后存储所占用的大小就是 1 byte。 但是对于中文 ascii 就无能为力了， 因为中文的字数远远超过 256 个， 导致序号不够用了，所以我们要引入更多的序号，这就诞生了例如 GBK ，UTF-8 等编码方式。最大的序号也就决定了编码方式中一个字符所占用的最小空间， 当然可以采用变长编码的方式节约空间。

Unicode 是字符集， unicode 的引入是为了囊括世界上所有语言的字符，如 Unicode 字符列表。Unicode 字符集引入了字符到序号的映射关系， 有了序号我们也就知道它代表哪个字符了。 有了序号之后我们可以采用我们需要的编码方式进行编码，例如 UTF-8，GBK，ascii 等。

总结起来就是 UTF-8 等编码方式解决的是二进制到序号之间的对应问题， Unicode 等字符集解决的是字符到序号之间的对应问题。

## unicode 和 stirng
+ byte string 里面存储的是unicode通过utf-8编码后得到的bytes，所以byte string解码(decode)后即可得到unicode
+ unicode是byte string通过utf-8解码后得到的，unicode用utf-8编码(encode)可以得到对应的bytes


## python中判断相等

```
import sys
import os
import requests
reload(sys)
sys.setdefaultencoding("utf-8")

print sys.getdefaultencoding()

print u"哈" == "哈"

结果：
utf-8
True
```  

```  
import sys
import os
import requests
#reload(sys)
#sys.setdefaultencoding("utf-8")

print sys.getdefaultencoding()

print u"哈" == "哈"
结果 
ascii
False
/Users/langminglang/Documents/jenkins_jiaoben/untitled.py:12: UnicodeWarning: Unicode equal comparison failed to convert both arguments to Unicode - interpreting them as being unequal
  print u"哈" == "哈"
```  

```  
import sys
import os
import requests

print sys.getdefaultencoding()
print u"哈".encode('utf-8') == "哈"
结果
ascii
True
```  
  
首先要明确

- 判断相等的时候实际上判断的是二进制是否相等
- python 中 string， 默认采用了 utf-8 的方式进行了编码
- 其他使用默认编码进行编码
- python sys.getdefaultencoding()： 获取默认编码， 可以看到默认编码为 ascii

所以，由上面的例子可以看出：

- "哈"是string,经过了这个过程：汉字——unicode序号——utf-8编码得到二进制
- u"哈" 是unicode，直接是个unicode序号，相当于string中间步骤的产物，只有一个过程：编码，默认是ascii编码
- 所有默认情况下，虽然二者对应的序号相同，但是由于采取的编码方式不同，所以二进制不同，不相等
- 若设定默认的编码方式为utf8，与string相同，则能得到相同的二进制


另外思考OC 为啥不需要编码，但是URL需要编码呢？

- OC内部，有定义了了统一的数据类型，每个类型用了约定好的编码方式，比较大小的时候还是比较的二进制，由于编码方式一样，所以不需要，在约定编码了
- 但是与服务端通信的时候，由于没有约定好的数据类型了，所以需要约定好编码方式


## 小结

- python 中需要约定好编码方式
- python 中直接比较 str 和 unicode 是很危险， 因为它的结果并不固定！


