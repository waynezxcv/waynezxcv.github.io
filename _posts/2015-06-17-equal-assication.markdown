---
layout:     post
title:      "OC中的对象等同、关联对象以及方法调配"
subtitle:   "OC中的对象等同、关联对象以及方法调配"
date:       2015-06-17
author:     "waynezxcv"
header-img: "img/post.png"
tags:
    - Objective-C
---


### OC中的对象等同

* == 操作符比较的是两个指针本身。
* ``isEqual：``方法判定两个对象相等，那么hash方法返回同一个值。但是如果两个对象hash方法返回同一个值，isEqual：方法未必认为他们相同。

一个复写hash方法的例子：

```
- (NSUInteger)hash {
	NSUInteger property1Hash = [_property1Hash hash];
	NSUInteger property2Hash = [_property2Hash hash];
	return property1Hash^property2Hash;
}
```
***

### 关联对象AssociatedObject

使用关联对象可以给一个类别来添加属性。

设置关联对象值时，所用的key是个不透明指针。如果在NSDictionary中，两个key的isEqual：方法认为他们相同，则两个键会匹配到同一个值，**但是在设置关联对象时，若要匹配到同一个值，则两者必须使用完全相同的指针才行。这就是为什么通常设置关联对象时，通常使用静态全局变量做键。**

SDWebimage中，一个使用关联对象的例：

```

static char loadOperationKey;

- (NSMutableDictionary *)operationDictionary {
    NSMutableDictionary *operations = objc_getAssociatedObject(self, &loadOperationKey);
    if (operations) {
        return operations;
    }
    operations = [NSMutableDictionary dictionary];
    objc_setAssociatedObject(self, &loadOperationKey, operations, OBJC_ASSOCIATION_RETAIN_NONATOMIC);
    return operations;
}


```

***

### 方法调配Method Swizzling

类的方法列表会把选择子的名称映射到相关的方法实现上，使得“动态消息派发系统”能够据此找到应该调用的方法。这些方法均以函数指针的形式来表示，这种指针叫做IMP。通过类的方法列表，可以方便的给类添加新方法、交换两个方法的实现。

使用示例


```

+ (void)load {
    [super load];
    Method originMethod = class_getInstanceMethod([self class],NSSelectorFromString(@"o_method"));
    
    Method newMethod = class_getInstanceMethod([self class], NSSelectorFromString(@"n_method"));
    
    if (!class_addMethod([self class], @selector(n_method), method_getImplementation(newMethod), method_getTypeEncoding(newMethod))) {
        method_exchangeImplementations(newMethod, originMethod);
    }
}

- (void)n_method {
	//新的方法实现...
}

```

***

## 参考

《Effective Objective-C 2.0》
