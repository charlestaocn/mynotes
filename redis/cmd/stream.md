---
Aliases: [ "#stream" ]
---
### 添加到 stream

```redis
XADD key [NOMKSTREAM] [<MAXLEN | MINID> [= | ~] threshold
  [LIMIT count]] <* | id> field value [field value ...]
```


![[b08a9be9861ec24d33e9755aac797125_MD5.webp]]

### 添加 xadd

```redis 

%% 不指定id  自动生成  <millisecondsTime>-<sequenceNumber>  **满足自增的特性，支持范围查找** %% 
redis> XADD stream-key * field1 value1 field2 value2 field3 value3
"1703837217845-0"

---------------------------------------

%% 指定 id  形式必须是 "整数-整数" %% 
redis> xadd stream-key 2-0 field1 value1 
"2-0"

---------------------------------------

%% 指定大小   %%
%% 不指定 id 后面的会将前面的挤出    %%
redis> XADD stream-key  maxlen =1   * field1 value1 field2 value2 
"1703837217845-0"

redis> XADD stream-key  maxlen =1   * field1 value1 field3 value3
"1703838356891-0""

%%  指定id 会直接 报错 %%

ERR The ID specified in XADD is equal or smaller than the target stream top item


```

### 查看信息 xinfo

```redis

xinfo stream stream-key

 1) "length"
 2) (integer) 1
 3) "radix-tree-keys"
 4) (integer) 1
 5) "radix-tree-nodes"
 6) (integer) 2
 7) "last-generated-id"
 8) "2-0"
 9) "groups"
10) (integer) 0
11) "first-entry"
12) 1) "2-0"
    2) 1) "field1"
       2) "value1"
13) "last-entry"
14) 1) "2-0"
    2) 1) "field1"
       2) "value1"


```
### 查看内容 xrange / XREVRANGE(反向查看)


![xrange获取消息数据](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5fb9744fe7d143dc8dbe03d9ef7f08b6~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)



```redis

%% 指定范围内的数据  [1636003481706-1,1636003499055-0] %%

xrange stream-key 1636003481706-1 1636003499055-0

--------------------------------------------------------------------


%% 指定范围内的数据  (1636003481706-1,1636003499055-0) %%
xrange stream-key (1636003481706-1 (1636003499055-0

--------------------------------------------------------------------


%% 指定时间戳范围内的数据  (1636003481706,1636003499055) %%
xrange stream-key (1636003481706 (1636003499055


--------------------------------------------------------------------


%%  查看 key 中 所有范围的消息  %%
%%
`-` 表示最小id的值
`+`表示最大id的值 
%%
xrange key - + [COUNT count]

--------------------------------------------------------------------

%% 返回固定条数  %%
xrange key - + count 1 


```


### 删除消息 XDEL

```redis 

xdel key ID [ID ...]

```


### 查看长度 xlen

```redis

xlen key

```

### 条件删除 xtrim

```redis
xtrim key MAXLEN|MINID [=|~] threshold [LIMIT count]
```

- MAXLEN: 从末尾开始计数，超过个数的被删除
- MINID： 删除比给定id 小的 

### 读取steam 消息 xread

```redis

%% 非阻塞 read  %%
xread  streams stream-key  id
--------------------------------------
%%  阻塞 read 是多播的，多个 cli  block read ，有消息进入时 都会 read 到  %%
%% 没有消息时 阻塞住 超时时间后 返回  %%

xread block 10000 streams stream-key id

```
