---
layout:     post
title:      "ARC下的内存管理"
subtitle:   "ARC下的内存管理"
date:       2015-06-15
author:     "waynezxcv"
header-img: "img/post.png"
tags:
    - iOS
---


### ARC下内存管理的规则
* 不能使用retain/release/reatinCount/auotorelease
* 不能使用NSAllocateObject/NSDeallocateObjcet
* 须遵守内存管理的方法命名规则

```
	（1）以alloc、new、copy、mutableCopy开始的方法，在返回对象时，必须返回给调用方所应当持有的对象。
	（2）以init开始方法，必须是实例方法，并且必须要返回对象。
```

* 不要显示调用dealloc方法
* 使用@auotoreleasepool代替NSAutoReleasePool
* 不能使用区域NSZone
* 对象类型变量，不能作为C语言结构体（struct，union）的成员
* 显示转换id和void*

***

### 持有修饰符说明

* \__strong:默认，类似C语言中的自动变量（局部变量），持有对象，在作用域结束时释放。
* \__weak：不持有对象，在对象释放时指向nil，iOS5，OS X Lion及之后版本才能使用
* \__unsafe__unretain:类似__weak,不持有对象，但是不能在对象释放后自动指向nil。
* \__autorelease ：等同于非ARC下的autorelease方法，把对象注册到自动释放池当中。一般不需要显式使用。