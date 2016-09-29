---
layout:     post
title:      "温故：数据结构(一) -- 线性表"
subtitle:   "数据结构基础巩固，线性表"
date:       2014-09-12
author:     "waynezxcv"
header-img: "img/post.png"
tags:
    - 数据结构
---

### 目录

1. [线性表的顺序存储结构](#section1)
2. [线性表的链式存储结构](#section2)



### 线性表的顺序存储结构

线性表的顺序存储结构，指的是用一段地址连续的存储单元一次存储线性表的数据元素。


#### 线性表的顺序存储结构的代码描述

```

//顺序存储的线性表

 #define MAX_SIZE 20
 #define OK 1
 #define ERROR 0
 #define TURE 1
 #define FALSE 0

typedef int ElementType;//元素类型
typedef int Status;//返回状态

typedef struct SqListTag {
    ElementType data[MAX_SIZE];
    int length;
} SqList;


/**
 *  获取元素
 *
 */
Status getElement(SqList L,int i ,ElementType * e) {
    if (i > L.length || i < 1 || L.length == 0 ) {
        return ERROR;
    }
    *e = L.data[ i -1];
    return OK;
}


/**
 *  插入操作
 *
 */
Status insertElement(SqList *L,int i,ElementType e) {
    //线性表已满
    if (L->length == MAX_SIZE) {
        return ERROR;
    }
    //当i不在线性表的元素范围内
    if (i < 1 || i > L->length + 1) {
        return ERROR;
    }
    if (i <= L->length) {
        for (int j = L->length - 1; j > i -1; j --) {
            L->data[j + 1] = L->data[j];//将插入位置之后的元素依次后移
        }
    }
    L->data[i -1] = e;//插入元素
    L->length ++;//长度增加
    return OK;
}

/**
 *  删除操作
 *
 */
Status deleteElement(SqList *L,int i) {
    if (L ->length == 0) {
        return ERROR;
    }
    
    if (i < 1 || i > L->length) {
        return ERROR;
    }

    
    for (int j = i; j < L -> length; j ++) {
        L ->data[j - 1] = L -> data[j];//将删除之后的元素依次前移
    }
    
    L ->length --;//长度减少
    return OK;
}


```

#### 线性表顺序存储的优缺点

优点：
* 无须为表中元素之间的逻辑关系而增加额外的存储空间。
* 可以快速地存取表中任意位置的元素。（时间复杂度O(1)）

缺点
* 插入和删除操作需要移动大量元素（时间复杂度O(n)）
* 当线性表长度变化比较大时，难以确定存储空间的容量
* 造成存储空间的“碎片”


***


### 线性表的链式存储结构

线性表链式结构除了要存储数据元素信息外，还要存储它后继元素的存储地址。

* 头指针，我们把链表中第一个结点的存储位置叫做头指针。线性链表的最后一个结点指针为NULL。头结点是链表的必要元素。
* 头结点，有时候，我们为了方便地对链表进行操作，会在单链表的第一个结点前附设一个头结点。头结点数据域不存储任何信息，头结点不是链表的必要元素。

#### 线性表的链式存储结构的代码描述

```

//链式存储的线性表

 #define OK 1
 #define ERROR 0
 #define TURE 1
 #define FALSE 0

typedef int ElementType;//元素类型
typedef int Status;//返回状态

typedef struct NodeTag Node;

struct NodeTag {
    ElementType data;
    Node * next;
};

typedef Node* LinkList;//定义链表


/**
 *  头插法，创建一个元素个数为i的单链表
 *
 */
void createListHead(LinkList *L, int i) {
    //首先创建一个空链表
    *L = malloc(sizeof(Node));
    (*L) -> next = NULL;
    LinkList p;
    for (int j = 0; j < i ; j ++) {
        p = malloc(sizeof(Node));//生成新节点
        p ->data = rand();//对数据存储区随机赋值
        p -> next = (*L) -> next;
        (*L) -> next = p;//插入到表头
    }
}


/**
 *  单链表的整表删除
 *
 */

Status deleteList (LinkList *L) {
    
    LinkList p ,q;
    
    p = (*L) -> next;//p指向第一个节点
    
    while (p) {
        q = p -> next;
        free(p);
        p = q;
    }
    
    (*L)->next = NULL;//头结点指针域为NULL
    return OK;
}


/**
 *  查询元素
 */
Status getElement(LinkList L,int i ,ElementType * e) {
    
    LinkList p;
    p = L->next;//声明一个指针p，指向L的第一个元素
    
    int j = 1;
    while (p && j < i) {//由于链表元素个数未知，不方便用for循环，用while
        p = p ->next;
        ++j;
    }
    if (!p || j > i) {
        return ERROR;
    }
    *e = p ->data;
    return OK;
}


/**
 *  插入元素
 */
Status insertElement(LinkList* L,int i,ElementType e) {
    
    LinkList p = *L;
    
    int j = 1;
    
    
    while (p && j < i) {//寻找第i - 1个元素
        p = p ->next;
        ++j;
    }
    
    if (!p || j > i) {
        return ERROR;
    }
    //插入一个新节点
    LinkList s = malloc(sizeof(LinkList));
    s -> data = e;
    s -> next = p -> next;
    p ->next = s;
    return OK;
}

/**
 *  删除元素
 */

Status deleteElement(LinkList* L,int i ,ElementType *e) {
    LinkList p = *L;
    int j = 1;
    
    
    while (p->next && j < i) {
        p = p -> next;
        j ++;
    }
    
    if (!(p->next) || j > i) {
        return ERROR;
    }
    
    LinkList q;
    q = p -> next;
    p -> next = q ->next;
    *e = q->data;
    free(q);
    return OK;
}


```


#### 静态链表

用数组描述的链表。该数组的元素都由两个数据域组成，data和cur。data用来存放数据元素。cur用来存放该元素后继数组中的下标，相当于单链表中的next。

优点：

* 在插入和删除操作时，无需后移和前移元素，只需要修改游标。

缺点：

* 没有解决连续存储分配带来的表长度难以确定的问题

#### 循环链表

将单链表中终端节点的指针端由空指针改为指向头结点。循环列表跟单链表的主要差异在于循环的判断条件上，**原来判断p->next是否为空,现在则是p->next不等于头结点，则循环结束。**


#### 双向链表

为了克服单链表单向性这一缺点，双向链表在单链表的每个节点中，再设置一个指向其前驱结点的指针域。

```

//双向链式存储的线性表，双向链表
typedef int ElementType;//元素类型
typedef struct DulNodeTag DulNode;
struct DulNodeTag {
    ElementType data;
    DulNode *prior;
    DulNode *next;
};
typedef DulNode * DulLinkList;


```

双向链表中的某一节点p的后继节点的前驱节点是它自己。

```
        p->next->prior = p = p->prior->next;

```

```
/**
 *  双向链表的插入操作
 */

Status insertElement(DulLinkList *L,int i , ElementType e) {
    DulLinkList p = *L;
    //寻找第i - 1个节点
    int j = 1;
    while (p && j < i) {//寻找第i - 1个元素
        p = p ->next;
        ++j;
    }
    if (!p || j > i) {
        return ERROR;
    }
    
    DulLinkList s = malloc(sizeof(DulNode));
    s->data = e;
    s->prior = p;
    s->next = p ->next;
    p->next->prior = s;
    p -> next = s;
    return OK;
}

/**
 *  双向链表的删除操作
 */

Status deleteElement(DulLinkList *L,int i , ElementType *e) {
    
    DulLinkList p = *L;
    
    //寻找第i - 1个节点
    int j = 1;
    while (p && j < i) {//寻找第i - 1个元素
        p = p ->next;
        ++j;
    }
    if (!p || j > i) {
        return ERROR;
    }
    DulLinkList q = p ->next;
    *e = q->data;
    p->next = q -> next;
    q->next->prior = p;
    free(q);
    return OK;
}


```

既然单向链表有循环结构，那么双向链表也有循环结构，双向循环链表。
