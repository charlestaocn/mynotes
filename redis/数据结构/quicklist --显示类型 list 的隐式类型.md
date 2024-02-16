

##  ==quicklist是[双向链表](linkedlist）和压缩列表（ziplist）组合的列表。==

## 为什么使用quicklist

>- 双向链表（linkedlist）实现高效的插入和删除；  
>- 压缩列表（ziplist）内存连续，减少内存碎片，访问效率高；

>3.2之前使用linkedlist 增删快 但是查询慢(需要遍历，每次只能遍历一个内容得到后一个指针才能向后)  
>
```java
/* 获取第四个元素需要多次遍历*/

%% 
get(4); ===>

1.next => 2.next => 3.next => 4.val 

 %%

```


>使用 ziplist 分存 之后，根据


```java
/* 获取第四个元素需要多次遍历*/

%%根据list中个数 判断是否在当前这个list 中 ，在的话直接  查询当前list  不在 就跳过当前list 跳到下个list 再次计算 count 直到找到 %%
%% 这样不需要 每个内容都遍历 ，可以跳过很多次 遍历  弥补了查询速度的缺陷 %%

```


```c
typedef struct quicklist {
    quicklistNode *head;         /* 指向第一个node */
    quicklistNode *tail;         /* 指向最后一个node */
    unsigned long count;        /* total count of all entries in all ziplists */
    unsigned long len;          /* number of quicklistNodes */
    int fill : QL_FILL_BITS;              /* fill factor for individual nodes */
    unsigned int compress : QL_COMP_BITS; /* depth of end nodes not to compress;0=off */
    unsigned int bookmark_count: QL_BM_BITS;
    quicklistBookmark bookmarks[];
} quicklist;
```

![在这里插入图片描述](https://img-blog.csdnimg.cn/c7b5052393324fbc859b358977767733.png#pic_center)



```c

typedef struct quicklistNode {
    struct quicklistNode *prev;    /*指向前一个node*/
    struct quicklistNode *next;    /*指向最后一个node*/
    unsigned char *zl;             /*指向 ziplist*/
    unsigned int sz;             /* ziplist size in bytes */
    unsigned int count : 16;     /* count of items in ziplist */
    unsigned int encoding : 2;   /* RAW==1 or LZF==2 */
    unsigned int container : 2;  /* NONE==1 or ZIPLIST==2 */
    unsigned int recompress : 1; /* was this node previous compressed? */
    unsigned int attempted_compress : 1; /* node can't compress; too small */
    unsigned int extra : 10; /* more bits to steal for future usage */
} quicklistNode;

```


