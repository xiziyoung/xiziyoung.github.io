---
title: mysql使用
categories:
- 数据库
date: 2017-8-10 12:45:21
tags:
- mysql
---
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
|  1 | 123456@qq.com | 殰S婋帣??K%槠 | 12345678901 | 葡贇韚[qK》?x? | 123456@qq.com | 12345678901 |
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
|  2 | test@qq.com | O藬蕟L2爽?鷫? | 55555555555 | 科eV3X6?恢ss糉 | test@qq.com | 55555555555 |
+----+-------------+-----------------+-------------+----------------+-------------+-------------+

```
注意: php 的composer库 phpseclib/phpseclib 可以用来处理PKCS＃1（v2.1）RSA，DES，3DES，RC4，Rijndael，AES，Blowfish，Twofish，SSH-1，SSH-2， SFTP和X.509
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

