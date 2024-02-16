
- 当数据较少时，sorted set是由一个ziplist来实现的。
- 当数据多的时候，sorted set是由一个叫zset的数据结构来实现的，这个zset包含一个dict + 一个 #skiplist 。dict用来查询数据到分数(score)的对应关系，而skiplist用来根据分数查询数据（可能是范围查找）。

```c

/*

- 当sorted set中的元素个数，即(数据, score)对的数目超过128的时候，也就是ziplist数据项超过256的时候。
- 当sorted set中插入的任意一个数据的长度超过了64的时候。
*/


if (server.zset_max_ziplist_entries == 0 ||  
    server.zset_max_ziplist_value < sdslen(c->argv[scoreidx+1]->ptr))  
{  
    zobj = createZsetObject();  
} else {  
    zobj = createZsetZiplistObject();  
}

```



```c
typedef struct zset {  
    dict *dict;  
    zskiplist *zsl;  
} zset;
```