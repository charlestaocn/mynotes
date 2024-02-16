---
aliases:
  - "#skiplist"
---

![[a00696ec2716ddf3f323b5fc9bd42ddd_MD5.png]]![[a7e863ed2f45694b8126c8753638a408_MD5.png]]




### 插入数据随机产生层数

```c
/* 返回一个 1-32 随机整数，并且返回高层数的几率较小(一般都是低层数)  */

#define ZSKIPLIST_MAXLEVEL 32 /* Should be enough for 2^64 elements */  
#define ZSKIPLIST_P 0.25      /* Skiplist P = 1/4 */

 int zslRandomLevel(void) {  
    int level = 1;  
    while ((random()&0xFFFF) < (ZSKIPLIST_P * 0xFFFF))  
        level += 1;  
    return (level<ZSKIPLIST_MAXLEVEL) ? level : ZSKIPLIST_MAXLEVEL;  
}

```

![[ee7e59a50872eaf7a7cd22552a2de671_MD5.png]]


```c
#define ZSKIPLIST_MAXLEVEL 32 /* Should be enough for 2^64 elements */  
#define ZSKIPLIST_P 0.25      /* Skiplist P = 1/4 */

/* ZSETs use a specialized version of Skiplists */  
typedef struct zskiplistNode {  
    sds ele; // 节点存储的具体数据 
    double score;  // 节点对应的分值 
    struct zskiplistNode *backward;  // 前向指针 要查找的比当前小需要向前回
    struct zskiplistLevel {  
        struct zskiplistNode *forward;  //后向指针
        unsigned long span;  //当前的向后指针跨越了多少个节点
    } level[];  
} zskiplistNode;  
  
typedef struct zskiplist {  
    struct zskiplistNode *header, *tail;  // 头尾指针
    unsigned long length;  // skiplist的长度
    int level;  // 最高的那个层数
} zskiplist;
```

