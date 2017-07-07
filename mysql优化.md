# Mysql原理及优化

# 一.原理
SQL语句的解析顺序
简单的说一个sql语句是按照如下的顺序解析的：

1.   FROM
2.   WHERE
3.	GROUP BY
4.	HAVING
5.	SELECT 
6.	ORDER BY

 

执行语句：
```mysql
SELECT
	a.aid,
	count(b.bid)
FROM
	a
LEFT JOIN b ON a.aid = b.aid
WHERE
	a.aname = 'apple'
GROUP BY
	a.aid
HAVING
	count(b.bid) < 3
ORDER BY
	count(b.bid);
```

### 1.from
对FROM子句中的前两个表执行笛卡尔积，生成虚拟表VT1。
如果左表有m行，右表有n行，那么笛卡尔积之后生成的VT1-J1表将会有m×n行
这一步等价于：
```mysql
SELECT
	*
FROM
	a
JOIN b
```


### 2.on
对VT1应用ON筛选器。 满足条件的行被插入VT2。

```mysql
SELECT
	*
FROM
	a
LEFT JOIN b ON a.id = b.aid
```

### 3.where
对VT3应用WHERE筛选器。只满足条件的行被插入VT4.
```mysql
SELECT
	*
FROM
	a
LEFT JOIN b ON a.id = b.aid
WHERE
	a.aname = 'apple'
```


### 4.group by ,  having
对VT4中按group by的字段分组，生成VT4。
对VT4应用HAVING筛选器件，生成VT5。


### 5.select
计算distinct, 聚合表达式(count,max..) ，并根据上一步虚拟表查找出的select *， 查询出需要的列 。

### 6.order by
对结果进行order by排序。


# 二.	索引篇
### 1. 索引的原理(B+树算法) 
![](http://jbcdn2.b0.upaiyun.com/2016/05/15c4b064af9ac7f357404a1b17ff1cae.png) 

### 2. 以下注意的写法，否则将导致添加的索引无效而进行全表扫描 
1. 应尽量避免在 where 子句中对字段进行 null 值判断 
```
select id from t where num is null;
```
可以在 num 上设置默认值 0（或者-1，或者空字符串，等等）,确保表中 num 列没有 null 值，然后这样查询
```
select id from t where num=0;
```
2. 应尽量避免在 where 子句中使用!=或<>操作符

3. 尽量避免在 where 子句中使用 or 来连接条件
```mysql
select id from t where num=10 or num=20;
```
可以这样查询：
```mysql
select id from t where num=10 union all select id from t where num=20;
```
4. 对于连续的数值，能用 between 就不要用 in 
```mysql
 select id from t where num between 1 and 3;
 ```
 
5. 下面的查询也将导致全表扫描：
```mysql
select id from t where name like '%c%';
```
若要提高效率，可以考虑全文检索（看数据量，避免过早优化，全文检索成本较高）。

6. 应尽量避免在 where 子句中对字段进行表达式操作（将操作移至等号右侧）
```mysql
select id from t where num/2=100;
```
可以这样写：
```mysql
select id from t where num=100*2;
```

7. 尽量避免在 where 子句中对字段进行函数操作
```mysql
select id from t where left(name,3)='abc';  
```
应改为：
```mysql
select id from t where name like 'abc%';
```

### 3.  Cardinality
*  表示**索引列唯一值**的个数。如source列的值只存在0,1 或者sex列的值只存在male,female，那么Cardinality就是2。
*  即使该列加了索引，mysql优化器会根据Cardinality判断是否用索引，如果Cardinality太小，放弃索引而全盘扫描的速度会更快。 
*  在Cardinality越大的列上建索引，索引效果越明显（如id）。

### 4.  联合索引
* 一次sql查询对于一张表**不能**用多个索引 
* 假设2个模块分别用上这两条语句

```mysql
select * from users where area='beijing' and age=22;
select * from users where area='guangzhou';
```
* 这时候可以建联合索引 index1(area,age)。
* 如果建立的是index2(age,area)，则改索引对第二条语句无效。
* 最左前缀原则：index(d1,d2,d3)相当于建了3个索引(d1), (d1,d2), (d1,d2,d3)。有两列需要用索引时，会用d1,d2；只有一列需要用索引时，只会用d1。
* 创建联合索引时应该将**最常用作限制条件的列**放在最左边。



 
# 三.其他优化技巧
#### 1. 对查询进行优化，应尽量避免全表扫描，首先应考虑在 where 及 order by 涉及的列上建立索引。

#### 2. 	索引并对连接使用同样的字段类型
* 如果你的sql语句包含许多连接查询, 你需要确保连接的字段在两张表上都建立了索引。 这会影响MySQL如何内部优化连接操作。
* 被连接的字段,需要使用同样类型。例如, 如果你使用一个DECIMAL字段, 连接另一张表的INT字段, MySQL将无法使用至少一个索引。 即使字符编码也需要使用相同的字符类型。
 
#### 3. 避免使用SELECT *
* 不要返回用不到的任何字段。
* 从数据表中读取的数据越多，查询操作速度就越慢。它增加了数据库服务器磁盘操作所需的时间。
* 当数据库服务器与Web服务器分开时，由于必须在服务器之间传输数据，然后web服务器还需要把数据返回给客户端，将会有更长的网络延迟。 

#### 4.  限制工作数据集的大小
当用到跨表查询，或者子查询，尽量在每张表的内部语句做过滤，而不是在外部。
```mysql
SELECT
	a.xm,
	a.xb,
	a.xl,
	a.age
FROM
	table1 a,
	table2 b
WHERE
	a.id = b.id
AND a.date BETWEEN '2009-01-01'
AND '2009-02-01'; 
```
可以改成
```mysql
select a.*
from (
    select xm,xb,xl,age
    from table1
    where date between '2009-01-01' and  '2009-02-01'
) a inner join table2 b where a.id=b.id
```

#### 5.  尽量避免在后台代码使用for循环查询N条语句

#### 6.  可做适当冗余
* 关联表的字段冗余
* 计算结果冗余
 

# 四. Nosql(redis)
* 优点：海量数据的增删改查，非常轻松应对
* 缺点：数据之间没有关系，不能一目了然； 没有强大的事务保证数据的完整和安全。
* 可用于首页内容缓存，访问量统计等。
 
####  redis中value有5中数据类型
* **string**
* **hash**
相当于java中的HashMap，适合存入javabean
* **list**
相当于LinkedList，插入或者更新速度快。
* **set**
相当于HashSet，适合做交集，并集，差集运算。
* **zset**
加score的set，合适做排行榜。
