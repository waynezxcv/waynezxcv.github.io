---
layout:     post
title:      "彻底理解C指针-- C指针与数组"
subtitle:   "深入理解C语言中的指针。"
date:       2014-09-02
author:     "waynezxcv"
header-img: "img/post.png"
tags:
    - C语言
    - 指针
---


### 目录

1. [指针和数组](#section1)
2. [为什么存在奇怪的指针运算符](#section2)
3. [将数组作为函数的参数进行传递](#section3)
4. [声明函数形参的方法](#section4)
5. [C为什么不做数组下标越界检查](#section5)
6. [指针和数组的常用方法](#section6)
7. [惯用语法](#section7)


### 指针和数组


在很多C语言的入门书籍是这么说明的:
>在C中，如果数组名后不加[]，单独地致谢数组名，那么此名称就表示“指向数组初始元素的指针”。

然而，这句话其实“不那么正确”。其实无论加不加下标计算符[],它都是表示指向数组初始元素的指针。[]下标计算符只不过是个语法糖。（例a[i]只不过是*(a + i)的语法糖）
证明如下：

```
int main(int argc, const char * argv[]) {

    int arrary[5];
    
    for (int i = 0; i < 5; i ++) {
        arrary[i] = i;
    }
    
    int *p = arrary;
    
    for (int i = 0; i < 5; i ++) {
        printf("%d\n",*(p + i));
    }
    return 0;
}

```

这里的

```
for (int i = 0; i < 5; i ++) {
	printf("%d\n",*(p + i));
}

```
和

```
for (int i = 0; i < 5; i ++) {
        printf("%d\n",p[i]);
    }
    
```
效果是一样的。也就是说``p[i]``和``*(p + i)``是同样的意思。

这样理解，更助于我们理解指针：

** 表达式中，数组可以解读成“指向它初始化元素的指针”。但是这和在后面加不加[]没有关系。**

** p[i]是\*(p + i)的简便写法。**

** 下标运算符[]原本只有这种用法，它和数组无关。**

需要强调的是，认为[]和数组无关，这里的[]指的是表达式中出现的下标运算符[],声明中的[]还是数组的意思。
证明如下：
如果将a + b改写成b + a,表达式的意义没有发生改变，所以你可以将*(p + i)写成\*(i + p)。**因此，因为p[i]是 \*(p + i)的简便写法，你也可以把p[i]写成i[p]，甚至写成i[array]**.

```
    for (int i = 0; i < 5; i ++) {
        printf("%d\n",i[arrary]);
    }
    
```
跟上面的打印结果是一样的！

当然，**这样写没什么意义， 太另类了，不能给我们带来任何好处。**这里只是为了说明**数组表达式中的[]下标计算符跟数组没有任何关系，只不过是指针偏移量取值计算的简写，这也就不难理解为什么C语言的数组下标是从0开始的。**

***

### 为什么存在奇怪的指针运算符

* 继承B语言中的特性。C引入了**"指针加1，指针就前进它所指向类型的长度"**这个规则。
* 处理数组时如果写成这样

```
    for (int i = 0; i < 5; i ++) {
        printf("%d\n",arrary[i]);

    }

```
而array[i]等价于*(array + i)，每次循环都要进行加法运算（先要对i做加法用于循环，然后又要对array做加法计算指针的偏移量用于取值），效率很低。因此，如果对指针进行运算，就只要进行一次加法运算了，效率更高。

```
    for (int *i = &arrary[0]; i != &arrary[5]; i ++) {
        printf("%p\n",i);
        printf("%d\n",*i);
    }
    
```

但是，现在的条件下，编译器做了各种优化，无论你怎么写，编译器基本上生成的都是相同的机器码。效率上不过有什么差异。

**C指针运算功能，源自于早起C自身么有优化手段。**
在现在的情况下，**不建议使用指针运算(指直接使用指针做加减法的运算)，而使用下标访问**，因为代码应该更加让人理解，而不是为了这种近乎于臆想的优化，不是吗？

***

### 将数组作为函数的参数进行传递

这里，我们来实现一个函数,其作用是从英文的文本文件中，将单词一个一个取出来。调用方式，我们模仿fgets()函数，实现如下:


```
int get_word(char *buf,int buf_size,FILE *filePath) {
    
    int len;
    int ch;
    
    /*跳过空白字符,读取赋值给ch*/
    while ((ch = getc(filePath)) != EOF && !isalnum(ch));
    if (ch == EOF) {
        return EOF;
    }
    
    /* 此时，ch中保存了单词的初始字符 */
    len = 0;
    do {
        buf[len] = ch;
        len ++;
        
        if (len > buf_size) {
            fprintf(stderr, "word is too long\n");
            exit(1);
        }
    }while ((ch = getc(filePath)) != EOF && !isalnum(ch));
    
    /* 此时，读取结束 */
    
    buf[len] = '\0';//字符串结束符
    
    return len;
}

int main(int argc, const char * argv[]) {
    char buf[256];
    
    while (get_word(buf, 256, stdin) != EOF) {
        printf("%s\n",buf);
    }
    return 0;
}

```

** 在C中，不能讲数组作为函数参数进行传递，但是你可以传递指向数组初始元素的指针的方法，来实现将数组作为参数传递的目的.**

** 无论如何都要将数组进行值传递时，建议将数组整体整理成结构体成员**

***

### 声明函数形参的方法

**只有在声明函数形参时，数组的声明才被理解成指针**

下面声明的形参，都具有相同的意义。

```
int func(int *a);//写法1
int func(int a[]);//写法2
int func(int a[10]);//写法3

```

**写法2和写法3是写法1的语法糖。且在C中，只有声明函数形参时，int a[]和int *a具有相同的意义，数组的声明才被理解成指针**

***


### C为什么不做数组下标越界检查

因为在C中，当数组出现在表达式中时，它会立即被解读成指针。此外，使用其他的指针变量也可以指向数组的任意元素，并且这个指针可以随意进行加减运算。引用数组元素时，虽然你可以写成a[i],但是它只不过是*(a + i)的语法糖。还有，当你向一个函数传递数组时，实际传递的是一个指向数组初始元素的指针。如果这个函数还存在其他的代码文件中，**那么编译器是不可能追踪到数组的.**因此，除了在某些解释型编程语言中，目前几乎没有编译器可以为我们做数组的越界检查。

***


###  指针与数组的常用方法


#### 以函数返回值之外的方式来返回值

在C中，可以通过函数返回值。但是，函数只能返回一个值。很多时候，我们需要通过函数返回多个值。这个时候，我们将指针作为参数传递给函数，之后再函数内部对指针指向的对象填充内容，就可以从函数返回多个值。

一个简单的🌰:

```

void func (int *a,int *b) {
    *a = 5;
    *b = 6;
}

int main(int argc, const char * argv[]) {
    int a;
    int b;
    func(&a, &b);
    
    printf("a = %d...b = %d\n",a,b);
    return 0;
}

```

#### 将数组作为函数的参数传递
在C语言中，数组不能作为参数进行传递。但是可以通过传递指向数组初始元素的指针，使得在函数内部操作数组成为可能。而在函数实现一侧，通过

```
array[i];
```
这种方式，就可以引用数组的内容。因为在本质上array[i]只不过是*(array + i)的语法糖。

一个简单的🌰:

```

void func (int *array,int size) {
    int i ;
    for (i = 0 ; i < size; i ++) {
        printf("array[%d] .. %d\n",i ,array[i]);
    }
}

int main (int argc,const char *argv[]){
    int array[] = {1,2,3,4,5,6};
    func(array, sizeof(array)/sizeof(int));
}


```

func()还以参数size来接收数组array的元素个数。这是因为array只是一个指针，所以func()并不知道调用方数组的元素个数。

#### 可变长数组

一般情况下，C语言在编译时必须知道数组的元素个数。但是也可以使用malloc()在运行时再为数组申请必要的内存区域。

一个简单的🌰:

```

int main (int argc,const char *argv[]) {
    
    char buff [256];
    int size;
    int *variable_array;
    int i ;
    
    printf("Input array size >");
    fgets(buff, 256, stdin);
    
    sscanf(buff, "%d",&size);
    
    variable_array = malloc(sizeof(int) * size);
    
    for (i = 0; i < size; i ++ ) {
        variable_array[i] = i;
    }
    
    for (i = 0; i < size; i ++) {
        printf("variable_array[%d]...%d\n",i,variable_array[i]);
    }
    return 0;
    
}


```
在使用malloc()实现可变长数组的时候，程序员必须自己来管理数组的元素个数。这和将数组作为参数进行传递时，被调用方无法知道数组的长度理由一样。---malloc()得到的不是数组，而是指针。


***

### 惯用语法

#### 自引用型结构体

为了创建链表和树，我们会声明包含指向相同类型的指针的结构体。

>使用结构体还是结构体指针？
结构体指针的缺点：
1.速度更快，兼容性（以前的C不支持将结构体作为参数传递）
2.只需要传递一个地址
>
结构体指针的缺点：
1.缺少对数据的保护，可能在函数中改变了原始数据的值。不过，可以通过const限定符来解决这个问题。
>
直接使用结构体的优点：
1.传递的参数是原始数据的副本，比较安全。
2.编码风格更清晰。



```
typedef struct Hoge_tag {
    int a;
    int b;
    struct Hoge_tag *next;
}Hoge;


```
在这种情况下，在声明成员next的时候，结构体Hoge的typedef还没结束，所以next只能写成struct Hoge_tag *next。

或者下面这样

```

typedef struct Hoge_tag Hoge;

struct Hoge_tag {
    int a;
    int b;
    Hoge *next;
};

```

#### 结构体的相互引用

在相互引用的结构体中，应该像下面这样只是将tag提前声明，下面的代码中Man持有执行“wife”的指针，Woman持有指向"husband"的指针

```

typedef struct man_tag Man;

typedef struct woman_tag {
    Man *husband;
} Woman;

struct man_tag {
    Woman* wife;
};

```




