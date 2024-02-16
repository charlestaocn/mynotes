>ziplist 是一个特殊双向链表，不像普通的链表使用前后指针关联在一起，它是存储在连续内存上的。

- zlbytes: 32 位无符号整型，记录 ziplist 整个结构体的占用空间大小。当然了也包括 zlbytes 本身。这个结构有个很大的用处，就是当需要修改 ziplist 时候不需要遍历即可知道其本身的大小。
- zltail: 32 位无符号整型, 记录整个 ziplist 中最后一个 entry 的偏移量。所以在尾部进行 POP 操作时候不需要先遍历一次。
- zllen: 16 位无符号整型, 记录 entry 的数量， 所以只能表示 2^16。但是 Redis 作了特殊的处理：当实体数超过 2^16 ,该值被固定为 2^16 - 1。 所以这种时候要知道所有实体的数量就必须要遍历整个结构了。
- entry: 真正存数据的结构。
- zlend: 8 位无符号整型, 固定为 255 (0xFF)。为 ziplist 的结束标识。

> **entry** 中包含有previous_entry_length、encoding、content等属性

![[redis/_resources/ziplist/0a51bbd488054b7a9bf7336307004758_MD5.png]]



[[redis/_resources/ziplist/cb70e416aaa9175346f62a81ff86258f_MD5.jpeg|Open: 截屏2023-12-26 16.14.42.png]]
![[redis/_resources/ziplist/cb70e416aaa9175346f62a81ff86258f_MD5.jpeg]]


[[redis/_resources/ziplist/c0f311f19fe5b3e913545335f40e7079_MD5.jpeg|Open: 截屏2023-12-26 16.30.49.png]]
![[redis/_resources/ziplist/c0f311f19fe5b3e913545335f40e7079_MD5.jpeg]]



第一中情况：一般结构 <prevlen> <encoding> <entry-data>

- prevlen：前一个entry的大小；
- encoding：不同的情况下值不同，用于表示当前entry的类型和长度；
- entry-data：真是用于存储entry表示的数据；

第二种情况：此时entry结构：<prevlen> <encoding>

- 在entry中存储的是 int类型时，encoding和entry-data会合并在encoding中表示，此时没有entry-data字段；
- redis中，在存储数据时，会先尝试将string转换成int存储，节省空间；

> **prevlen编码：**当前一个元素长度小于254（255用于zlend）的时候，prevlen长度为1个字节，值即为前一个entry的长度，如果长度大于等于254的时候，prevlen用5个字节表示，第一字节设置为254，后面4个字节存储一个小端的无符号整型，表示前一个entry的长度；
> 
> **encoding编码：** encoding的长度和值根据保存的是int还是string，还有数据的长度而定； 前两位用来表示类型，当为“11”时，表示entry存储的是int类型，其他表示存储的是string； 

存储int时：

- |11000000| encoding为3个字节，后2个字节表示一个int16；
- |11010000| encoding为5个字节，后4个字节表示一个int32;
- |11100000| encoding 为9个字节，后8字节表示一个int64;
- |11110000| encoding为4个字节，后3个字节表示一个有符号整型；
- |11111110| encoding为2字节，后1个字节表示一个有符号整型；
- |1111xxxx| encoding长度就只有1个字节，xxxx表示一个0 - 12的整数值；

存储string时：

- |00pppppp| ：此时encoding长度为1个字节，该字节的后六位表示entry中存储的string长度，因为是6位，所以entry中存储的string长度不能超过63；
- |01pppppp|qqqqqqqq| 此时encoding长度为两个字节；此时encoding的后14位用来存储string长度，长度不能超过16383；
- |10000000|qqqqqqqq|rrrrrrrr|ssssssss|ttttttt| 此时encoding长度为5个字节，后面的4个字节用来表示encoding中存储的字符串长度，长度不能超过2^32 - 1;