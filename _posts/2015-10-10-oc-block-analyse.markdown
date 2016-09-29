---
layout:     post
title:      "深入理解Block"
subtitle:   "从Block的使用方法到实现原理,阅读《Objective-C高级编程》Blocks相关章节笔记。"
date:       2015-10-10
author:     "waynezxcv"
header-img: "img/post.png"
tags:
    - Objective-C
---


## 目录

1. [Block的定义和使用方法](#block)
2. [Block的实质](#block1)
3. [截取局部变量的过程](#block2)
4. [关于__block说明符](#block3)
5. [Block的存储区域](#block4)
6. [捕获变量](#block5)
7. [Block循环引用](#block6)

***

## Block的定义和使用方法
<p id="block"></p>

### Block的定义
带有局部变量的匿名函数(C++和Python当中的Lambda函数)。

### Block语法
与一般的C语言函数相比，仅有两点不同:

* 由^开头。
* 没有函数名。


举个栗子：


```
^int(int count) {
	return count ++;
};
```

### Block类型的变量
举个🌰：

```
int (^Blk) (int count);
```

给Block类型的变量赋值

```
blk  = ^(int i) {
	return i ++;
};
```

### 截取局部变量
Block会保存局部变量定义时的瞬间值。

```
int main () {
	int i = 1;
	void (^block)() = ^{
	printf(" i = %d",i);
	};
	i = 2;//这里改变i的值。
	block();//这里调用block，但是结果是 "i = 1"，block会保存局部变量定义时的瞬间值,保存后就无法改写了，除非加上__block说明符
	return 0;
}
```

Block会保存局部变量定义时的瞬间值,保存后就无法改写了，除非加上__block说明符

```
int i = 0;
void (^block)() = ^{
	i = 1;//代码会产生编译错误。
};

//若果 int i 加上 __block说明符，则可在block表达式中对i 赋值。
__block int i = 0;
void (^block)() = ^{
	i = 1;//可以编译通过
};


```
虽然对Block外部的局部变量无法赋值，但是调用局部变量的方法是没有问题的。

```
NSMutableArray* array = [[NSMutableArray alloc] init];

(^block)() = ^{
		id object = [[NSObject alloc] init];
		[array addObject:object];//只是无法赋值，调用方法是可以的
};
```

***

## Block的实质
<p id="block1"></p>

使用clang（LLVM编译器）将含有Block语法的源代码转化成C++语言源代码

```
clang -rewrite-objc 源代码文件名
```

转换前源代码如下

```
int main() {
	void (^blk)(void) = ^{
		printf("Block\n");
	};
}
```
下面就讲转换后的源代码一一分析。

```
 ^{
	printf("Block\n");
};
```

转换后：

```
static void __main_block_func_0(struct __main_block_impl_0 *__cself) {
        printf("block\n");
}
```
通过Block实现的匿名函数，其实就是被当做简单的C语言函数来处理，根据Block语法数数的函数名，这里是main和该Block语法在该函数出现的顺序，生成了clang变换的函数名。该函数的参数\__cself相当于OC当中的self。

先看看第一个参数的声明：

```
struct __main_block_impl_0 *__cself
```
这是一个\__main_block_impl_0结构体的指针。该结构体声明如下：

```
struct __main_block_impl_0 {
  struct __block_impl impl;//成员变量impl
  struct __main_block_desc_0* Desc;//成员变量Desc

  __main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, int flags=0) {
    impl.isa = &_NSConcreteStackBlock;
    impl.Flags = flags;
    impl.FuncPtr = fp;
    Desc = desc;
  }//构造函数。

};
```
成员变量impl的结构体声明如下：

```
struct __block_impl {
  void *isa;//isa指针，表明Block其实就是一个Objective-C对象
  int Flags;//标志变量，在实现block的内部操作时会用到
  int Reserved;//预留变量
  void *FuncPtr;//函数指针
};
```
成员变量Desc的声明如下：

```
static struct __main_block_desc_0 {
  size_t reserved;//预留的变量
  size_t Block_size;//Block的大小
};

```
构造函数的调用时：

```
void (*blk)() = ((void (*)())&__main_block_impl_0((void *)__main_block_func_0, &__main_block_desc_0_DATA));
((void (*)(__block_impl *))((__block_impl *)blk)->FuncPtr)((__block_impl *)blk);
```
去掉转换部分后：

```
struct __main_block_imp1_0 tmp = __main_block_imp1_0(__main_block_func_0 ,  &__main_block_desc_0_DATA);
struct __main_block_imp1_0* blk = &tmp;
```

第一个参数 “\__main_block_func_0” 是由Block语法转换的C语言函数指针。
第二个参数 “&\__main_block_desc_0_DATA” 是作为静态全局变量初始化的__main_block_desc_0_DATA结构体的实例指针。

再来看看构造函数是如何初始化的。

```
  __main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, int flags=0) {
    impl.isa = &_NSConcreteStackBlock;
    impl.Flags = flags;
    impl.FuncPtr = fp;
    Desc = desc;
}
```
该结构体struct __block_impl,根据它的构造函数可知，它会进行如下的初始化

```
isa = &_NSConcreteStackBlock;
Flags = 0;
Reserved = 0;
FuncPtr = __main_block_func_0;
Desc =  &__main_block_desc_0_DATA;
FuncPtr = __main_block_func_0;
```
先看isa = &_NSConcreteStackBlock，_NSConcreteStackBlock相当于objc_class结构体实例。这表明，也就是说这个Block其实就是_NSConcreteStackBlock类的实例对象。
也就是说Block的实质，即为Objective-C的对象。

>runtime中对Objective-c的类声明如下

```
typedef struct objc_class {
	Class isa;//指向Class对象，表明这个实例的所属的类
	Class super_class;//指向父类，表明继承关系
	const char* name;//类名
	long version;
	long info;
	long instance_size;
	struct objc_ivar_list* ivars;这个类//所包含的成员变量数组
	struct objc_method_list* methods;//这个类所包含的方法数组
	struct objc_cache* chace;
	struct objc_protocol_list* protocols;//这个类所包含协议数组
};
```


而调用block时,就是简单的使用函数指针调用函数

```
((void (*)(__block_impl *))((__block_impl *)blk)->FuncPtr)((__block_impl *)blk);
```

***

## 截取局部变量的过程
<p id="block2"></p>

这次转换的源代码是

```
int main () {
    int i = 0;
    void (^blk)() = ^{
        printf("block:%d\n",i);
    };
    i ++;
    blk();
    return 0;
}
```

同样的，使用clang -rewrite-objc转换之后，变成了这样：

```
struct __main_block_impl_0 {
  struct __block_impl impl;
  struct __main_block_desc_0* Desc;
  int i;
  __main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, int _i, int flags=0) : i(_i) {
    impl.isa = &_NSConcreteStackBlock;
    impl.Flags = flags;
    impl.FuncPtr = fp;
    Desc = desc;
  }
};

static void __main_block_func_0(struct __main_block_impl_0 *__cself) {
  int i = __cself->i; // bound by copy

        printf("block:%d\n",i);
}


```
可以看到__main_block_impl_0结构体中较上次的转换结果中多了一个int i成员变量，且它在Block定义的时候，就已经对int i进行了赋值，保存到了结构体实例当中。

```
int i = __cself->i;
```
在blk()调用时，只是在使用__main_block_impl_0结构体内部的成员变量，这叫不难理解，为什么在执行了i++对i重新赋值之后，Block当中的i值并没有变化。

*** 

## 关于__block说明符
<p id="block3"></p>

再来看看带__block说明符的Block的转换情况。

这次的源代码是：

```
int main () {
    __block int i = 0;
    void (^blk)() = ^{
        printf("block:%d\n",i);
    };
    i ++;
    blk();
    return 0;
}
```

clang -rewrite-objc 转换后:

```

struct __Block_byref_i_0 {
  void *__isa;
__Block_byref_i_0 *__forwarding;
 int __flags;
 int __size;
 int i;
};

struct __main_block_impl_0 {
  struct __block_impl impl;
  struct __main_block_desc_0* Desc;
  __Block_byref_i_0 *i; // by ref
  __main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, __Block_byref_i_0 *_i, int flags=0) : i(_i->__forwarding) {
    impl.isa = &_NSConcreteStackBlock;
    impl.Flags = flags;
    impl.FuncPtr = fp;
    Desc = desc;
  }
};

static void __main_block_func_0(struct __main_block_impl_0 *__cself) {
  __Block_byref_i_0 *i = __cself->i; // bound by ref
        printf("block:%d\n",(i->__forwarding->i));
}


int main () {
    __attribute__((__blocks__(byref))) __Block_byref_i_0 i = {(void*)0,(__Block_byref_i_0 *)&i, 0, sizeof(__Block_byref_i_0), 0};
    void (*blk)() = ((void (*)())&__main_block_impl_0((void *)__main_block_func_0, &__main_block_desc_0_DATA, (__Block_byref_i_0 *)&i, 570425344));
    (i.__forwarding->i) ++;
    ((void (*)(__block_impl *))((__block_impl *)blk)->FuncPtr)((__block_impl *)blk);
    return 0;
}

```

我们发现加了\__block说明符后，i变成了一个结构体指针， \__Block_byref_i_0 *i。该结构体声明如下：

```
struct __Block_byref_i_0 {
  void *__isa;
__Block_byref_i_0 *__forwarding;//指向该结构体自身的指针，
 int __flags;
 int __size;
 int i;
};

```

其中\__Block_byref_i_0 *\__forwarding是指向该结构体自身的指针，
那么

```
^{
	printf("block:%d\n",i);
};
```

转换结果又如何呢？它变成了：

```
static void __main_block_func_0(struct __main_block_impl_0 *__cself) {
  __Block_byref_i_0 *i = __cself->i; // bound by ref
        printf("block:%d\n",(i->__forwarding->i));
}
```

可以看到，这里使用的i是栈上生成的该结构体的实例,这就是为什么在使用\__block说明符后，在执行了i++对i重新赋值之后，Block当中的i值也发生了改变。

不加\__block说明符只是单纯的值拷贝，而加了__block说明符后，是指针拷贝。

***


## Block的存储区域
<p id="block4"></p>

之前我们知道_NSConcreteStackBlock相当于一个objc_class结构体实例，即OC中的类。
其实还有很多与之类似的类。

```
_NSConcreteStackBlock//该类的对象Block保存在栈上
_NSConcreteGlobalBlock//该类的对象Block，与全局变量一样，保存在程序的数据区域（静态存储区域.data).
_NSConcreteMallocBlock//该类的对象Block，保存在malloc函数分配的内存块，即堆内存中。
```

### 内存基本构成
> 内存基本构成  可编程内存在基本上分为这样的几大部分：代码区域、静态存储区、堆区和栈区。
一般全句变量存放在数据区，局部变量存放在栈区，动态变量存放在堆区，函数代码存放在代码区。 
他们的功能不同，对他们使用方式也就不同。  
静态存储区：内存在程序编译的时候就已经分配好，这块内存在程序的整个运行期间都存在。它主要存放静态数据、全局数据和常量。
栈区：在执行函数时，函数内局部变量的存储单元都可以在栈上创建，函数执行结束时这些存储单元自动被释放。栈内存分配运算内置于处理器的指令集中，效率很高，但是分配的内存容量有限。
堆区：亦称动态内存分配。程序在运行的时候用malloc或new申请任意大小的内存，程序员自己负责在适当的时候用free或delete释放内存。动态内存的生存期可以由我们决定，如果我们不释放内存，程序将在最后才释放掉动态内存。


### _NSConcreteStackBlock
就如前面的例子所示，声明一个局部Block变量，生成的Block的结构体的isa成员变量为是\_NSConcreteStackBlock类对象，它是设置在栈上的。

### _NSConcreteGlobalBlock
声明一个全局Block变量，生成的Block的结构体的isa成员变量为\_NSConcreteGlobalBlock类的对象，它是设置在静态存储区的。

### _NSConcreteMallocBlock
我们知道，设置在栈上的Block如果其所属的变量作用域结束，该Block对象就会释放。用\__block修饰的变量，会产生一个\__Block_byref_i_0 *i 变量，该变量也是配置在栈上，如果其作用域结束，该\__block变量也会被释放。所以，提供了将Block和\__block变量从栈上复制到堆上的办法来解决这个问题。

把设置在栈上的Block复制到堆上，这样即使Block语法记述的变量作用域结束，堆上的Block还可以继续存在。
复制到堆上的Block，生成的Block的结构体的isa成员变量为\_NSConcreteMallocBlock类的对象，他设置在堆内存上。

例如有这么一个函数:

```
- (NSArray *)getBlockArray {
    int i = 10;
    return [[NSArray alloc] initWithObjects:
            ^{NSLog(@"blk0:%d",i);},
            ^{NSLog(@"blk1:%d",i);},nil];
}

```

这个函数返回一个NSArray数组，且数组中的元素为Block，这个Block截取了一个局部变量int i ;
然后执行以下代码

```
NSArray* array = [self getBlockArray];
blk block0 = [array objectAtIndex:0];
block0();
```
运行发现，程序崩溃了。这是因为Block定义在栈上，在返回时，栈上的这个Block已经被释放了。

```
- (NSArray *)getBlockArray {
    int i = 10;
    return [[NSArray alloc] initWithObjects:
            [^{NSLog(@"blk0:%d",i);} copy],
            [^{NSLog(@"blk1:%d",i);} copy],nil];
}

```
这样就没有问题了，打印出了：“blk0:10”。

其实在ARC下，将Block作为返回值时，在大多数情况下，编译器会适当地进行判断，自动生成复制到堆上的代码。
但向方法或函数的参数中传递Block时，编译器不能进行判断是否需要复制。例如前面例子中，向NSArray实例的initWithObjects方法传递了Block作为参数。这个时候，就需要我们手动复制。
但以下情况不用手动复制:
Cocoa框架中方法名中包含有usingBlock时
GCD的API，例如：
dispatch_async(dispatch_queue_t queue, ^(void)block);方法，第二个参数就不用手动复制。因为这些函数在内部实现时已经进行了copy操作。
其实不管Block配置在堆上，栈上，还是静态存储区，使用copy方法复制都不会产生任何问题，在不确定时，调用copy即可。但是，将Block从栈上复制到堆上
是一个相当消耗CPU的操作。
下面总结了将配置在不同存储域的Block进行copy方法会产生的结果

* 对_NSConcreteStackBlock类对象的Block进行copy：从栈复制到堆
* 对_NSConcreteGlobalBlock类对象的Block进行copy：什么也不做
* 对_NSConcreteMallocBlock类对象的Block进行copy：引用计数增加（且在ARC下，多次copy也不会有什么问题）

***

## __block变量的存储区域
<p id="block5"></p>


在Block从栈复制到堆时，若其中包含的\__block变量也会受到影响。
任何一个Block从栈复制到堆时，\__block变量也会一并从栈复制到堆，并被该Block持有（引用计数+1）。
如果Block被释放，那么\__block变量也会一并被释放。
这就是为什么\__block变量中struct __Block_byref_i_0结构体变量使用\__forwarding指针的原因，\__block从栈复制到堆，此时可以同时访问栈上的\__block变量和堆上的\__block变量

举个🌰

```
    __block int i = 0;
    void (^blk) () = [^{++ i;} copy];//对堆上的 i进行自增
    ++ i ;//对栈上的i进行自增
    blk();
    NSLog(@"%d",i);//打印结果为2

```
无论是[^{++ i;} copy]（对堆上的 i进行自增），还是 ++ i（对栈上的i进行自增）都可以转换成:

```
++ (i.__forwarding->i);
```
执行copy方法时，Block从栈上复制到了堆上，\__block int i将作为一个结构体对象（\__Block_byref_i_0）复制到堆上。
但是无论不管__block变量在栈上还是堆上，都能通过这个指针正确的访问\__block变量。

\__block说明符变量从栈上复制到堆上的过程如下：
在栈上时,\__forwarding是一个指向自己本身的指针。
复制到堆上后，栈上\__block变量会将成员变量\__forwarding替换为复制目标堆上的\__block变量的结构体实例的地址，
而堆上的\__block变量的\__forwarding还是一个指向自身的指针。
相当于，** 复制后，栈上跟堆上的__block变量的结构体成员变量__forwarding都是一个指向堆上__block变量的结构体的指针。**

*** 

## 捕获变量
<p id="block6"></p>

有这么一段代码

```
blk_t blk;

{
  id array = [[NSMutalbeArray alloc] init];
  blk = [^(id object){
            [array addObject:object];
            NSLog(@"array count = %ld",[array count]);
  } copy];

}

blk([[NSObject alloc] init]);
blk([[NSObject alloc] init]);
blk([[NSObject alloc] init]);
```
运行结果：

```
array count = 1
array count = 2
array count = 3
```

按道理array在其作用域结束时，立即被释放，它对[[NSMutalbeArray alloc] init]生成对象的强引用失效，此时NSMutalbeArray对象也应该被释放才对，
但代码却正常运行，这一结果意味着，array对象在最后Block执行阶段超出了它的作用域而存在。

转换源码后，我们发现blk在生成的结构体中包含一个id \__strong array对象，也就是说blk持有了array对象,
并且在\__main_block_desc_0结构体中增加了成员变量copy和dispose函数，block会在恰当的时候调用copy和dispose函数来进行内存管理。

copy函数和dispose函数调用时机如下

* copy函数：栈上的Block复制到堆时。
* dispose函数：堆上的Block被释放时。

那么什么时候栈上的Block会复制到堆上呢？

* 调用Block的copy实例方法时。
* Block作为函数返回值返回时。
* 将Block赋值给附有__strong修饰符id类型的类，或Block类型成员变量时。
* 在方法名中含有usingBlock的Cocoa框架方法或者GCD的API中传递Block时。（这些函数的内部会对传进去的Block参数调用copy方法）
* 
copy函数和dispose函数中，通过BLOCK_FIELD_IS_OBJECT和BLOCK_FIELD_IS_BYREF参数来区分copy函数和dispose函数的对象类型是对象还是\__block变量。

那么在刚才的代码中，如果不调用Block的copy方法又会如何呢？

```
blk_t blk;

{
  id array = [[NSMutalbeArray alloc] init];
  blk = ^(id object){
            [array addObject:object];
            NSLog(@"array count = %ld",[array count]);
  };

}

blk([[NSObject alloc] init]);
blk([[NSObject alloc] init]);
blk([[NSObject alloc] init]);

```

执行该代码，程序会崩溃。因为上面代码并不调用Block的copy函数，即使截取了对象，它也会随变量作用域结束而被释放。

前面我们使用的是用\__strong修饰符的id array对象（默认为\__strong），那么如果使用\__weak修饰符的id array又会如何呢？

```
blk_t blk;

{
   id array = [[NSMutalbeArray alloc] init];
   __weak id array2 = array;
  blk = [^(id object){
            [array2 addObject:object];
            NSLog(@"array2 count = %ld",[array2 count]);
  } copy];

}

blk([[NSObject alloc] init]);
blk([[NSObject alloc] init]);
blk([[NSObject alloc] init]);
```
运行结果：

```
array2 count = 0
array2 count = 0
array2 count = 0
```

这是因为array2并不持有id array对象，在id array对象在作用域结束时，就被释放。此时，被__weak修饰符修饰的array2将被赋值为nil。但代码可以正常执行，并不会崩溃。（__weak修饰符的好处 ：））

***

## Block循环引用
<p id="block7"></p>

如果Block在使用附有__strong修饰符的对象类型变量，那么当Block从栈复制到堆时，这个对象会被Block持有，容易引起循环引用。
举个🌰

```
typedef void (^blk_t)(void);

@interface MyObject : NSObject
{
	blk_t _blk;
}
@end

@implementation MyObject

- (id)init {
	self  = [super init];
	if(self){
		_blk = ^{
			NSLog(@"%@",self);
		};
	}
	return self;
}

@end

int main() {
	MyObject* object = [[MyObject alloc] init];
	return 0;
}

```
MyObject类对象持有blk_t类型的成员变量。而blk_t类型的成员变量在初始化时，执行的Block语法附有__strong修饰符的id类型变量self。并且
由于Block语法赋值在了成员变量_blk中，因此这个Block将由栈复制到堆，并持有self。现在self持有_blk，而_blk又持有self，因此造成了循环引用。
为了避免循环引用，可声明附有__weak修饰符的变量，并将self赋值使用。

```
- (id)init {
	self = [super init];
	if(self) {
		__weak typeof(self) weakSelf = self;
		 _blk = ^{
			NSLog(@"%@",weakSelf);
		};
	}
	return self;
}

```

现在，self持有_blk，但是_blk并不持有self了，就避免了循环引用。

在AFNetworking源码中，对于这种情况，他的写法是

```
- (id)init {
	self = [super init];
	if(self) {
		__weak typeof(self) weakSelf = self;
		 _blk = ^{
		 	__strong typeof(weakSelf) strongSelf = weakSelf;
			NSLog(@"%@",strongSelf);
		};
	}
	return self;
}

```

第一个\__weak说明符是为了防止循环引用，后面的__strong说明符是为了防止在当前作用域还没结束的时候，weakSelf就被释放了。

***

## 参考
* 《Objective-C高级编程》


