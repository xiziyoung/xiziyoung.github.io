---
title: mysql优化
categories:
- 数据库
date: 2018-04-08 11:10:47
tags:
- mysql
---

查询优化, 索引优化, 库表机构优化 需要齐头并进, 一个不落

# 查询优化和索引优化
- - -
多用DESC 或者 EXPLAIN分析查询情况;        
###  常规:     
1. 使用缓存, 缓存技术在很多时候能很好的解决数据库查询慢的问题.(redis等等).            
2. 不要使用select *，只返回需要的列，能够大大的节省数据传输量，与数据库的内存使用量;               
(以下示例为了简洁, 大量使用了select *, 实际项目中尽量避免)            
3. 排序优化:
1).一般的获取分页数据,数据集较小,文件排序也很快; 但是对于类似一次按照顺序导出大量数据的时候,数据集很大,文件排序就会很慢,得让排序用上索引;
order by在使用索引的时候同样要遵守最左前缀原则;
需要所有排序列的排序方向一致(都为正序或都为倒序);
连表查询时,order by的字段需要都是第一个表的才能命中索引;
2).避免limit 的 offset值过大.       
select * from test_table where sex = 'M' order by rating limit 1000000,10;              
无论如何优化索引,这个查询都将很慢,因为随着偏移量的增加,mysql需要扫描大量需要丢弃的数据;                
可以通过限制用户能够翻页的数据量来避免该问题,一般也没用户在意搜索结果的上千页;

 <font color=black>延迟关联 : </font>
优化这类索引的另一个比较好的策略是使用延迟关联，通过使用覆盖索引查询返回需要的主键，再根据这类逐渐关联原来获取得需要的行。这就可以减少MySQL扫描那些需要丢弃的行数。下面这个查询显示了如何高效的使用（sex rating）索引进行排序和分页：
```sql
mysql> SELECT <cols> FROM profiles INNER JOIN(
      ->           SELECT <primary key cols> FROM profiles
      ->           WHERE x.sex = 'M' ORDER BY rating LIMIT 100000,10
      ->  ) AS x USING(<primary key cols>);
上面语句优化的结果：
mysql> SELECT <cols> FROM profiles INNER JOIN(
      ->           SELECT id FROM profiles
      ->           WHERE x.sex = 'M' ORDER BY rating LIMIT 100000,10
      ->  ) AS x WHERE x.id = profiles.id;
```

3).如果不是因为每次要获取相同条目的数据使用分页limit和offset; 而是因为想分批次处理完一批数据集(避免一次加载到内存中),而使用limit和offset,可以使用between and来避免offset的性能问题;
例如: 给平台满足条件的用户(假如有很多,50万)发送消息;
常见的方式是脚本for循环用户表select <cols> from user_tab where ... limit 10000, 500;每次取500个满足条件的用户,直打取完;这样这个循环到了后面offset的值会非常的大,造成性能问题;
推荐:使用between and循环取区间数据集的方法 来代替上面的limit offset;具体实现如下:
要求用户id是自增的; 查询最小的用户id 和 最大的用户id, 每次批量处理一部分, 使用 between  and 来查找用户id;
```php
$minUserId = User::where('create_time', '<=', $endTime)->where('delete', 0)->min('user_id');
$maxUserId = User::where('create_time', '<=', $endTime)->where('delete', 0)->max('user_id');
$studyReportModel = new MonthlyStudyReport();
for ($start = $minUserId; $start <= $maxUserId; $start += $pageSize + 1) {
                $end = ($start + $pageSize) > $maxUserId ? $maxUserId : $start + $pageSize;
    
                $msg = 'GenerateMonthlyStudyReport本次处理的用户id区间起始: ' . $start . " 结束为" . $end;
                $this->output->info($msg);
                Log4dd::info($msg);
    
                $userIds = User::where('user_id', 'between', [$start, $end])->where('create_time', '<=', $endTime)->where('delete', 0)->column('user_id');
                !empty($userIds) && $this->sendMessage($userIds);//如果查到这个between and范围内的数据不为空,则对这些用户执行消息发送; 使用between and区间查询,因为后面还有where附加条件,所以这个区间里不满足条件的用户一样会被过滤掉;
                //虽然每次获取到的数据集条目不一定是500个(筛选后可能小于500), 但是并不影响我们最终给所有满足条件的用户发送了消息; 而且避开了limit offset过大的性能问题;
    
                $msg = 'GenerateMonthlyStudyReport该批次处理完';
                Log4dd::info($msg);
                $this->output->info($msg);         
}        
```
        
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
7. 防止sql注入, 类似laravel中如果写原生的raw查询等,参数使用绑定的方式,而不要直接写死到语句中;
8. 事务最小化原则      
9. 死锁的预防;
10. or查询转union
```sql
SELECT field1_index, field2_index FROM test_table WHERE field1_index = '1' OR field2_index = '1'; 
如果field1_index, field2_index上建有索引 这种or查询将导致使用不了索引,
可以修改为以下使用UNION:
SELECT field1_index, field2_index FROM test_table WHERE field1_index = '1' 
UNION
SELECT field1_index, field2_index FROM test_table WHERE field2_index = '1';
```
11. 连接查询时,大表转小表,然后再连接:

```sql
(1).where条件放在on内外的区别(特别是外连接查询时):
mysql> SELECT * FROM `t1`;
+----+------+---------------------+
| id | name | deleted_at          |
+----+------+---------------------+
|  1 | 小明 | 2019-08-27 15:40:30 |
|  2 | 小天 | NULL                |
+----+------+---------------------+
2 rows in set (0.05 sec)

mysql> SELECT * FROM `t2`;
+----+-----+---------------------+
| id | age | deleted_at          |
+----+-----+---------------------+
|  2 |  25 | 2019-08-27 15:41:41 |
+----+-----+---------------------+
1 row in set (0.04 sec)

mysql> SELECT * FROM t1 LEFT JOIN t2 ON t1.id = t2.id AND  t2.deleted_at IS NULL WHERE t1.deleted_at IS NULL;
+----+------+------------+------+------+------------+
| id | name | deleted_at | id   | age  | deleted_at |
+----+------+------------+------+------+------------+
|  2 | 小天 | NULL       | NULL | NULL | NULL       |
+----+------+------------+------+------+------------+
1 row in set (0.04 sec)
-- 条件跟在ON条件里面,先筛选t2表的记录，然后根据t1表返回t1表所有行

mysql> SELECT * FROM t1 LEFT JOIN t2 ON t1.id = t2.id  WHERE t1.deleted_at IS NULL AND  t2.deleted_at IS NULL;
Empty set
-- 条件在外面，先根据t1表返回所有记录，然后根据t2表条件删除记录


(2).利用关联表的where写在on里面可以减小关联表的数据量来优化内连接join查询:
select * from a json b on b.a_id = a.id where b.field1 = 100;
-- 可改为
select * from a json b on b.a_id = a.id and b.field1 = 100;

```

### 索引:      
索引对于良好的性能非常关键,尤其是在表数据量大时; 索引可以先简单理解为书的目录,可以帮你快速定位你想看的内容所在的页码;       
索引类型:PRIMARY key 主键, normal 一般、unique 唯一、full text 全文           

<font color=red>**特别说明:**</font>         
当查询需要访问大多数行时，顺序读取比通过索引更快。顺序读取可以最大限度地减少磁盘搜索，即使查询不需要所有行也是如此。      
mysql优化器会基于其他因素,如表大小、行数、I/O块大小来决定是使用索引还是表扫描.        
一般对于非常小的表 或者 对于大表的大部分数据量的查询(最佳索引跨越表的30%以上), 查询将进行全表扫描;       
```sql  
-- 例如你在user表中查询sex='男'的记录;sex列建有索引, 但sex='男'的记录占总数据量的50%时;
EXPLAIN SELECT * FROM `user` where `sex`='男';
-- 你将看到查询分析的 possible_keys=sex,但是type=ALL,key=Null , 这说明实际并没走索引,而是进行的全表扫描;
```
<font color=yellow>所以在测试下面索引的用法时: 如果发现没有按照你的预期,出现了全表扫描, 请先看下是否是小表或者大表查询超过数据量的30%导致的; </font>    

**索引优化常见注意事项:**  
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
    
5. B-Tree索引中,复合索引(多列索引)**最左前缀**原则:           
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

6. 对于范围条件查询,Mysql无法再使用范围列后面的其他索引了,但对于"多个等值条件查询"则没有这个限制;
```sql
CREATE TABLE `user` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `age` tinyint(4) NOT NULL,
  `work` varchar(255) NOT NULL,
  PRIMARY KEY (`id`),
  KEY `age_work` (`age`,`work`) USING BTREE
) ENGINE=InnoDB AUTO_INCREMENT=0 DEFAULT CHARSET=utf8;

mysql> EXPLAIN SELECT * FROM `user` where age in (70,75) and `work`='DOCTOR';
+----+-------------+-------+------------+-------+---------------+----------+---------+------+------+----------+-----------------------+
| id | select_type | table | partitions | type  | possible_keys | key      | key_len | ref  | rows | filtered | Extra                 |
+----+-------------+-------+------------+-------+---------------+----------+---------+------+------+----------+-----------------------+
|  1 | SIMPLE      | user  | NULL       | range | age_work      | age_work | 768     | NULL |    2 |      100 | Using index condition |
+----+-------------+-------+------------+-------+---------------+----------+---------+------+------+----------+-----------------------+
1 row in set
mysql> 
mysql> EXPLAIN SELECT * FROM `user` where age >=70 and `work`='DOCTOR';
+----+-------------+-------+------------+-------+---------------+----------+---------+------+------+----------+-----------------------+
| id | select_type | table | partitions | type  | possible_keys | key      | key_len | ref  | rows | filtered | Extra                 |
+----+-------------+-------+------------+-------+---------------+----------+---------+------+------+----------+-----------------------+
|  1 | SIMPLE      | user  | NULL       | range | age_work      | age_work | 768     | NULL |   12 |       10 | Using index condition |
+----+-------------+-------+------------+-------+---------------+----------+---------+------+------+----------+-----------------------+
1 row in set
通过查看filtered是否为100%来判断多列索引的利用情况
```

7. 索引列的顺序           
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
 
 
8. 强制类型转换会全表扫描      
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
 
9. 覆盖查询:        
仅使用索引树中的信息从表中检索列信息，而不必执行额外的搜索以读取实际行。        
即索引包含所有需要查询的字段的值。       
具有以下优点：     
* 索引通常远小于数据行的大小，只读取索引能大大减少数据访问量。        
* 一些存储引擎（例如 MyISAM）在内存中只缓存索引，而数据依赖于操作系统来缓存。因此，只访问索引可以不使用系统调用（通常比较费时）。       
* 对于 InnoDB 引擎，若辅助索引能够覆盖查询，则无需访问主索引。            


10. 一个索引的题目:
```sql
CREATE TABLE `t` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `x` int(11) NOT NULL,
  `y` int(11) NOT NULL,
  `z` int(11) NOT NULL,
  `w` int(11) NOT NULL,
  PRIMARY KEY (`id`),
  KEY `x_y_z` (`x`,`y`,`z`)
) ENGINE=InnoDB AUTO_INCREMENT=2038 DEFAULT CHARSET=utf8;
-- 题目: 数据表如上,  下面哪些查询能用到索引x_y_z
--(1) select * from t where where x=2002 and y=22 and z = 20; --用到索引x_y_z, 整个全用 

--(2) select * from t where where x=2002 and y=22 and z like '%2'; --只使用到了索引x_y部分 

--(3) select * from t where where x=2002 and y=22 and z like '2%'; --只使用到了索引x_y部分; 如果z字段为字符串类型(如varchar),则整个x_y_z都可以使用上; 

--(4) select * from t where where x>=2002 and y=22 and z = 20; --只使用到了索引x部分 

--(5) select * from t where where x=2002 and y>22 and z = 20; --只使用到了索引x_y部分 

--(6) select * from t where where x=2002 and y=22 order by z; --用到索引x_y_z, 整个全用 
mysql> EXPLAIN SELECT * FROM `t` where x=2002 and y=22 order by z;
+----+-------------+-------+------------+------+---------------+-------+---------+-------------+------+----------+-----------------------+
| id | select_type | table | partitions | type | possible_keys | key   | key_len | ref         | rows | filtered | Extra                 |
+----+-------------+-------+------------+------+---------------+-------+---------+-------------+------+----------+-----------------------+
|  1 | SIMPLE      | t     | NULL       | ref  | x_y_z         | x_y_z | 8       | const,const |   11 |   100.00 | Using index condition |
+----+-------------+-------+------------+------+---------------+-------+---------+-------------+------+----------+-----------------------+
1 row in set (0.05 sec)

--(7) select * from t where where x=2002 order by y,z; -- 用到索引x_y_z, 整个全用 
mysql> EXPLAIN SELECT * FROM `t` where x=2002 order by y,z;
+----+-------------+-------+------------+------+---------------+-------+---------+-------+------+----------+-----------------------+
| id | select_type | table | partitions | type | possible_keys | key   | key_len | ref   | rows | filtered | Extra                 |
+----+-------------+-------+------------+------+---------------+-------+---------+-------+------+----------+-----------------------+
|  1 | SIMPLE      | t     | NULL       | ref  | x_y_z         | x_y_z | 4       | const |   72 |   100.00 | Using index condition |
+----+-------------+-------+------------+------+---------------+-------+---------+-------+------+----------+-----------------------+
1 row in set (0.04 sec)

--(8) select * from t where where x=2002 order by z,y; -- 只使用到了索引x部分; 因为排序中的顺序是z,y没有遵守左前缀 
mysql> EXPLAIN SELECT * FROM `t` where x=2002 order by z,y;
+----+-------------+-------+------------+------+---------------+-------+---------+-------+------+----------+---------------------------------------+
| id | select_type | table | partitions | type | possible_keys | key   | key_len | ref   | rows | filtered | Extra                                 |
+----+-------------+-------+------------+------+---------------+-------+---------+-------+------+----------+---------------------------------------+
|  1 | SIMPLE      | t     | NULL       | ref  | x_y_z         | x_y_z | 4       | const |   72 |   100.00 | Using index condition; Using filesort |
+----+-------------+-------+------------+------+---------------+-------+---------+-------+------+----------+---------------------------------------+
1 row in set (0.06 sec)
-- 可以看到 Extra  中出现了 Using filesort

--(9) select * from t where where x=2002 order by y asc, z desc; -- 只使用到了索引x_y部分; 因为排序中的y,z反向不一致, 一个正序, 一个倒序;
mysql> explain select * from t where x=2002 order by y asc;
+----+-------------+-------+------------+------+---------------+-------+---------+-------+------+----------+-----------------------+
| id | select_type | table | partitions | type | possible_keys | key   | key_len | ref   | rows | filtered | Extra                 |
+----+-------------+-------+------------+------+---------------+-------+---------+-------+------+----------+-----------------------+
|  1 | SIMPLE      | t     | NULL       | ref  | x_y_z         | x_y_z | 4       | const |   72 |   100.00 | Using index condition |
+----+-------------+-------+------------+------+---------------+-------+---------+-------+------+----------+-----------------------+
1 row in set (0.08 sec)
-- 
mysql> explain select * from t where x=2002 order by y asc, z desc;
+----+-------------+-------+------------+------+---------------+-------+---------+-------+------+----------+---------------------------------------+
| id | select_type | table | partitions | type | possible_keys | key   | key_len | ref   | rows | filtered | Extra                                 |
+----+-------------+-------+------------+------+---------------+-------+---------+-------+------+----------+---------------------------------------+
|  1 | SIMPLE      | t     | NULL       | ref  | x_y_z         | x_y_z | 4       | const |   72 |   100.00 | Using index condition; Using filesort |
+----+-------------+-------+------------+------+---------------+-------+---------+-------+------+----------+---------------------------------------+
1 row in set (0.08 sec)
-- 可以看到 Extra  中出现了 Using filesort

--(10) select * from t where where x=2002 and y=22 order by z,w; --只使用到索引x_y部分; 因为排序中还有额外的一列w不在索引中;
mysql> EXPLAIN SELECT * FROM `t` where x=2002 and y=22 order by z,w;
+----+-------------+-------+------------+------+---------------+-------+---------+-------------+------+----------+---------------------------------------+
| id | select_type | table | partitions | type | possible_keys | key   | key_len | ref         | rows | filtered | Extra                                 |
+----+-------------+-------+------------+------+---------------+-------+---------+-------------+------+----------+---------------------------------------+
|  1 | SIMPLE      | t     | NULL       | ref  | x_y_z         | x_y_z | 8       | const,const |   11 |   100.00 | Using index condition; Using filesort |
+----+-------------+-------+------------+------+---------------+-------+---------+-------------+------+----------+---------------------------------------+
1 row in set (0.04 sec)
-- 可以看到 Extra  中出现了 Using filesort

--(11) EXPLAIN SELECT z FROM `t` where x=2002 and y=22 group by z;  --用到索引x_y_z, 整个全用 
mysql> EXPLAIN SELECT z FROM `t` where x=2002 and y=22 group by z;
+----+-------------+-------+------------+------+---------------+-------+---------+-------------+------+----------+--------------------------+
| id | select_type | table | partitions | type | possible_keys | key   | key_len | ref         | rows | filtered | Extra                    |
+----+-------------+-------+------------+------+---------------+-------+---------+-------------+------+----------+--------------------------+
|  1 | SIMPLE      | t     | NULL       | ref  | x_y_z         | x_y_z | 8       | const,const |   11 |   100.00 | Using where; Using index |
+----+-------------+-------+------------+------+---------------+-------+---------+-------------+------+----------+--------------------------+
1 row in set (0.07 sec)

--(12) EXPLAIN SELECT z FROM `t` where x=2002 and y=22 group by z;  --用到索引x_y_z, 整个全用 
mysql> EXPLAIN SELECT z FROM `t` where x=2002 and y=22 group by z;
+----+-------------+-------+------------+------+---------------+-------+---------+-------------+------+----------+--------------------------+
| id | select_type | table | partitions | type | possible_keys | key   | key_len | ref         | rows | filtered | Extra                    |
+----+-------------+-------+------------+------+---------------+-------+---------+-------------+------+----------+--------------------------+
|  1 | SIMPLE      | t     | NULL       | ref  | x_y_z         | x_y_z | 8       | const,const |   11 |   100.00 | Using where; Using index |
+----+-------------+-------+------------+------+---------------+-------+---------+-------------+------+----------+--------------------------+
1 row in set (0.07 sec)

--(13) SELECT z FROM `t` where x=2002 and y>22 group by z;  -- 只用到了索引x_y部分
mysql> EXPLAIN SELECT z FROM `t` where x=2002 and y>22 group by z; 
+----+-------------+-------+------------+-------+---------------+-------+---------+------+------+----------+-------------------------------------------+
| id | select_type | table | partitions | type  | possible_keys | key   | key_len | ref  | rows | filtered | Extra                                     |
+----+-------------+-------+------------+-------+---------------+-------+---------+------+------+----------+-------------------------------------------+
|  1 | SIMPLE      | t     | NULL       | range | x_y_z         | x_y_z | 8       | NULL |    8 |   100.00 | Using where; Using index; Using temporary |
+----+-------------+-------+------------+-------+---------------+-------+---------+------+------+----------+-------------------------------------------+
1 row in set (0.08 sec)
-- 可以看到 Extra  中出现了 Using temporary
```
 
 
 
        
        
# 库表结构设计优化及其他
- - -
### 选择合适的存储引擎:
建表时就考虑选择存储引擎，如果是新闻之类的数据，由于插入和查询需求大，就选择myasim  (Myisam：表锁，全文索引) ; 如果是电商中的数据,由于要用到事务和行锁,选择innodb  ( innodb行(记录)锁，事务（回滚），外键.  事务安全型存储引擎，更加注重数据的完整性和安全性。)

Memory：内存存储引擎，速度快、数据容易丢失

### 表结构设计:
使用最小的字段通常更好,例如:固定长度的可以用char，类似md5()的密码;
避免null;
记得默认值,commit; 
建立恰当的索引;
id自增，AUTO_INCREMENT
标记类型的type/is_delete等可以用tinyint；
bitmap;


### 缓存
mysql服务自身提供的有缓存系统,可以开启缓存;       
但建议使用redis/memcached        
### 分区       
基本概念，把一个表，从逻辑上分成多个区域，便于存储数据。        
采用分区的前提，数据量非常大。         
如果数据表的记录非常多，比如达到上亿条，数据表的活性就大大降低，数据表的运行速度就比较慢、效率低下，影响mysql数据库的整体性能，就可以采用分区解决，分区是mysql本身就支持的技术    
### 分表
水平拆分：是把一个表的全部记录信息分别存储到不同的分表之中。            
程序需要注意的是从哪个表读入,向哪个表更新;       

垂直拆分：是把一个表的全部字段分别存储到不同的表里边。             

分表方式: 方式一: 常规的关联id取模100来分表, 改方式较老旧, 不在一个子表的关联聚合等查询比较困难;
         方式二: 使用mycat等工具;

### 清理数据碎片       
### 读写分离 
