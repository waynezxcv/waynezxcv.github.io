---
layout:     post
title:      "温故：数据结构(三) -- 树"
subtitle:   "数据结构基础巩固"
date:       2014-09-25
author:     "waynezxcv"
header-img: "img/post.png"
tags:
    - 数据结构
---

### 目录

1. [树的定义及抽象数据类型]()
2. [树的存储结构]()
3. [二叉树]()
4. [树、森林与二叉树的转化]()
6. [二叉排序树]()
7. [平衡二叉树AVL树]()
8. [多路查找树B树]()

### 树的定义及抽象数据类型



### 树的存储结构

#### 双亲表示法

```

/**
 *  树的双亲表示法
 */
//节点的结构
typedef struct PTNode {
    ElementType data;
    int parent;
} PTNode;

//树的结构

typedef struct PTree {
    PTNode nodes[MAX_TREE_SIZE];
    int r;//根节点的位置
    int n;//结点数
}PTree;


```

另外还可以在节点中添加长子域和右兄弟域。存储结构设计师一个非常灵活的过程。
可以根据具体需求去设计。
存储结构是否合理，取决于运算是否合适、方便，时间复杂度好不好等。

#### 孩子表示法
把每个结点的child结点排列起来，用单链表作为存储结构。则n个结点有n个孩子链表，如果该结点为叶子结点，则此单链表为空。然后n个头指针又组成一个线性表，采用顺序存储结构，存放入一个一维数组当中。

```

/**
 *  孩子表示法
 */


//孩子节点
struct CTNode{
    int child;
    struct CTNode *next;
} ;
typedef struct CTNode *childPtr;

//表头结构
typedef struct {
    ElementType data;
    childPtr firstChild;
} CTBox;


//树的结构

typedef struct  {
    CTBox data[MAX_TREE_SIZE];
    int r;//根节点位置
    int n;//结点数
    
} CTree;

```

也可以在表头结构中加入parent位置，用于查找双亲。称为双亲孩子表示法。

#### 孩子-兄弟表示法

```
/**
 *  孩子兄弟表示法
 */
typedef struct CSNode {
    ElementType data;
    struct CSNode *firstChild,*rightSib;//长子和右兄弟
}CSNode, *CSTree;

```

此方法给查找某个结点的某个孩子带来方便，只需要通过通过firstchild找到此结点的长子，然后通过长子结点的rightsib找到它的右兄弟，接着一直找下去，直到找到具体的孩子。

这个方法最大的好处就是把一颗复杂的树变成了一颗二叉树。


### 二叉树Binary tree

#### 二叉树的定义、特点

二叉树是n(n>=0)个结点的有限集合，该集合或者为为空集（空二叉树），或者由一个根结点和两棵互不相交的、分别称为根的左子树和右子树的二叉树组成。

* 二叉树每个结点最多有两棵子树。
* 左子树和右子树是有顺序的。
* 即使某结点只有一棵子树，也是区分左子树和右子树的。

##### 特殊的二叉树

* 斜树，所有的结点都只有左子树的称为左斜树，所有的结点都只有右子树的称为右斜树。
* 满二叉树，所有的结点都有左子树和右子树，所有的叶子都在同一层上。
* 完全二叉树，除最后一层外，每一层的结点数都达到最大值，最后以后只缺少右边的若干结点。

满二叉树一定是完全二叉树，完全二叉树不一定是满二叉树。


#### 二叉树的存储结构

一般来说，完全二叉树使用顺序存储结构，非完全二叉树使用链式存储结构。

```

/**
 *  二叉树的链式存储结构
 */

typedef struct BiTNode {
    
    ElementType data;//数据域
    struct BiTNode *lChild,*rChild;//左右孩子指针。
    
}BiTNode, *BiTree;



```

#### 二叉树的遍历

##### 前序遍历

遍历顺序，根节点-》左子树-》右子树。

```
//二叉树的前序遍历递归算法
void PreOrderTraverse (BiTree T) {
    if (T == NULL) {
        return;
    }
    
    printf("%d",T->data);//读取数据，或者其他对结点的操作
    
    PreOrderTraverse(T->lChild);
    PreOrderTraverse(T->rChild);
}


```


##### 中序遍历

遍历顺序 ,左子树 -> 根节点 -> 右子树

```
//二叉树的中序遍历递归算法
void  InOrderTraverse (BiTree T) {
    if (T == NULL) {
        return;
    }
    
    InOrderTraverse(T -> lChild);
    printf("%d",T->data);//读取数据，或者其他对结点的操作
    InOrderTraverse(T -> rChild);
}


```

##### 后续遍历

遍历顺序，左子树 -》 右子树 -》 根节点

```

//二叉树的后序遍历递归算法

void PostOrderTraverse (BiTree T) {
    if (T == NULL) {
        return;
    }
    
    PostOrderTraverse(T -> lChild);
    PostOrderTraverse(T -> rChild);
    printf("%d",T->data);//读取数据，或者其他对结点的操作

}



```


##### 线索二叉树

我们可以利用二叉树中的空指针域，存放指向结点的某种遍历次序下的前驱和后继结点的地址。我们把这种前驱和后继的指针成为线索，加上线索的二叉链表称为线索链表，相应的二叉树就称为线索二叉树。对二叉树以某种次序遍历使其变成线索二叉树的过程称为线索化。


```
/**
 *  线索二叉树的结构实现
 */

typedef enum {
    Link = 0,//表示指向左右孩子的指针
    Thread = 1//表示指向前驱和后继的指针
}PointerTag;




typedef struct BiThrNode {
    
    ElementType data;
    PointerTag lTag,rTag;
    struct BiThrNode *lChild,*rChild;
    
}BiThrNode, *BiThrTree;


```

### 树、森林与二叉树的转化

#### 树转化为二叉树

1. 加线，在所有兄弟结点之间加一条连线。
2. 去线，对树中的每个结点，只保留它与第一个孩子结点的连线。
3. 层次调整。将整棵树顺时针旋转一定角度，注意，第一个孩子是二叉树结点的左子树，兄弟结点转化过来的子树是结点的右子树。


#### 森林转化为二叉树

森林是若干棵树组成的。所以可以理解为，森林中的每一棵树都是兄弟。

1. 把每个书转换成为二叉树
2. 第一棵二叉树不动，依次把后一棵二叉树的根节点作为前一棵二叉树的根节点的右子树，用线连接起来。

### 二叉排序树

二叉排序树又称二叉查找树。它具有以下性质：

* 若它的左子树不为空，则左子树上的所有结点的值都小于它根节点的值。
* 若它的右子树不为空，则右子树上的所有结点的值都大于它根节点的值。
* 它的左右子树也分别为二叉排序树。


构造一棵二叉排序树的目的，不是为了排序，而是为了提高查找和插入、删除关键字的速度。

#### 二叉排序树查找操作

```
/**
 *  二叉树的链式存储结构
 */

typedef struct BiTNode {
    
    ElementType data;//数据域
    struct BiTNode *lChild,*rChild;//左右孩子指针。
    
}BiTNode, *BiTree;


//递归查找二叉排序树T中是否存在key
//若查找成功，则指针p指向该数据元素结点，并返回TRUE
//否则p指向查找路径中访问的最后一个结点，并返回FALSE
int SearchBST(BiTree T,int key,BiTree f,BiTree *p) {
    if (!T) {
        *p = f;
        return 0;
    }
    else if (key == T->data) {//查找成功
        *p = T;
        return 1;
    }
    else if(key < T -> data) {
        SearchBST(T -> lChild, key, T, p);//继续查找左子树
    }
    else {
        SearchBST(T -> rChild, key, T, p);//继续查找右子树
    }
}

```

#### 二叉排序树的插入操作

```

//二叉排序树的插入操作

int InsertBST(BiTree *T,int key) {
    BiTree p,s;
    
    if (!SearchBST(*T, key, NULL, &p)) {
        
        s = malloc(sizeof(BiTree));
        s->data = key;
        
        s->lChild = s -> rChild = NULL;
        
        if (!p) {
            *T = s;
        }
        else if (key < p -> data) {
            p->lChild = s;
        }
        else {
            p -> rChild = s;
        }
        return 1;
    }
    else {
        return 0;
    }
}



```

#### 二叉排序树的删除操作

删除结点的三种情况：

* 叶子结点，直接删除
* 仅有左或右子树的结点，删除该结点，并将其的子树代替它原来的位置。
* 左右子树都有结点。找到它的直接前驱或者后继s，用s来代替它。


```

//二叉排序树的删除操作

int Delete(BiTree *p) {
    
    BiTree q,s;
    //若左子树为空
    if ((*p) -> lChild == NULL) {
        q = *p;
        *p = (*p) -> rChild;
        free(q);
    }
    //若右子树为空
    else if ((*p) -> rChild == NULL) {
        q = *p;
        *p = (*p) -> lChild;
        free(q);
    }
    //左右子树均不为空
    else {
        
        q = *p;
        
        s = (*p) -> lChild;
        
        while (s -> rChild) {
            q = s;
            s = s -> rChild;
        }
        
        (*p) ->data = s -> data;
        
        if (q != *p) {
            q -> rChild = s ->lChild;
        }
        
        else {
            q -> lChild = s-> lChild;
        }
        
        free(s);
        
        
    }
    return 1;
    
}


int DeleteBST(BiTree *T,int key) {
    if (! *T) {
        return 0;
    }
    
    else {
        if (key == (*T) -> data) {
           return Delete(T);
        }
        else if (key < (*T) ->data ) {
            return DeleteBST( &(*T) -> lChild, key);
        }
        else {
            return DeleteBST(&(*T) -> rChild, key);
        }
    }
}


```

### 平衡二叉树（AVL树）

我们希望二叉排序树比较平衡，即其深度与完全二叉树相同，那么查找的时间复杂度就是O(logn)，接近于二分查找。不平衡的最差情况，就是斜树，查找的时间复杂度为O（n），这等同于顺序查找。

因此，我们希望二叉排序树能够平衡。

平衡二叉树是一种二叉排序树，其每一个结点的左子树和右子树的高度差最多为1.

#### 平衡二叉树的实现原理

在构建二叉树的过程中，每当插入一个结点时，先检查是否因插入而破坏了树的平衡性，若是，则找出最小不平衡子树。在保持二叉排序树的特性的前提下，调整最小不平衡子树中各结点之间的连接关系，进行相应的旋转，使之成为平衡树。

平衡因子：我们将二叉树结点的左子树的深度减去右子树深度的值称为平衡因子BF。只要有一个结点的BF的绝对值大于1，则二叉树就不平衡。

最小不平衡子树：距离插入结点最近的，且平衡因子的绝对值大于1的结点为根的子树，称为最小不平衡子树。

所谓平衡二叉树，就是在二叉排序树的创建过程中保证它的平衡性。一旦发现不平衡的情况，马上处理。
当最小不平衡树根节点的平衡因子大于1时，就右旋。
当最小不平衡树根结点的平衡因子小于1时，就左旋。
插入结点后，最小不平衡子树的BF与它的子树的BF符号相反时，就需要对结点先进行一次旋转使得符号相同后，再反向旋转一次。


### 多路查找树（B树）

B树是一种平衡的多路查找树。2-3树和2-3-4树都是B树的特例。结点最大的孩子数目称为B树的阶。
一个m阶的B树具有以下属性：

* 如果根节点不是叶节点，则其至少有两棵子树。
* 每一个非根结点都有k-1个元素和k个孩子。每一个叶子结点都有k-1个元素。
* 所有的叶子结点都处于同意层次。
* 所有分支结点包含下列信息（n,A0,K1,A1,K2...KN,AN）。其中K为关键字，A为指向子树根节点的指针，n为关键字的个数。

在B树上查找的过程是一个顺指针查找结点和在结点中查找关键字的交叉过程。

比如查找数字7，先读取到根节点3、5、8三个元素，发现7不在其中，但在5和8之间，因此通过A2指针找到6、7
结点，查找到索要的元素。


