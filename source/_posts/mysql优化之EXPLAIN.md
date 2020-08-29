---
title: mysql优化之EXPLAIN
categories:
- 数据库
date: 2018-04-05 16:50:34
tags:
- mysql
---
MySQL可以通过EXPLAIN或DESC来查看并分析SQL语句的执行情况;   
explain为mysql提供语句的执行计划信息。可以应用在 SELECT, DELETE, INSERT, REPLACE, UPDATE 语句上。  
explain的执行计划，只是作为语句执行过程的一个参考，实际执行的过程不一定和计划完全一致，但是执行计划中透露出的讯息却可以帮助选择更好的索引和写出更优化的查询语句。    
还可以使用 EXPLAIN检查优化程序是否以最佳顺序连接表...  


# EXPLAIN Output Format
参考: https://dev.mysql.com/doc/refman/8.0/en/explain-output.html     

explain 会为select语句中用到的每一个表返回一行信息,       
EXPLAIN输出中的表,按照mysql在处理语句时读取的顺序列出;      


## EXPLAIN Output Columns

Column          |  JSON Name       |  Meaning
---             |:--:               |---:
id              |  select_id       | The SELECT identifier
select_type        |   None           | The SELECT type
table          |   table_name     | The table for the output row
partitions     |   partitions     | The matching partitions
type           |   access_type        | The join type
possible_keys  |   possible_keys  | The possible indexes to choose
key                |   key                |The index actually chosen
key_len            |   key_length     |The length of the chosen key
ref                |   ref                |The columns compared to the index
rows           |   rows           |Estimate of rows to be examined
filtered       |   filtered       |Percentage of rows filtered by table condition
Extra          |   None           |Additional information

下面对表头的每一行进行解释说明:        
### id      
select 标识, select在查询时的序列号, 表示的是查询中执行select子句或者是操作表的顺序。      
说明:     
id相同表示加载表的顺序是从上到下。  
id不同id值越大，优先级越高，越先被执行。  
id有相同，也有不同，同时存在。id相同的可以认为是一组，从上往下顺序执行；在所有的组中，id的值越大，优先级越高，越先执行。 
如果行引用其他行的联合结果，则该值可以为Null。在这种情况下，table列会显示一个类似于< union m，n>的值，以指示该行是explain输出表中select的id值为m和n行的并集。    

示例:     
```sql
mysql> 
EXPLAIN
SELECT t1.name,(SELECT id from t2 where id=2) as id2 
FROM `t1` LEFT JOIN t3 on t3.id=t1.id
UNION
(SELECT name,id from t2);

+------+--------------+------------+------------+--------+---------------+---------+---------+----------------+------+----------+-----------------+
| id   | select_type  | table      | partitions | type   | possible_keys | key     | key_len | ref            | rows | filtered | Extra           |
+------+--------------+------------+------------+--------+---------------+---------+---------+----------------+------+----------+-----------------+
|    1 | PRIMARY      | t1         | NULL       | ALL    | NULL          | NULL    | NULL    | NULL           |    3 |      100 | NULL            |
|    1 | PRIMARY      | t3         | NULL       | eq_ref | PRIMARY       | PRIMARY | 4       | leetcode.t1.id |    1 |      100 | Using index     |
|    2 | SUBQUERY     | t2         | NULL       | const  | PRIMARY       | PRIMARY | 4       | const          |    1 |      100 | Using index     |
|    3 | UNION        | t2         | NULL       | ALL    | NULL          | NULL    | NULL    | NULL           |    3 |      100 | NULL            |
| NULL | UNION RESULT | <union1,3> | NULL       | ALL    | NULL          | NULL    | NULL    | NULL           | NULL | NULL     | Using temporary |
+------+--------------+------------+------------+--------+---------------+---------+---------+----------------+------+----------+-----------------+
```
select的执行顺序依次为 3 -> 2 -> 1 (id=1第一行) -> 1 (id=1的第二行) -> NULL        

### select_type
查询的类型,类型如下表所示:      

select_type Value           | JSON Name                      | Meaning
---                         |:--:                           |---:
SIMPLE                   | None                      | Simple SELECT (not using UNION or subqueries)
PRIMARY                      | None                      | Outermost SELECT
UNION                    | None                      | Second or later SELECT statement in a UNION
DEPENDENT UNION                | dependent (true)             | Second or later SELECT statement in a UNION, dependent on outer query
UNION RESULT               | union_result                | Result of a UNION.
SUBQUERY                  | None                      | First SELECT in subquery
DEPENDENT SUBQUERY          | dependent (true)             | First SELECT in subquery, dependent on outer query
DERIVED                      | None                      | Derived table
DEPENDENT DERIVED           | dependent (true)             | Derived table dependent on another table
MATERIALIZED               | materialized_from_subquery      | Materialized subquery
UNCACHEABLE SUBQUERY         | cacheable (false)                | A subquery for which the result cannot be cached and must be re-evaluated for each row of the outer query
UNCACHEABLE UNION           | cacheable (false)                | The second or later select in a UNION that belongs to an uncacheable subquery (see UNCACHEABLE SUBQUERY)

1. SIMPLE : 简单的查询,没有用到联合查询或者子查询
2. PRIMARY : 如果查询有任何复杂的子查询，则最外层标记为PRIMARY（比如说查询包含DERIVED、UNION、UNION RESUlT...这些, 则最外层的那个查询标记为 PRIMARY）
3. UNION : union查询中的第二个或者最后语句; select ... t1 UNION (select .... t2 ),此时第二个查询语句select_type会被标上union 
4. DEPENDENT UNION : 在UNION中第二个或最后的SELECT语句，同时该语句依赖外部的查询. 
5. UNION RESULT : 联合查询的结果
6. SUBQUERY : 子查询中的第一个select
7. DEPENDENT SUBQUERY : 
8. DERIVED : 派生表. ( 参考:https://www.yiibai.com/mysql/derived-table.html )
9. DEPENDENT DERIVED : 
10. MATERIALIZED : 
11. UNCACHEABLE SUBQUERY : 
12. UNCACHEABLE UNION : 

派生表
set optimizer_switch='derived_merge=off';       
EXPLAIN SELECT name from (SELECT id,name from t1 ) as T;        
参考: https://www.cnblogs.com/ivictor/p/9281488.html      


### TABLE   
输出行所引用的表的名称。这也可以是以下值之一：     
* \< union M,N\>  
* \< derived N\>  
* \< subquery N\> 


### partitions
查询匹配到记录的分区。非分区表的该值为null。        


### type
EXPLAIN Join Types      
表示表的连接类型;       
**下面来描述这些join type, 从最佳type到最差的顺序说明:**  
1. system :     
表只有一行  (= system table)     
2. const :
表最多有一个匹配行，该行在查询开始时读取。因为只有一行，所以优化器的其余部分可以将此行列中的值视为常量。const表非常快，因为它们只读一次。     
当你将PRIMARY KEY或 UNIQUE index与常量值进行比较时，使用就是const(如果是联合主键或联合唯一索引,需要将这个索引的所有组成部分都进行常量比较)。      
如下示例：       
 ```sql
# 单一主键
SELECT * FROM tbl_name WHERE primary_key=1;
# 联合主键
SELECT * FROM tbl_name WHERE primary_key_part1=1 AND primary_key_part2=2;
```
3. eq_ref : (equal  reference ?)
只匹配到一行的时候。除了system和const之外，这是最好的连接类型了。当我们使用主键索引或者唯一索引的时候，(如果是联合主键或联合唯一索引,需要将这个索引的所有组成部分都用到),就会是该类型。         
(简单来说，就是多表连接中使用primary key或者 unique key作为关联条件)          
在对已经建立索引列(要保证每次只匹配一行,所以得是唯一索引)进行 = 操作的时候，eq_ref会被使用到。比较值可以使用一个常量也可以是一个表达式, 这个表达示可以是其他的表的行。          
```sql
#. 多表关联查询，单行匹配
SELECT * FROM ref_table,other_table
  WHERE ref_table.key_column=other_table.column;
-- 说明ref_table.key_column该行需有主键/唯一键,这样的话,最终对应的other_table.column在ref_table中最多只能匹配到一行
  
  
#. 多表关联查询，联合索引，多行匹配
SELECT * FROM ref_table,other_table
  WHERE ref_table.key_column_part1=other_table.column
  AND ref_table.key_column_part2=1;
```

4. ref :
可以理解为要比较的 值/表达式/其他表的行, 对应的在该表中的索引字段上有多行与之匹配;(与eq_ref不同的是匹配到了多行, 所以这个索引不能是唯一索引);     
匹配到的记录越少,性能越好;      
ref可以用于使用 = 、or 、 <=> 运算符进行比较的索引列;      
```sql
类似表A上有联合索引1 index_key1 ('country','age'); 单列索引2 index_key2 'email';
select * from A where `country`='EN';
select * from A where `email`='**';
上面两个explain的type将为ref   
```
示例:
```sql
# 根据索引（非主键，非唯一索引），匹配到多行
SELECT * FROM ref_table WHERE key_column=expr;

# 多表关联查询，单个索引，多行匹配
SELECT * FROM ref_table,other_table
  WHERE ref_table.key_column=other_table.column;

# 多表关联查询，联合索引，多行匹配
SELECT * FROM ref_table,other_table
  WHERE ref_table.key_column_part1=other_table.column
  AND ref_table.key_column_part2=1;
```
 
5. fulltext :
使用全文索引的时候才会出现
6. ref_or_null :        
这个查询类型和ref很像，但是 MySQL 会做一个额外的查询，来看哪些行包含了NULL (这是因为mysql单列索引不存null值，复合索引不存全为null的值，如果列允许为null，可能会得到"不符合预期"的结果集)。这种类型常见于解析子查询的优化。 
```sql
SELECT * FROM ref_table
  WHERE key_column=expr OR key_column IS NULL;
```
7. ~~index_merge~~ :        
参考: https://dev.mysql.com/doc/refman/8.0/en/index-merge-optimization.html       
8. ~~unique_subquery~~ :        
该类型替换了下面形式的IN子查询的eq_ref, 而且子查询是主键或者唯一索引     
```sql
value IN (SELECT primary_key FROM single_table WHERE some_expr)
```
unique_subquery 只是一个索引查找功能，它可以完全替换子查询以提高效率。 
9. index_subquery : 
类似 unique_subquery, 不同的是它在子查询里使用的是非唯一索引。    
```sql
value IN (SELECT key_column FROM single_table WHERE some_expr)
```
10. range : 
范围查询,常见于 >, >=, <, <=,  BETWEEN, LIKE, IN() 
11. index :
index类型和ALL类型一样，区别就是index类型是扫描的索引树。以下两种情况会触发：          
如果索引是查询的覆盖索引，就是说索引查询的数据可以满足查询中所需的所有数据，则只扫描索引树，不需要回表查询。 在这种情况下，explain 的 Extra 列的结果是 Using index。仅索引扫描通常比ALL快，因为索引的大小通常小于表数据。        
全表扫描会按索引的顺序来查找数据行。使用索引不会出现在Extra列中。     

12. ALL :
全表扫描 (赶紧看看能不能加索引, 重写更优化的查询语句等方式来优化下)        

### possible_keys 
显示查询时可能用到的索引        

### key
MySQL实际决定使用的索引      

### key_len
MySQL决定使用的索引的长度。该值使你可以确定MySQL实际使用了多列索引的多少部分; (在优化索引最左匹配原则的时候,你可以使用到)

### rows
该rows列表示MySQL认为必须检查以执行查询的行数。(即扫描的行数)        
对于InnoDB表，此数字是估算值，可能并不总是准确的。        

### filtered 
旧版本mysql使用explain extended时会出现这个列，5.7之后的版本默认就有这个字段，不需要使用explain extended了。这个字段表示存储引擎返回的数据在server层过滤后，剩下多少满足查询的记录数量的比例，注意是百分比，不是具体记录数。

### Extra   
此列包含有关MySQL如何解析查询的其他信息。     
常见情况:       
1. Using where:列数据是从仅仅使用了索引中的信息而没有读取实际的行动的表返回的，这发生在对表的全部的请求列都是同一个索引的部分的时候，表示mysql服务器将在存储引擎检索行后再进行过滤     
2. Using temporary：表示要解析查询，MySQL需要创建一个临时表来保存结果。常见于排序和分组查询   
3. Using filesort：mysql需要一次额外的传递来完成排序;(即在当前查询中mysql无法直接利用索引排序,需要额外执行排序操作)       
例如:      
```sql
-- 课程周期表
CREATE TABLE `uc_course` (
  `id` int(1) unsigned NOT NULL AUTO_INCREMENT,
  `parent` int(1) unsigned NOT NULL DEFAULT '0' COMMENT '对应课程表中的课程id',
  `name` varchar(150) NOT NULL COMMENT '课程周期名称',
  `org_id` int(1) unsigned NOT NULL,
  PRIMARY KEY (`id`),
  KEY `org_id` (`org_id`),
  KEY `parent` (`parent`) USING BTREE,
  KEY `org_and_parent` (`org_id`,`parent`) USING BTREE
) ENGINE=InnoDB AUTO_INCREMENT=0 DEFAULT CHARSET=utf8;

-- (1).可以直接使用索引排序的情况1:
mysql> EXPLAIN SELECT * FROM `uc_course` where  parent in (28,744,1387,1441) ORDER BY parent desc;
+----+-------------+-----------+------------+-------+---------------+--------+---------+------+------+----------+-----------------------+
| id | select_type | table     | partitions | type  | possible_keys | key    | key_len | ref  | rows | filtered | Extra                 |
+----+-------------+-----------+------------+-------+---------------+--------+---------+------+------+----------+-----------------------+
|  1 | SIMPLE      | uc_course | NULL       | range | parent        | parent | 4       | NULL |  159 |      100 | Using index condition |
+----+-------------+-----------+------------+-------+---------------+--------+---------+------+------+----------+-----------------------+
-- 可以发现该查询使用到的索引key=parent,就是我们order by的字段, 所以排序也能使用到该索引,Extra列没有出现Using filesort

-- (2.1).不能直接使用索引排序的情况1:
mysql> EXPLAIN SELECT * FROM `uc_course` where  org_id = 9 ORDER BY parent desc;
+----+-------------+-----------+------------+------+---------------+--------+---------+-------+------+----------+---------------------------------------+
| id | select_type | table     | partitions | type | possible_keys | key    | key_len | ref   | rows | filtered | Extra                                 |
+----+-------------+-----------+------------+------+---------------+--------+---------+-------+------+----------+---------------------------------------+
|  1 | SIMPLE      | uc_course | NULL       | ref  | org_id        | org_id | 4       | const |  639 |      100 | Using index condition; Using filesort |
+----+-------------+-----------+------------+------+---------------+--------+---------+-------+------+----------+---------------------------------------+
-- 上面的查询计划我们可以看到Extra列出现了Using filesort, 这是为什么了,我们的parent列上明明有索引? 
-- 这是因为这里的查询语句中用到的索引key=org_id,没有用到排序字段parent,所以在查询到数据后,mysql还要额外操作一次来完成排序;即会出现 using filesort;

-- (2.2).不能直接使用索引排序的情况2:
mysql> EXPLAIN SELECT * FROM `uc_course` where  org_id = 9 and  parent in (28,744,1387,1441) ORDER BY parent desc;
+----+-------------+-----------+------------+------+---------------+--------+---------+-------+------+----------+----------------------------------------------------+
| id | select_type | table     | partitions | type | possible_keys | key    | key_len | ref   | rows | filtered | Extra                                              |
+----+-------------+-----------+------------+------+---------------+--------+---------+-------+------+----------+----------------------------------------------------+
|  1 | SIMPLE      | uc_course | NULL       | ref  | org_id,parent | org_id | 4       | const |  639 |     22.3 | Using index condition; Using where; Using filesort |
+----+-------------+-----------+------------+------+---------------+--------+---------+-------+------+----------+----------------------------------------------------+
-- 这一次虽然我们在where中加上了对parent的条件查询,但是查询计划显示的最终我们使用的索引key只有org_id,还是没有用到parent,索引依旧出行了using firesort;

-- (3). 如何避免(2)中出现的情况了?
-- 添加复合索引: KEY `org_and_parent` (`org_id`,`parent`) USING BTREE
mysql> EXPLAIN SELECT * FROM `uc_course` where  org_id = 9 ORDER BY parent desc;
+----+-------------+-----------+------------+------+-----------------------+----------------+---------+-------+------+----------+-------------+
| id | select_type | table     | partitions | type | possible_keys         | key            | key_len | ref   | rows | filtered | Extra       |
+----+-------------+-----------+------------+------+-----------------------+----------------+---------+-------+------+----------+-------------+
|  1 | SIMPLE      | uc_course | NULL       | ref  | fk_org,org_and_parent | org_and_parent | 4       | const |  639 |      100 | Using where |
+----+-------------+-----------+------------+------+-----------------------+----------------+---------+-------+------+----------+-------------+
1 row in set

mysql> EXPLAIN SELECT * FROM `uc_course` where  org_id = 9 and  parent in (28,744,1387,1441) ORDER BY parent desc;
+----+-------------+-----------+------------+-------+------------------------------+----------------+---------+------+------+----------+-----------------------+
| id | select_type | table     | partitions | type  | possible_keys                | key            | key_len | ref  | rows | filtered | Extra                 |
+----+-------------+-----------+------------+-------+------------------------------+----------------+---------+------+------+----------+-----------------------+
|  1 | SIMPLE      | uc_course | NULL       | range | fk_org,parent,org_and_parent | org_and_parent | 8       | NULL |  141 |      100 | Using index condition |
+----+-------------+-----------+------------+-------+------------------------------+----------------+---------+------+------+----------+-----------------------+
-- 我们可以看到并没有再出现using filesort; 因为查询命中的索引key是org_and_parent,它是复合索引,包含了where中需要的org_id,也包括了order by用到的字段parent; 

-- (4).如果查询需要关联多张表,则只有当order by子句引用的字段**全部**为第一个表时,才能使用索引做排序;
-- 该点具体参考: <<高性能mysql>>一书的5.3.7节
-- 课程表结构:
CREATE TABLE `uc_course_package` (
  `id` int(1) unsigned NOT NULL AUTO_INCREMENT,
  `name` varchar(150) NOT NULL COMMENT '课程名称',,
  PRIMARY KEY (`id`),
) ENGINE=InnoDB AUTO_INCREMENT=0 DEFAULT CHARSET=utf8;
-- (4.1)当order by 字段来自第一张表时: 可利用索引排序
mysql> EXPLAIN SELECT * FROM `uc_course` as c LEFT JOIN uc_course_package as p on c.parent=p.id where  parent in (28,744,1387,1441) ORDER BY c.parent desc;
+----+-------------+-------+------------+--------+---------------+---------+---------+---------------+------+----------+-----------------------+
| id | select_type | table | partitions | type   | possible_keys | key     | key_len | ref           | rows | filtered | Extra                 |
+----+-------------+-------+------------+--------+---------------+---------+---------+---------------+------+----------+-----------------------+
|  1 | SIMPLE      | c     | NULL       | range  | parent        | parent  | 4       | NULL          |  159 |      100 | Using index condition |
|  1 | SIMPLE      | p     | NULL       | eq_ref | PRIMARY       | PRIMARY | 4       | uooc.c.parent |    1 |      100 | NULL                  |
+----+-------------+-------+------------+--------+---------------+---------+---------+---------------+------+----------+-----------------------+
-- (4.2)当order by 字段来自后面的表时: 出现 using filesort 
mysql> EXPLAIN SELECT * FROM  uc_course_package as p  LEFT JOIN `uc_course` as c  on c.parent=p.id where  parent in (28,744,1387,1441) ORDER BY c.parent desc;
+----+-------------+-------+------------+-------+---------------+---------+---------+-----------+------+----------+----------------------------------------------+
| id | select_type | table | partitions | type  | possible_keys | key     | key_len | ref       | rows | filtered | Extra                                        |
+----+-------------+-------+------------+-------+---------------+---------+---------+-----------+------+----------+----------------------------------------------+
|  1 | SIMPLE      | p     | NULL       | range | PRIMARY       | PRIMARY | 4       | NULL      |    4 |      100 | Using where; Using temporary; Using filesort |
|  1 | SIMPLE      | c     | NULL       | ref   | parent        | parent  | 4       | uooc.p.id |    1 |      100 | NULL                                         |
+----+-------------+-------+------------+-------+---------------+---------+---------+-----------+------+----------+----------------------------------------------+

```
`In some cases, MySQL cannot use indexes to resolve the ORDER BY, although it still uses indexes to find the rows that match the WHERE clause. These cases include the following:
 The key used to fetch the rows is not the same as the one used in the ORDER BY.`   
参考: https://dev.mysql.com/doc/refman/5.5/en/order-by-optimization.html  

4. Using join buffer：改值强调了在获取连接条件时没有使用索引，并且需要连接缓冲区来存储中间结果。如果出现了这个值，那应该注意，根据查询的具体情况可能需要添加索引来改进能。 
5. Impossible where：这个值强调了where语句会导致没有符合条件的行。   
6. Select tables optimized away：这个值意味着仅通过使用索引，优化器可能仅从聚合函数结果中返回一行    
7. Using index : 索引覆盖查询 (仅使用索引树中的信息从表中检索列信息，而不必执行额外的搜索以读取实际行。)  
