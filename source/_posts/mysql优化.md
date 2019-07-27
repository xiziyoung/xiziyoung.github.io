---
title: mysql优化
categories:
- 数据库
date: 2018-04-08 11:10:47
tags:
- mysql
---
# 查询优化
- - -
多用DESC 或者 EXPLAIN分析查询情况;        
##  常规:     
1. 使用缓存, 缓存技术在很多时候能很好的解决数据库查询慢的问题            
2. 不要使用select *，只返回需要的列，能够大大的节省数据传输量，与数据库的内存使用量;               
(以下示例为了简洁, 大量使用了select *, 实际项目中尽量避免)            
3. 避免limit 的 offset值过大      
select * from test_table where sex = 'M' limit 1000000,10;              
无论如何优化索引,这个查询都将很慢,因为随着便宜量的增加,mysql需要扫描大量需要丢弃的数据;                
可以通过限制用户能够翻页的数据量来避免该问题,一般也没用户在意搜索结果的上千页;         
4. 允许为null的列，查询有潜在大坑:           
SELECT * FROM `test_table` where name != 'tom';该查询结果集中将不包括name is null 的情况;             
建议:设计表是字段尽量用not null约束以及给默认值。           
5. 如果明确知道只有一条结果返回，limit 1能够提高效率             
例如你里系统用户不允许一个邮箱注册多个账号,即一个邮箱在用户表里只会有一条有效记录.          
select * from user where email='123456@qq.com';             
可以优化为：          
select * from user where email='123456@qq.com' limit 1; 
原因：     
你知道只有一条结果，但数据库并不知道，明确告诉它，让它主动停止游标移动             
6. 分解大连接查询      
连表查询虽然能让我们更方便的拿到想要的数据, 但应该避免多个大表的连接查询, 连表过多会导致查询慢(特表是很多大表连接,且无法很好的用索引优化到查询时);           
可以先查出主要的数据结果集, 然后利用结果集中的关联字段去对应表中查询到关联结果集,再用程序将关联结果集聚合到总的数据结果集中;            
或者先查出小表的数据集,然后用这个数据集join大表;     



## 索引:
索引对于良好的性能非常关键,尤其是在表数据量大时; 索引可以先简单理解为书的目录,可以帮你快速定位你想看的内容所在的页码;       
索引类型:PRIMARY key 主键, normal 一般、unique 唯一、full text 全文           

特别说明:       
当查询需要访问大多数行时，顺序读取比通过索引更快。顺序读取可以最大限度地减少磁盘搜索，即使查询不需要所有行也是如此。      
mysql优化器会基于其他因素,如表大小、行数、I/O块大小来决定是使用索引还是表扫描.        
一般对于非常小的表 或者 对于大表的大多数数据量的查询(最佳索引是否跨越表的30%以上), 查询时将进行全表扫描;       
```sql  
-- 例如你在user表中查询sex='男'的记录;sex列建有索引, 但sex='男'的记录占总数据量的50%时;
EXPLAIN SELECT * FROM `user` where `sex`='男';
-- 你将看到查询分析的 possible_keys=sex,但是type=ALL,key=Null , 这说明实际并没走索引,而是进行的全表扫描;
```

1. 查询经常用到的高频字段,用来连表的关联字段 最好建立上索引;       
2. 前导模糊查询不能命中索引;        
例如: select * from test_table where name like '%XX'; 无法使用索引      
而非前导模糊查询则可以：select * from test_table where name like 'XX%';     
3. 负向条件查询不能命中索引     
select * from test_table where age != 30;       
包括 not in等;     
not in/not exists都不是好习惯;(not exists比not in 好 , not exists可以走索引,避免不了负向查询就尽量用exists或者not exists);         
4. 在进行查询时，索引列不能是表达式的一部分，也不能是函数的参数，否则无法使用索引。     
explain select * from test_table where YEAR(date) < '2019';     
即使date列有索引, 改查询也将无法命中索引;        
类似的还有下面的这种神操作:      
EXPLAIN SELECT * FROM `user` where age+1 = 49; 无法命中age索引;       
233333
计算尽量放到业务层完成,而非数据库层      
5. B-Tree索引中,复合索引(多列索引)最左匹配原则:          
(说明:该项适用与B-Tree索引;哈希和其他类型的索引并不会像B-Tree一样按照顺序存储)     
即索引key(field1, field2, field3),要想很好利用该索引,需要你的查询语句能使用到靠左的字段,将索引中从左起到右的字段 部分/全部 使用到查询语句中;         
例如:         
```sql
CREATE TABLE `user` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `name` varchar(255) NOT NULL,
  `gender` tinyint(4) NOT NULL DEFAULT '1' COMMENT '1代表男,2代表女',
  `age` tinyint(4) NOT NULL,
  `country` varchar(255) NOT NULL,
  `work` varchar(255) NOT NULL,
  PRIMARY KEY (`id`),
  KEY `user_composite_index` (`country`,`work`,`age`) USING BTREE
) ENGINE=InnoDB AUTO_INCREMENT=7 DEFAULT CHARSET=utf8;

-- (1).可以很好用到索引user_composite_index的情况
EXPLAIN SELECT * FROM `user` WHERE `country`='EN';
EXPLAIN SELECT * FROM `user` WHERE `country`='EN' AND `work`='IT';
EXPLAIN SELECT * FROM `user` WHERE `country`='EN' AND `work`='IT' ORDER BY `age` desc;
   -- sql语句中的写法顺序好像没有关系,如下也可
   EXPLAIN SELECT * FROM `user` WHERE age=50 AND `country`='EN' AND `work`='IT';

-- (2).完全无法使用索引user_composite_index的情况
EXPLAIN SELECT * FROM `user` WHERE `work`='IT';
EXPLAIN SELECT * FROM `user` WHERE `age` = 50;
EXPLAIN SELECT * FROM `user` WHERE `work`='IT' ORDER BY `age` desc;
EXPLAIN SELECT * FROM `user` WHERE `country`!='EN' AND `work`='IT' ORDER BY `age` desc;


-- (3).部分使用到索引的情况
explain  SELECT * FROM `user` WHERE `country`='EN' AND  `age`=50;

explain  SELECT * FROM `user` WHERE `country`='EN' AND `work`!='IT' AND `age`=50;
+----+-------------+-------+------------+-------+----------------------+----------------------+---------+------+------+----------+-----------------------+
| id | select_type | table | partitions | type  | possible_keys        | key                  | key_len | ref  | rows | filtered | Extra                 |
+----+-------------+-------+------------+-------+----------------------+----------------------+---------+------+------+----------+-----------------------+
|  1 | SIMPLE      | user  | NULL       | range | user_composite_index | user_composite_index | 1534    | NULL |    2 |    14.29 | Using index condition |
+----+-------------+-------+------------+-------+----------------------+----------------------+---------+------+------+----------+-----------------------+
通过查看执行计划filtered,这个字段表示存储引擎返回的数据在server层过滤后，剩下多少满足查询的记录数量的比例，注意是百分比，不是具体记录数。      
此处不为100%;     
因为`work`使用了不等于,负向查询导致`work`没使用到索引; 后面的`age`因为在其左的`work`没命中索引,也将无法命中索引.      

```
所以，在创建复合索引时，要根据业务需求，where子句中查询使用最频繁的一列放在索引最左边。      

6. 索引列的顺序           
让选择性最强的索引列放在前面。         
索引的选择性是指：不重复的索引值和记录总数的比值。最大值为 1，此时每个记录都有唯一的索引与其对应。选择性越高，每个记录的区分度越高，查询效率也越高。     
例如下面显示的结果中 customer_id 的选择性比 staff_id 更高，因此最好把 customer_id 列放在多列索引的前面。      
```sql
SELECT COUNT(DISTINCT staff_id)/COUNT(*) AS staff_id_selectivity,
COUNT(DISTINCT customer_id)/COUNT(*) AS customer_id_selectivity,
COUNT(*)
FROM payment;

-- 结果:
  staff_id_selectivity: 0.0001
customer_id_selectivity: 0.0373
               COUNT(*): 16049
```
 
 
7. 强制类型转换会全表扫描      
 例如:        
 ```sql
 -- 商品表中serial_number 类型为varchar(15),该列添加的有索引
 CREATE TABLE `goods` (
   `id` int(11) NOT NULL AUTO_INCREMENT,
   `name` varchar(255) NOT NULL,
   `serial_number` varchar(15) NOT NULL,
   PRIMARY KEY (`id`),
   KEY `serial_number` (`serial_number`)
 ) ENGINE=InnoDB AUTO_INCREMENT=5 DEFAULT CHARSET=utf8;
 
 -- 使用字符串时,查询使用到了索引
 mysql> desc SELECT * FROM `goods` where serial_number='123456';
 +----+-------------+-------+------------+------+---------------+---------------+---------+-------+------+----------+-------+
 | id | select_type | table | partitions | type | possible_keys | key           | key_len | ref   | rows | filtered | Extra |
 +----+-------------+-------+------------+------+---------------+---------------+---------+-------+------+----------+-------+
 |  1 | SIMPLE      | goods | NULL       | ref  | serial_number | serial_number | 47      | const |    1 |      100 | NULL  |
 +----+-------------+-------+------------+------+---------------+---------------+---------+-------+------+----------+-------+
 1 row in set
 
 -- 使用int类型去强制转换时,查询未使用到索引, type为all,说明全表扫描了
 mysql> 
 mysql> desc SELECT * FROM `goods` where serial_number=123456;
 +----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+-------------+
 | id | select_type | table | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra       |
 +----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+-------------+
 |  1 | SIMPLE      | goods | NULL       | ALL  | serial_number | NULL | NULL    | NULL |    4 |       25 | Using where |
 +----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+-------------+
 1 row in set
 
 mysql> 
 ```
 
8. 覆盖查询:        
仅使用索引树中的信息从表中检索列信息，而不必执行额外的搜索以读取实际行。        
即索引包含所有需要查询的字段的值。       
具有以下优点：     
* 索引通常远小于数据行的大小，只读取索引能大大减少数据访问量。        
* 一些存储引擎（例如 MyISAM）在内存中只缓存索引，而数据依赖于操作系统来缓存。因此，只访问索引可以不使用系统调用（通常比较费时）。       
* 对于 InnoDB 引擎，若辅助索引能够覆盖查询，则无需访问主索引。        
    
 
# 设计优化
- - -
## 缓存
mysql服务自身提供的有缓存系统,可以开启缓存;       
但建议使用redis/memcached        
## 分区       
基本概念，把一个表，从逻辑上分成多个区域，便于存储数据。        
采用分区的前提，数据量非常大。         
如果数据表的记录非常多，比如达到上亿条，数据表的活性就大大降低，数据表的运行速度就比较慢、效率低下，影响mysql数据库的整体性能，就可以采用分区解决，分区是mysql本身就支持的技术    
## 分表
水平拆分：是把一个表的全部记录信息分别存储到不同的分表之中。            
程序需要注意的是从哪个表读入,向哪个表更新;       

垂直拆分：是把一个表的全部字段分别存储到不同的表里边。             

## 清理数据碎片       
## 读写分离 
## 表结构设计    
