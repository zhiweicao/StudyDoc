## 一个Oracle调优

场景：

在一个Oracle的数据库中，有一部分表（大小在100万到几千万行不等），当算法团队使用以下语句获取前1000行

```sql
select * from 表名 where ROWNUM < 1000;
```

这个操作一般要花费30分钟左右。而在一些哪怕有5000万行的表中，查询前1000行也只需要1s以内，而这种情况的表目前大概有几十个。

### 问题分析

首先，30分钟大概率是发生了全表扫描的问题，从百度上了解到，

#### 尝试一：

Oracle有高水位的问题，即表插入的字段后删除，数据库水位仍然保留，于是导致数据库扫描了很多已经删除行，后来经查询不是。

那么基本就可以确定是索引问题，我先查了慢SQL表的索引，发现没有主键索引，只有普通索引，这也是导致SQL执行速度慢的原因。

#### 尝试二：

刚开始受错误资料的影响，以为没有主键索引，于是以普通索引的ID为切入点，尝试链表查询，后来发现这种场景下，仍然需要整表扫描，猜测可能是因为有索引中大量的重复ID。

#### 尝试三：

后来，对rownum进行查询，rownum被称为伪行，是动态生成的。但是Oracle有一个ROWID的字段，是一串哈希值，是行对应的物理地址，不能直接进行小于1000的Select。于是我先Select这个表前1000行的ROWID，然后使用这一千行的ROWID直接和原表的所有内容和ROWID进行并表查询。最终查询时间从30多分钟缩短到0.3秒

```sql
select c.*
from (
  select ROWID ri
  from HX.HX_SNAP_CST_ITEM_COSTS 
  where ROWNUM < 10000
) t ,(
  SELECT a.*, ROWID ri /*+ index(HX_SNAP_CST_ITEM_COSTS ROWNUM) */
  from HX.HX_SNAP_CST_ITEM_COSTS a
) c
where t.ri = c.ri
```

