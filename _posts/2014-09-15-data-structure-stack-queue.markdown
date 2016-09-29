---
layout:     post
title:      "温故：数据结构(二) -- 栈和队列"
subtitle:   "数据结构基础巩固"
date:       2014-09-15
author:     "waynezxcv"
header-img: "img/post.png"
tags:
    - 数据结构
---

### 目录

1. [栈](#section1)
2. [队列](#section2)


### 栈

栈是限定只能在表尾（栈顶）进行插入和删除操作的线性表。允许插入和删除的一端称为栈顶，另一端称为栈底，不含任何数据元素的栈称为空栈。栈又称后进先出的线性表，LIFO结构。

#### 栈的抽象数据类型

```

ADT 栈
Data 同线性表，元素具有相同的类型，相邻元素具有前驱和后继关系。
Operation

InitStack(*S):初始化操作，创建一个空栈s。
DestroyStack（*S）:若栈存在，则销毁它。
ClearStack（*S）：将栈清空。
StackEmpty（S）：若栈为空返回ture，否则返回false
GetTop（S，*e）:若栈存在且非空，用e返回栈顶的元素。
Push（* S，e）：若栈存在，插入元素e，并成为栈顶元素
Pop（* S，*e）：若栈存在且非空，删除栈顶元素，并用e返回它的值。
StackLength（S）：返回栈的元素个数。

endADT

```
#### 栈的顺序存储结构
栈的顺序存储结构是线性表顺序存储结构的简化，我们称它顺序栈。

```

#define OK 1
#define ERROR 0
#define TURE 1
#define FALSE 0
typedef int Status;//返回状态

#define MAX_STACK_SIZE 10
typedef int ElementType;

/**
 *  顺序栈的定义
 */
typedef struct {
    int data[MAX_STACK_SIZE];
    int top;//用于栈顶的指针
}SqStack;

/**
 *  入栈
 *
 */
Status StackPush(SqStack *s,ElementType e) {
    
    if (s->top == MAX_STACK_SIZE - 1) {
        return ERROR;
    }
    
    s->data[s->top] = e;
    s->top ++;
    return OK;
}

/**
 *  出栈
 *
 */
Status StackPop(SqStack *s,ElementType *e) {
    if (s ->top == -1) {
        return  ERROR;
    }
    *e = s->data[s->top];
    s->top --;
    return OK;
}

```

#### 两栈共享空间

用一个数组来存储两个类型相同的栈。让一个栈的栈底为数组始端，下标为0处。让另一个栈的栈底为数组尾端，下标为n-1处。当top1 + 1 == top2时，表示栈满。

```

/**
 *  两栈共享存储空间
 */

typedef struct {
    ElementType data[MAX_STACK_SIZE];
    int top1;
    int top2;
}SqDoubleStack;

/**
 *  共享存储空间的入栈
 */

Status DoubleStackPush(SqDoubleStack *S,ElementType e,int stackNumber) {
    
    if (S->top1 + 1 == S->top2) {
        return ERROR;//满栈
    }
    
    if (stackNumber == 1) {
        
        S->data[S->top1] = e;
        S->top1 ++;
    }
    
    else if (stackNumber == 2) {
        
        S->data[S->top2 - 1] = e;
        S ->top2 -- ;
        
    }
    return OK;
}


/**
 *  共享存储空间的出栈
 */

Status DoubleStackPop (SqDoubleStack *S,ElementType *e,int stackNumber) {
    
    if (stackNumber == 1) {
        if (S -> top1 == -1) {
            return ERROR;
        }
        
        *e = S->data[S->top1];
        
        S->top1 --;
        
    }
    
    else if (stackNumber == 2) {
        if (S->top2 == MAX_STACK_SIZE) {
            return ERROR;
        }
        
        *e = S->data[S->top2];
        
        S->top2 ++;
        
    }
    return OK;
}


```

使用共享空间栈的数据结构，一般是当两个栈的空间需求有相反时，也就是一个增长一个缩短的情况，否则两个栈同时增长，很快就会满栈溢出。

#### 栈的链式存储结构及实现

栈的链式存储结构，简称链栈。由于单链表有头指针，而栈顶指针也是必须的，所以比较好的办法是把栈顶放在单链表的头部。另外，由于已经有了栈顶在头部了，单链表中比较常用的头结点也就失去了意义，通常对链栈来说，不需要头结点。对链栈来说，基本不存在栈满的情况，除非内存已经没有可以使用的空间了。对于空栈来说，链表原定义是头指针指向空，那么链栈的空其实是top=NULL的时候。

链栈的实现🌰

```

typedef struct StackNodeTag stackNode;

typedef struct StackNodeTag {
    ElementType data;
    stackNode* next;
}* LinkStackPtr;

typedef struct LinkStackPtr {
    LinkStackPtr top;
    int count;
}LinkStack;

/**
 *  入栈操作
 */

Status Push(LinkStack *S,ElementType e) {
    LinkStackPtr s = malloc(sizeof(stackNode));
    s->data = e;
    s->next = S-> top;
    S->top = s;
    S->count ++;
    return OK;
}

/**
 *  出栈操作
 *
 */
Status Pop(LinkStack *S,ElementType *e) {
    
    if (StackEmpty(* S)) {
        return ERROR;
    }
    LinkStackPtr p = S->top;

    *e = p -> data;
    S->top = p->next;
    free(p);
    S->count--;
    return OK;
}

```
栈的插入和删除时间复杂度都是O(1)。
**如果栈在使用过程中元素变化不可预料，有时很小，有时又非常大，那么最好用链栈。反之，如果它的变化在可控范围内，建议使用顺序栈。**

***


#### 栈的应用

##### 实现递归
我们把一个直接调用自己或通过一些列的调用语句间接调用自己的函数，称作为递归函数。但是，写递归函数最怕的就是想入永不结束的无穷递归当中，所以，每个递归定义**必须至少有一个条件，满足时递归不再进行，即不再引用自身而是返回值退出。**
递归过程退回的顺序是它前进顺序的逆序，因此编译器使用栈来实现递归。

*** 

### 队列

队列是一种只允许在一端进行插入操作，而在另一端进行删除操作的线性表。队列是一种先进先出FIFO的结构。
允许插入的一端称为队尾，允许删除的一端称为队头。

#### 队列的链式存储结构

```
typedef struct QNodeTag QNode;
typedef QNode* QnodePtr;

struct QNodeTag {
    ElementType data;
    QnodePtr next;
};

typedef struct LinkQueue {
    
    QnodePtr front,rear;//队头和对位指针。
    
}LinkQueue;


/**
 *  入队操作
 */

Status enQueue(LinkQueue *Q,ElementType e) {
    
    QnodePtr p = malloc(sizeof(QNode));//创建一个节点
    p->data = e;//给节点数据域赋值
    p->next = NULL;//给节点指针域赋值
    
    Q->rear->next = p;//把Q的队尾的next指向p,即插入一个节点
    Q->rear = p;//将Q的队尾指向p
    
    return OK;
}

/**
 *  出队操作
 */

Status deQueue(LinkQueue *Q,ElementType *e) {
    if (Q->front == Q ->rear) {
        return ERROR;//队列为空
    }
    
    QnodePtr p = Q->front->next;//获取队头节点
    *e = p->data;//获取队头数据并通过e返回值
    Q->front->next = p->next;//将队头指向p的next
    if (Q->rear == p) {
        Q->rear = Q->front;//如果删除后队头就是队尾，则指向头结点
    }
    free(p);//释放p
    return OK;
}


```

***