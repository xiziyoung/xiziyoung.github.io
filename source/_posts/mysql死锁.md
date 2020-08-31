---
title: mysql死锁
categories:
- 数据库
date: 2018-07-06 18:59:17
tags:
- mysql
---

## mysql中的锁:    

在MySQL中的锁分两大类，一种是读锁，一种是写锁，读锁（read lock）也可以称为共享锁（shared lock），写锁（write lock）也通常称为排它锁（exclusive lock）。    

读锁是共享的，或者说是相互不阻塞的。多个客户在同一时刻可以同时读取一个资源，且互不干扰。写锁则是排他的，就是说一个写锁会阻塞其他的写锁和读锁，这是出于安全策略的考虑，只有这样，才能确保在给定时间里，只有一个用户能执行写入，并防止其他用户读取正在写入的同一资源。  



(待补充: 锁力度,行锁,间隙锁等)  



## 死锁:

![img](lock.png)

### 死锁模拟

表结构:    

```sql
CREATE TABLE `test` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `name` varchar(255) NOT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB AUTO_INCREMENT=3 DEFAULT CHARSET=utf8;

```



**事务一T1中:**

```sql
mysql> START TRANSACTION;
Query OK, 0 rows affected

mysql> update `test` set `name`="T1" where `id`=1;
Query OK, 1 row affected
Rows matched: 1  Changed: 1  Warnings: 0

mysql> 
```

​	

**事务二T2中:**

新的事务T2  

```sql
mysql> START TRANSACTION;
Query OK, 0 rows affected

mysql> update `test` set `name`="T1" where `id`=3;
Query OK, 1 row affected
Rows matched: 1  Changed: 1  Warnings: 0

mysql> 
```



**事务T1:**

事务T1也执行删除操作 

```
mysql> update `test` set `name`="T1" where `id`=3;
---阻塞等待中,等待T2释放id=3记录的X锁
```



**事务T2**

```
mysql> update `test` set `name`="T2" where `id`=1;
1213 - Deadlock found when trying to get lock; try restarting transaction
mysql> 
```

事务T2抛错,

**事务T1继续执行:**

```sql
mysql> update `test` set `name`="T1" where `id`=3;
Query OK, 1 row affected
Rows matched: 1  Changed: 1  Warnings: 0

mysql> mysql> commit;
Query OK, 0 rows affected
```



发生死锁后，InnoDB会为对一个客户(此处为 事务T2)产生错误信息并释放锁。返回给客户的信息：   

```sql
Deadlock found when trying to get lock;
try restarting transaction
```

所以，另一个客户(事务T1)可以正常执行任务。死锁结束。    



## 日常注意:

事务尽量简单化:

```php
        //各种其他可以不在事务中执行的操作,应该在事务外执行,例如其他的数据获取,数据处理,数据格式调整,其他耗时处理等等

		$this->beginTrans(); //在事务内, 尽量只集中处理事务该处理的sql部分
        try {
            $this->update();
            
            //因为在事务中,处理这些,事务就会等待这些处理完了,再提交或者回滚,那样事务内的锁就会挂起太久, 容易导致死锁问题

            $this->delete();

            $this->commit();
        } catch (Exception $e) {
            $this->rollBack();
            throw new MessageException('操作失败', 0);
        }
```

