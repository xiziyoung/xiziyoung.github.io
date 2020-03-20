---
title: mysql使用
categories:
- 数据库
date: 2017-8-10 12:45:21
tags:
- mysql
---

**开篇**:
mysql的书写顺序:
```sql
select *columns* from *tables*
    where *predicae1*
    group by *columns*
    having *predicae1*
    order by *columns*
    limit *start*, *offset*;
```
但是mysql的查询执行顺序并不是按照书写顺序来的,而是按照如下的执行顺序:
```sql
from *tables*
where *predicae1*
group by *columns*
having *predicae1*
select *columns*
order by *columns*
limit *start*, *offset*;
```


## 1.多个值的 IN 匹配     
语法: (字段1, 字段2)  IN ( (字段1的结果1, 字段2的结果1), (字段1的结果2, 字段2的结果2) )   
语法示例:  select * from  table  where  (field1, field2) in ( (field1_value,field2_value) , (field1_value, field2_value), (field1_value, field2_value) );
完整举例:   
```sql
-- ----------------------------
-- Table structure for Employee
-- ----------------------------
DROP TABLE IF EXISTS `Employee`;
CREATE TABLE `Employee` (
  `Id` int(11) NOT NULL AUTO_INCREMENT,
  `Name` varchar(255) NOT NULL,
  `Salary` int(11) NOT NULL,
  `DepartmentId` int(11) NOT NULL,
  PRIMARY KEY (`Id`)
) ENGINE=InnoDB AUTO_INCREMENT=0 DEFAULT CHARSET=utf8;

-- ----------------------------
-- Records of Employee
-- ----------------------------
INSERT INTO `Employee` VALUES ('1', 'Joe', '70000', '1');
INSERT INTO `Employee` VALUES ('2', 'Henry', '80000', '2');
INSERT INTO `Employee` VALUES ('3', 'Sam', '60000', '2');
INSERT INTO `Employee` VALUES ('4', 'Max', '90000', '1');
INSERT INTO `Employee` VALUES ('5', 'Tom', '90000', '1');


-- 查询语句:
SELECT `Name` , `Salary`, `DepartmentId` FROM `Employee` where (Salary,DepartmentId) in((90000,1), (80000,2));
-- 等价于: 
SELECT `Name` , `Salary`, `DepartmentId` FROM `Employee` 
where (`DepartmentId`, `Salary`) in (SELECT `DepartmentId`,MAX(`Salary`) as Salary FROM `Employee` GROUP BY `DepartmentId`);

```

## 2.字段加密
常见的:    
SELECT MD5('123');  
SELECT SHA('123');  
SELECT PASSWORD('123');     
MD5()：计算字符串的MD5校验和      
SHA：计算字符串的SHA校验和    
PASSWORD()：创建一个经过加密的密码字符串，适合于插入到MySQL的安全系  
统。该加密过程不可逆，和unix密码加密过程使用不同的算法。主要用于MySQL的认证系统。   
加密函数非常之多,官网:https://dev.mysql.com/doc/refman/8.0/en/encryption-functions.html   

### 2.1 AES_ENCRYPT 和 AES_DECRYPT (推荐)
使用AES算法解密/加密    
其加密的结果最好使用blob类型存储; 
语法: AES_ENCRYPT('要加密的值', 'token key')       
      AES_DECRYPT('被加密的字段', 'token key')        
```sql
CREATE TABLE `encrypt` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `email` varchar(255) NOT NULL,
  `email_e` varbinary(255) DEFAULT NULL,
  `phone` varchar(255) NOT NULL,
  `phone_e` varbinary(255) DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB AUTO_INCREMENT=0 DEFAULT CHARSET=utf8;

-- 首先设置用来加密的key值, 该变量在以下语句中使用
SET @key = 'D7DF5B64DF1181EF1D62D646A13AA860';

-- (1)
-- 添加完整的数据, email_e和phone_e使用AES_ENCRYPT('value',key)加密
INSERT INTO `encrypt` (id,email,email_e,phone,phone_e) VALUES (1,'123456@qq.com', AES_ENCRYPT('123456@qq.com', @key), '12345678901', AES_ENCRYPT('12345678901', @key));
-- 使用 AES_DECRYPT(field, key)来解密,查询   
SELECT *,AES_DECRYPT(email_e, @key) as eamil_value,AES_DECRYPT(phone_e, @key) as phone_value FROM `encrypt` where id=1;
-- 查询结果   
+----+---------------+----------------+-------------+----------------+---------------+-------------+
| id | email         | email_e        | phone       | phone_e        | eamil_value   | phone_value |
+----+---------------+----------------+-------------+----------------+---------------+-------------+
|  1 | 123456@qq.com |    一些乱码     | 12345678901 |    一些乱码     | 123456@qq.com | 12345678901 |
+----+---------------+----------------+-------------+----------------+---------------+-------------+

-- (2)
-- 你还可以对key再次的转变一次, 例如使用UNHEX, 即AES_ENCRYPT('value',UNHEX(key))加密
INSERT INTO `encrypt` (id,email,phone) VALUES (2,'test@qq.com', '55555555555');
UPDATE `encrypt` set  `email_e`= AES_ENCRYPT(`email`, UNHEX(@key)),`phone_e`= AES_ENCRYPT(`phone`, UNHEX(@key)) WHERE id =2;
-- 对应的查询
SELECT *,AES_DECRYPT(email_e, UNHEX(@key)) as eamil_value,AES_DECRYPT(phone_e, UNHEX(@key)) as phone_value FROM `encrypt` where id=2;
-- 查询结果
+----+-------------+-----------------+-------------+----------------+-------------+-------------+
| id | email       | email_e         | phone       | phone_e        | eamil_value | phone_value |
+----+-------------+-----------------+-------------+----------------+-------------+-------------+
|  2 | test@qq.com | 一些乱码         | 55555555555 |    一些乱码     | test@qq.com | 55555555555 |
+----+-------------+-----------------+-------------+----------------+-------------+-------------+

```
注: 上面的sql查询中的email_e和phone_e原来的乱码会影响到本博客hexo-next的搜索功能加载不了,故用正常字符描述...      
**tips:** php 的composer库 phpseclib/phpseclib 可以用来处理PKCS＃1（v2.1）RSA，DES，3DES，RC4，Rijndael，AES，Blowfish，Twofish，SSH-1，SSH-2， SFTP和X.509           
故mysql的AES_ENCRYPT可以和php的这个库结合使用              

### 2.2 ENCODE() 和 DECODE()     
ENCODE('需要编码字符串', 'token key')          
DECODE('被编码的字段', 'token key')       
该函数有两个参数：被加密或解密的字符串和作为加密或解密基础的密钥。   
Encode结果是一个二进制字符串，以BLOB类型存储。加密程度相对比较弱   
```sql
-- 表结构同上
SET @key = 'D7DF5B64DF1181EF1D62D646A13AA860';

INSERT INTO `encrypt` (id,email,email_e,phone,phone_e) VALUES (5,'new@qq.com', ENCODE('new@qq.com', @key), '000000', ENCODE('000000', @key));

SELECT *,DECODE(email_e, @key) as eamil_value,DECODE(phone_e, @key) as phone_value FROM `encrypt` where id=5;
```

## 3.case when的使用   
### 3.1 case when 用于查询: 
这里介绍三种使用方法: 
```sql
-- ----------------------------
-- Table structure for user
-- ----------------------------
DROP TABLE IF EXISTS `user`;
CREATE TABLE `user` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `name` varchar(255) NOT NULL,
  `gender` tinyint(4) NOT NULL DEFAULT '1' COMMENT '1代表男,2代表女',
  `age` tinyint(4) NOT NULL,
  `country` varchar(255) NOT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB AUTO_INCREMENT=7 DEFAULT CHARSET=utf8;

-- ----------------------------
-- Records of user
-- ----------------------------
INSERT INTO `user` VALUES ('1', '小明', '1', '20', '中国');
INSERT INTO `user` VALUES ('2', 'Tom', '1', '18', 'USA');
INSERT INTO `user` VALUES ('3', 'Sam', '2', '50', 'USA');
INSERT INTO `user` VALUES ('4', '小华', '2', '22', '中国');
INSERT INTO `user` VALUES ('5', '老张', '1', '70', '中国');
INSERT INTO `user` VALUES ('6', 'Joe', '1', '75', 'USA');

-- (1). CASE 字段1 WHEN 具体的值(不可为某个范围匹配) THEN 'value' ELSE 'valus' AS 字段别名,
--      CASE 字段2 WHEN 具体的值 THEN 'value' ELSE 'value' AS 字段别名,
SELECT *,
CASE `country` WHEN '中国' THEN '龙的传入' ELSE '外国人'  END AS '背景',
CASE `gender` WHEN '1' THEN '男' ELSE '女'  END AS '性别'
FROM `user`;
-- query result:
+----+------+--------+-----+---------+----------+------+
| id | name | gender | age | country | 背景     | 性别 |
+----+------+--------+-----+---------+----------+------+
|  1 | 小明 |      1 |  20 | 中国    | 龙的传入 | 男   |
|  2 | Tom  |      1 |  18 | USA     | 外国人   | 男   |
|  3 | Sam  |      2 |  50 | USA     | 外国人   | 女   |
|  4 | 小华 |      2 |  22 | 中国    | 龙的传入 | 女   |
|  5 | 老张 |      1 |  70 | 中国    | 龙的传入 | 男   |
|  6 | Joe  |      1 |  75 | USA     | 外国人   | 男   |
+----+------+--------+-----+---------+----------+------+


-- (2).CASE WHEN 单个字段  这个可以对字段进行取范围,也可具体值
SELECT *,  
CASE WHEN `age` < 30 THEN '青年'
WHEN `age` BETWEEN 30 and 50 THEN '中年' 
WHEN `age` = 70 THEN '古稀' 
ELSE '老年'  END AS '年龄'
FROM `user`;
-- query result:
+----+------+--------+-----+---------+------+
| id | name | gender | age | country | 年龄 |
+----+------+--------+-----+---------+------+
|  1 | 小明 |      1 |  20 | 中国    | 青年 |
|  2 | Tom  |      1 |  18 | USA     | 青年 |
|  3 | Sam  |      2 |  50 | USA     | 中年 |
|  4 | 小华 |      2 |  22 | 中国    | 青年 |
|  5 | 老张 |      1 |  70 | 中国    | 古稀 |
|  6 | Joe  |      1 |  75 | USA     | 老年 |
+----+------+--------+-----+---------+------+

-- (3)CASE WHEN 字段1,字段2，可以对多个字段进行替换
SELECT *,  
CASE WHEN `age` < 30 THEN '青年'
WHEN `country` = '中国' THEN '龙的传入' 
ELSE '未知'  END AS '备注'
FROM `user`;

-- query result:
+----+------+--------+-----+---------+----------+
| id | name | gender | age | country | 备注     |
+----+------+--------+-----+---------+----------+
|  1 | 小明 |      1 |  20 | 中国    | 青年     |
|  2 | Tom  |      1 |  18 | USA     | 青年     |
|  3 | Sam  |      2 |  50 | USA     | 未知     |
|  4 | 小华 |      2 |  22 | 中国    | 青年     |
|  5 | 老张 |      1 |  70 | 中国    | 龙的传入 |
|  6 | Joe  |      1 |  75 | USA     | 未知     |
+----+------+--------+-----+---------+----------+

```

### 3.2 case when 用于更新:
```sql
-- (1).单个字段的更新示例:
UPDATE `categories` SET
    `display_order` = CASE `id`
        WHEN 1 THEN 3
        WHEN 2 THEN 4
        WHEN 3 THEN 5
        ELSE 0
    END
WHERE `id` IN (1,2,3)

-- (2).多个字段的更新示例:
UPDATE `categories` SET `author` = '小明', `update_time` = '2017-10-10',
    `display_order` = CASE `id`
        WHEN 1 THEN 3
        WHEN 2 THEN 4
        WHEN 3 THEN 5
        ELSE 0
    END,
    `title` = CASE `id`
        WHEN 1 THEN 'New Title 1'
        WHEN 2 THEN 'New Title 2'
        WHEN 3 THEN 'New Title 3'
        ELSE 'null'
    END
WHERE `id` IN (1,2,3)
```
提醒: 在使用批量更新, 批量插入等方式的时候, 需要注意最终执行的sql语句的长度, 一般sql语句的长度默认最大 1M , 可通过mysql配置文件my.ini修改 max_allowed_packet(不推荐);       
建议: 在批量插入/更新时, 若担心语句过程, 可以将要添加或者更新的数据 分成 多个批次执行; 例如插入10000条数据, 可以每次插入2000条, 分五次执行;      


## 4.给查询结果增加递增的序号列  
语法:     
select (@rowNO := @rowNo+1) AS rowno,其他字段 from (select @rowNO:=0) as temp, `查询的表`....       
关于 := 说明        
```text
Unlike =, the := operator is never interpreted as a comparison operator. This means you can use := in any valid SQL statement (not just in SET statements) to assign a value to a variable.

``` 
参考: https://dev.mysql.com/doc/refman/8.0/en/assignment-operators.html#operator_assign-value     
### 4.1 示例:     
```sql
-- ----------------------------
-- Table structure for Scores
-- ----------------------------
DROP TABLE IF EXISTS `Scores`;
CREATE TABLE `Scores` (
  `Id` int(11) NOT NULL AUTO_INCREMENT,
  `Score` float(5,2) NOT NULL,
  PRIMARY KEY (`Id`)
) ENGINE=InnoDB AUTO_INCREMENT=0 DEFAULT CHARSET=utf8;

-- ----------------------------
-- Records of Scores
-- ----------------------------
INSERT INTO `Scores` VALUES ('1', '3.50');
INSERT INTO `Scores` VALUES ('2', '3.65');
INSERT INTO `Scores` VALUES ('3', '4.00');
INSERT INTO `Scores` VALUES ('4', '3.85');
INSERT INTO `Scores` VALUES ('5', '4.00');
INSERT INTO `Scores` VALUES ('6', '3.65');
-- 查询
select (@rowNO := @rowNo+1) AS rowno,Score from (select @rowNO:=0) as temp, `Scores`  ORDER BY Score desc;

-- group by的自增序号
-- 错误:
select (@rowNO := @rowNo+1) AS rowno,Score from (select @rowNO:=0) as temp, `Scores` GROUP BY Score  ORDER BY Score desc;
-- 正确:
select (@rowNO := @rowNo+1) AS rowno,s.Score from (select @rowNO:=0) as temp, (SELECT * from `Scores` GROUP BY Score  ORDER BY Score desc) as s;
```

### 4.2 leetcode的一个题:       
编写一个 SQL 查询来实现分数排名。如果两个分数相同，则两个分数排名（Rank）相同。请注意，平分后的下一个名次应该是下一个连续的整数值。换句话说，名次之间不应该有“间隔”。        

+----+-------+
| Id | Score |
+----+-------+
| 1  | 3.50  |
| 2  | 3.65  |
| 3  | 4.00  |
| 4  | 3.85  |
| 5  | 4.00  |
| 6  | 3.65  |
+----+-------+
例如，根据上述给定的 Scores 表，你的查询应该返回（按分数从高到低排列）：        

+-------+------+
| Score | Rank |
+-------+------+
| 4.00  | 1    |
| 4.00  | 1    |
| 3.85  | 2    |
| 3.65  | 3    |
| 3.65  | 3    |
| 3.50  | 4    |
+-------+------+

来源：力扣（LeetCode）     
链接：https://leetcode-cn.com/problems/rank-scores     
著作权归领扣网络所有。商业转载请联系官方授权，非商业转载请注明出处。      

题目分析:       
先group by order by 拿到分数从高到底的排名,增加排名序号; 然后用分数表left join 按照分数高低排名即可       

```sql
-- ----------------------------
-- Table structure for Scores
-- ----------------------------
DROP TABLE IF EXISTS `Scores`;
CREATE TABLE `Scores` (
  `Id` int(11) NOT NULL AUTO_INCREMENT,
  `Score` float(5,2) NOT NULL,
  PRIMARY KEY (`Id`)
) ENGINE=InnoDB AUTO_INCREMENT=0 DEFAULT CHARSET=utf8;

-- ----------------------------
-- Records of Scores
-- ----------------------------
INSERT INTO `Scores` VALUES ('1', '3.50');
INSERT INTO `Scores` VALUES ('2', '3.65');
INSERT INTO `Scores` VALUES ('3', '4.00');
INSERT INTO `Scores` VALUES ('4', '3.85');
INSERT INTO `Scores` VALUES ('5', '4.00');
INSERT INTO `Scores` VALUES ('6', '3.65');

-- 最终查询语句:
select m.`Score`,convert(r.rowno, UNSIGNED) as Rank from Scores as m LEFT JOIN (select (@rowNO:= @rowNo+1) AS rowno,s.Score from (SELECT *  FROM `Scores` group by `Score` order by `Score` desc ) as s, (select @rowNO:=0) as temp
) as r on m.`Score`=r.`Score` ORDER BY m.`Score` desc;
```


## 5.类型转换convert, 首字符排序 
### 5.1.语法说明:   
MySQL 的CAST()和CONVERT()函数可用来获取一个类型的值，并产生另一个类型的值。两者具体的语法如下：  
CAST(value as type);    
CONVERT(value, type);   
即: CAST(xxx AS 类型), CONVERT(xxx,类型)。    
例如: SELECT CONVERT('23.00', SIGNED);    

可以转换的类型是有限制的。这个类型可以是以下值其中的一个：   
二进制，同带binary前缀的效果 : BINARY      
字符型，可带参数 : CHAR()       
日期 : DATE       
时间: TIME        
日期时间型 : DATETIME        
浮点数 : DECIMAL      
整数 : SIGNED         
无符号整数 : UNSIGNED    

### 5.2.以汉字首字符排序    
name字段按照汉字正序 , 以name开头第一个字符来排序,依次是空格 0-9 a-z 字符 ,汉字首字拼音的首字母按照a-z来排序;    
select DISTINCT `name` from `your_table` order by convert(`name` using gbk) asc limit 300;  


## 6. union 和 union all 的区别
union在进行表求并集后会去掉重复的元素，所以会对所产生的结果集进行排序运算，删除重复的记录再返回结果。 
union all则只是简单地将两个结果集合并后就返回结果。因此，如果返回的两个结果集中有重复的数据，那么返回的结果就会包含重复的数据。 
**tips:** 使用联合查询,想区分结果集中的数据来自哪一张表,可以在查询中增加一个标志   
```sql
SELECT id,name ,1 as from_table_name  FROM `teacher`  union  select id,name,2 as from_table_name  from `student`;
```


## 7.获取每个分组下的前N条数据
示例:找出各单位薪资前三高的员工
```sql
-- ----------------------------
-- Table structure for user_salary
-- ----------------------------
DROP TABLE IF EXISTS `user_salary`;
CREATE TABLE `user_salary` (
  `Id` int(11) NOT NULL AUTO_INCREMENT,
  `Name` varchar(255) NOT NULL,
  `Salary` int(11) NOT NULL COMMENT '薪水',
  `DepartmentId` int(11) NOT NULL COMMENT '部门id',
  PRIMARY KEY (`Id`)
) ENGINE=InnoDB AUTO_INCREMENT=12 DEFAULT CHARSET=utf8 COMMENT='各部门员工薪资表';

-- ----------------------------
-- Records of user_salary
-- ----------------------------
INSERT INTO `user_salary` VALUES ('1', 'one_1', '2000', '1');
INSERT INTO `user_salary` VALUES ('2', 'two_1', '3000', '1');
INSERT INTO `user_salary` VALUES ('3', 'three_1', '1000', '1');
INSERT INTO `user_salary` VALUES ('4', 'four_1', '4000', '1');
INSERT INTO `user_salary` VALUES ('5', 'one_2', '8000', '2');
INSERT INTO `user_salary` VALUES ('6', 'two_2', '5000', '2');
INSERT INTO `user_salary` VALUES ('7', 'three_2', '6000', '2');
INSERT INTO `user_salary` VALUES ('8', 'four_2', '7000', '2');
```
由于mysql执行的先后顺序导致了不能简单的使用group by + order by + limit 来完成该查询;
(1).查询示例:
```sql
mysql> SELECT * FROM `user_salary` as a where 3>(SELECT count(*) FROM `user_salary` as b where b.`DepartmentId`=a.`DepartmentId` and b.`salary` > a.`salary`)
ORDER BY a.DepartmentId asc,a.Salary desc;
+----+---------+--------+--------------+
| Id | Name    | Salary | DepartmentId |
+----+---------+--------+--------------+
|  4 | four_1  |   4000 |            1 |
|  2 | two_1   |   3000 |            1 |
|  1 | one_1   |   2000 |            1 |
|  5 | one_2   |   8000 |            2 |
|  8 | four_2  |   7000 |            2 |
|  7 | three_2 |   6000 |            2 |
+----+---------+--------+--------------+
6 rows in set

mysql> 
```
说明:采用逆向思维。各部门薪资最高的前三位，也就是薪资比该条记录的薪资还高的不能超过三条记录，即同一部门中,比当前记录薪资高的数据count(*)<3;

(2).修改下,第一种满足的数据出现大于所需条目的情况
```sql
-- 像部门2增加数据
INSERT INTO `young`.`user_salary` (`Id`, `Name`, `Salary`, `DepartmentId`) VALUES ('9', 'five_2', '8000', '2');
INSERT INTO `young`.`user_salary` (`Id`, `Name`, `Salary`, `DepartmentId`) VALUES ('10', 'six_2', '8000', '2');
INSERT INTO `young`.`user_salary` (`Id`, `Name`, `Salary`, `DepartmentId`) VALUES ('11', 'seven_2', '8000', '2');
```
同样的查询语句看结果:
```sql
 mysql> SELECT * FROM `user_salary` as a where 3>(SELECT count(*) FROM `user_salary` as b where b.`DepartmentId`=a.`DepartmentId` and b.`salary` > a.`salary`)
 ORDER BY a.DepartmentId asc,a.Salary desc;
 +----+---------+--------+--------------+
 | Id | Name    | Salary | DepartmentId |
 +----+---------+--------+--------------+
 |  4 | four_1  |   4000 |            1 |
 |  2 | two_1   |   3000 |            1 |
 |  1 | one_1   |   2000 |            1 |
 |  5 | one_2   |   8000 |            2 |
 |  9 | five_2  |   8000 |            2 |
 | 10 | six_2   |   8000 |            2 |
 | 11 | seven_2 |   8000 |            2 |
 +----+---------+--------+--------------+
 7 rows in set
 
 mysql> 
```
说明:可以看到部门2出现了四条满足情况的数据,因为最高的8000有四条;
(如果我们只需三条,可以拿到后按照我们想要的顺序排序,然后程序代码控制只取前三条)
(3). 类似(2)这样的,如果是想要不同值的前三名最高薪资,可以使用去重COUNT(DISTINCT expr,[expr...])
```sql
mysql> SELECT * FROM `user_salary` as a where 3>(SELECT count(DISTINCT b.`salary`) FROM `user_salary` as b where b.`DepartmentId`=a.`DepartmentId` and b.`salary` > a.`salary`)
ORDER BY a.DepartmentId asc,a.Salary desc;

+----+---------+--------+--------------+
| Id | Name    | Salary | DepartmentId |
+----+---------+--------+--------------+
|  4 | four_1  |   4000 |            1 |
|  2 | two_1   |   3000 |            1 |
|  1 | one_1   |   2000 |            1 |
|  5 | one_2   |   8000 |            2 |
|  9 | five_2  |   8000 |            2 |
| 10 | six_2   |   8000 |            2 |
| 11 | seven_2 |   8000 |            2 |
|  8 | four_2  |   7000 |            2 |
|  7 | three_2 |   6000 |            2 |
+----+---------+--------+--------------+
9 rows in set

mysql> 
```
**参考和题目来源leetcode**:
https://blog.csdn.net/wzy_1988/article/details/52871636
https://leetcode-cn.com/problems/department-top-three-salaries/


## 8.记录一个LeetCode的题目,对count的使用
题目链接:https://leetcode-cn.com/problems/trips-and-users/
题解:
(1). ①查出每天符合情况的, ②然后查出每天总的数目; 将①和②进行连表得出最终结果:
```sql
select total.Request_at as `Day`,if(round(complete.cnt/total.cnt, 2) > 0, round(complete.cnt/total.cnt, 2), 0.00) as `Cancellation Rate` from (
SELECT Request_at,count(*) as cnt FROM Users as u INNER JOIN Trips as t on u.Users_Id=t.Client_Id
where u.Role='client' and u.Banned='No' 
and Request_at BETWEEN '2013-10-01' and '2013-10-03' 
GROUP BY Request_at
) as total
LEFT JOIN (
SELECT Request_at,count(*) as cnt FROM Users as u INNER JOIN Trips as t on u.Users_Id=t.Client_Id
where u.Role='client' and u.Banned='No' and t.Status in ('cancelled_by_driver', 'cancelled_by_client')
and Request_at BETWEEN '2013-10-01' and '2013-10-03' 
GROUP BY Request_at
) as complete on total.Request_at = complete.Request_at;
```
很显然这些看起来很low
(2).利用count(null) = 0来解决这个问题(参考题解里的大佬答案):
```sql
select
    t.request_at Day, 
    (
        round(count(if(status != 'completed', status, null)) / count(status), 2)
    ) as 'Cancellation Rate'
from
    Users u inner join Trips t
on
    u.Users_id = t.Client_Id
and
    u.banned != 'Yes'
where
    t.Request_at BETWEEN '2013-10-01' and '2013-10-03' 
group by
    t.Request_at;
```
看起来容易理解多了;