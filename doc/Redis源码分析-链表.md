# 《Redis源码分析-链表》
链表在Redis中的应用非常广泛，比如列表键的底层实现之一就是链表。当一个列表键包含了数量比较多的元素，又或者列表中包含的元素都是比较长的字符串时，Redis就会使用链表作为列表键的底层实现。  
本章接下来的内容将对Redis的链表实现进行介绍，并列出相应的链表和链表节点API。
**本节内容**
* [链表和链表节点得实现](#1)
* [链表和链表节点API](#2)
* [总结](#3)

##  
<h2 id="1">链表和链表节点的实现</h2>  

每个链表节点使用一个adlist.h/list结构来表示：  
```C
/* Node, List, and Iterator are the only data structures used currently. */
/* listNode结点 */
typedef struct listNode {
	//结点的前一结点
    struct listNode *prev;
    //结点的下一结点
    struct listNode *next;
    //Node的函数指针
    void *value;
} listNode;
```
多个listNode可以通过prev和next指针组成双端链表，如下图所示。  
![image](https://github.com/xiethon/Redis-3.0/blob/master/doc/photos/链表.png)  
虽然仅仅使用多个listNode结构就可以组成链表，但使用adlist.h/list来持有链表的话，操作起来会更方便：
```C
/* listNode 列表 */
typedef struct list {
	//列表头结点
    listNode *head;
    //列表尾结点
    listNode *tail;
    
    /* 下面3个方法为所有结点公用的方法，分别在相应情况下回调用 */
    //复制函数指针
    void *(*dup)(void *ptr);
    //释放函数指针
    void (*free)(void *ptr);
   	//匹配函数指针
    int (*match)(void *ptr, void *key);
    //列表长度
    unsigned long len;
} list;
```
list结构为链表提供了表头指针head、表尾指针tail，以及链表长度计数器len，而dup、free和match成员则是用于实现多态链表所需的类型特定函数:
* list结构为链表提供了表头指针head、表尾指针tail，以及链表长度计数器len，而dup、free和match成员则是用于实现多态链表所需的类型特定函数。
*  free 函数用于释放链表节点所保存的值；
*  match函数则用于对比链表节点所保存的值和另一个输入值是否相等。  

下图是由一个list结构和三个listNode结构组成的链表。
![image](https://github.com/xiethon/Redis-3.0/blob/master/doc/photos/list.png)  

**Redis的链表实现的特性可以总结如下：**
* 双端：链表节点带有prev和next指针，获取某个节点的前置节点和后置节点的复杂度都是O(1)；
* 无环：表头节点的prev指针和表尾节点的next指针都指向NULL，对链表的访问以NULL为终点。 
* 带表头指针和表尾指针：通过list结构的head指针和tail指针，程序获取链表的表头节点和表尾节点的复杂度为O(1)。  
* 带链表长度计数器：程序使用list结构的len属性来对list持有的链表节点进行计数，程序获取链表中节点数量的复杂度为O(1)。
* 多态：链表节点使用void*指针来保存节点值，并且可以通过list结构的dup、free、match三个属性为节点值设置类型特定函数，所以链表可以用于保存各种不同类型的值。  

<h2 id="2">链表和链表节点的API</h2>  

下图列出了用于操作链表和链表节点的API  
![image](https://github.com/xiethon/Redis-3.0/blob/master/doc/photos/list_1.png)  
![image](https://github.com/xiethon/Redis-3.0/blob/master/doc/photos/list_2.png)

<h2 id="3">总结</h2>  

* 链表被广泛用于实现Redis的各种功能，比如列表键、发布与订阅、慢查询、监视器等；
* 每个链表节点由一个listNode结构来表示，每个节点都有一个指向前置节点和后置节点的指针，所以Redis的链表实现是双端链表。
* 每个链表使用一个list结构来表示，这个结构带有表头节点指针、表尾节点指针，以及链表长度等信息。
* 因为链表表头节点的前置节点和表尾节点的后置节点都指向NULL，所以Redis的链表实现是无环链表。 
* 通过为链表设置不同的类型特定函数，Redis的链表可以用于保存各种不同类型的值。
* [更多解析请参阅源码注释](https://github.com/xiethon/Redis-3.0/tree/master/src_note/list)