---
layout: post
title: "MySQL 8 Prepare stmt 查询结果异常问题排查"
date: "2022-05-05"
categories: [MySQL]
---

因为业务需要Window Function所以我们从MySQL5.7升级到了MySQL8（业务代码的SQL查询是适配了5.7和8的，也有部分客户是跑在MySQL8上）

以上是前提

SaaS版升级完MySQL8后出现了奇怪的问题。表现为：我们INSERT进去的数据SELECT不出来，直接报`sql no rows`的错误，但是我们拼接了同条SQL直接client查又可以查到，十分迷惑的问题。

以上是现象

排查方向锁定在几个方面：
1. 我们用的gorm有问题
2. MySQL的隔离级别不对
3. Binlog延迟
4. 手动拼的SQL和gorm执行的不是同一条

排查：

1. 首先我们没有用主从，读写都是在主库，因此没有Binlog的问题

2. MySQL的隔离级别也重新检查过，是没问题的，加上实际的业务是INSERT进去以后，过了好几分钟以后才会去SELECT，有这个时间Binlog都同步完了，主库读写也不能这么垃圾，这个也排除

3. gorm有问题，这个就难排查了，毕竟我们用了那么久gorm，都没有出过问题，升级MySQL 8以后就有问题也不太科学。但是秉着假设检验的态度，我们写了一个demo，没有使用gorm，直接用database/sql和官方的mysql driver查，发现结果也是`sql no rows`，既然官方driver也有这个问题，那就跟gorm关系不大。上Google搜索，发现Google上没有人反馈这个问题（主要是也不好检索），既然没有人反馈，我们先假定官方driver没有bug，且和MySQL8没有兼容性问题。

4. 整条查询SQL非常简单，从gorm中dump出来查询语句，长这样
```sql
select * from qianchuan_material_file where account_id = ? and mtr_outer_id = ?
```
因为语句很简单，所以手拼感觉应该也是没有问题，这里先假定他没有问题继续排查。

到这里根据经验定位的问题假设都被判否，只能拉上运维进入正式排查的范畴。

我们打开了general_log打印了全部的SQL和环境变量，看了一下SQL_MODE
```
2022-04-27T05:30:31.733790Z  791755 Query  SET NAMES utf8mb4
2022-04-27T05:30:31.735554Z  791755 Query  SET sql_mode='STRICT_TRANS_TABLES,NO_ZERO_IN_DATE,NO_ZERO_DATE,ERROR_FOR_DIVISION_BY_ZERO,NO_ENGINE_SUBSTITUTION'
```
SQL_MODE看起来没有异常，再grep到出问题的那段SQL
```
# Time: 2022-04-27T05:14:49.902796Z
# Query_time: 0.000205  Lock_time: 0.000060  Rows_sent: 0  Rows_examined: 0  Rows_affected: 0  Bytes_sent: 589
SET timestamp=1651036489;
SELECT * FROM `qianchuan_material_file` WHERE account_id = 19750 AND mtr_outer_id = '7087881148197896205' ORDER BY `qianchuan_material_file`.`mtr_outer_id` LIMIT 1;
```
可以看到`Rows_sent: 0`，也就是说MySQL真的没有返回东西，但打印的SQL看起来是没毛病的，这样基本就排除了driver的问题。

#### 问题解决

SHOW CREATE TABLE 看到account_id和mtr_outer_id都是bigint
```sql
CREATE TABLE `qianchuan_material_file` (
  `mtr_outer_id` bigint NOT NULL COMMENT '',
  `account_id` bigint unsigned NOT NULL COMMENT '',
  `file_type` tinyint unsigned NOT NULL COMMENT '',
  `file_id` varchar(128) NOT NULL COMMENT '',
  `created_at` datetime NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '',
  `updated_at` datetime NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '',
  PRIMARY KEY (`mtr_outer_id`,`account_id`,`file_type`,`file_id`),
  KEY `idx_account_id_file_id` (`account_id`,`file_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 utf8mb4_0900_ai_ci COMMENT=''
```
但是实际查询的时候传入的mtr_outer_id是带单引号’的，是否可能是这个原因引起的呢？我们把单引号去掉，在用gorm调用一次，发现有数据返回了。把单引号加上，没有数据返回。证实了是mtr_outer_id上有单引号的导致查不出数据。

那么问题来了，为什么在mysql-client上直接运行这条SQL是有数据返回的是为啥？
```sql
SELECT * FROM `qianchuan_material_file` WHERE account_id = 19750 AND mtr_outer_id = '7087881148197896205' ORDER BY `qianchuan_material_file`.`mtr_outer_id` LIMIT 1;
```

合理猜测是gorm里使用了Prepare statement语法，把单引号当做变量传入进去了导致匹配失败。我们在mysql-client里使用Prepare statement语法测试，复现了结果

<img width="803" alt="QQ20220505-172158@2x" src="https://user-images.githubusercontent.com/3870517/167053826-2535032c-d3a2-46fa-8463-0b926dbf50cd.png">


但随后更迷的事情出现了，SHOW TABLE可以看到两个id都是bigint，我们对两个id分别进行实验

<img width="965" alt="QQ20220505-172709@2x" src="https://user-images.githubusercontent.com/3870517/167053909-f7364280-01fc-4b38-9b33-8e49ee546654.png">
<img width="930" alt="QQ20220505-173205@2x" src="https://user-images.githubusercontent.com/3870517/167053911-86b5f75c-2c97-41d1-abd8-4e47761133ae.png">
<img width="921" alt="QQ20220505-173053@2x" src="https://user-images.githubusercontent.com/3870517/167053890-7a920f72-6efe-46cf-bb08-bd068287b91a.png">


可以看到，有且仅有mtr_outer_id加单引号时会出现异常。都不加单引号、都加单引号、account_id加单引号且mtr_outer_id不加单引号，这几种情况都能正常返回数据。

到这里问题的起因已经基本清楚，但是为啥是这样的情况还不得而知，这个问题细微到，我甚至都不知道如何在Google里写出问题的关键描述来进行检索。只能让运维同学去MySQL的社区里提个Issue。

后续：

本机Docker起MySQL测试
```shell
mysql> select version();
+-----------+
| version() |
+-----------+
| 8.0.29    |
+-----------+
```
MySQL 8.0.29 版本测试没有这个问题 -，-#

重新pull了和线上一致的版本8.0.27重新测试，问题复现
```shell
mysql> prepare stmt1 from 'SELECT * FROM `qianchuan_material_file` WHERE account_id = ? AND mtr_outer_id = ? ORDER BY `qianchuan_material_file`.`mtr_outer_id` LIMIT 1;';
Query OK, 0 rows affected (0.00 sec)
Statement prepared

mysql> set @a=21115;
Query OK, 0 rows affected (0.01 sec)

mysql> set @b='7082291618573664267';
Query OK, 0 rows affected (0.00 sec)

mysql> execute stmt1 using @a, @b;
Empty set (0.00 sec)

mysql> select version();
+-----------+
| version() |
+-----------+
| 8.0.27    |
+-----------+
1 row in set (0.00 sec)
```
所以这可能是一个小版本的Bug？并且在后续的小版本中被立即修复了