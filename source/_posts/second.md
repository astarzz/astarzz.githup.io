---
title: mysql不支持full join解决方案
date: 2018-11-2 17:22:27
categories: 
    - "数据库"
    - "mysql"
toc_number: true
tags:
	- mysql
	- full join
---
项目因为需要支持多个省份的不同运行环境，其中最大的差异自于数据库，mysql和oracle。每迭代一个新功能，都需要对数据库进行兼容测试，维护两套sql。
同为关系型数据库，但基础语法还是有些许差异。最近就有个新需求，因为mysql不支持full join制造了点小插曲，觉得还是有必要记录下来。
<!--more-->
 ## 问题背景
  * 目前有2张表：friends(朋友)、wokrmates(同事)，描述如下
  
  ```sql
  mysql>desc friends;
  +--------------+--------------+------+-----+---------+-------------- 
  --+
  | Field        | Type         | Null | Key | Default | Extra          
  |
  +--------------+--------------+------+-----+---------+-------------- 
  --+
  | id           | int(11)      | NO   | PRI | NULL    | 
  auto_increment |
  | belong       | int(11)      | YES  |     | NULL    |                
  |
  | friend_count | int(11)      | YES  |     | 0       |                
  |
  | desc         | varchar(100) | YES  |     | NULL    |                
  |
  +--------------+--------------+------+-----+---------+--------------          
  --+
  4 rows in set (0.00 sec)
  +----+--------+--------------+----------------+
  | id | belong | friend_count | desc           |
  +----+--------+--------------+----------------+
  |  1 |      1 |           99 | astar的朋友    |
  |  2 |      2 |           77 | jone的朋友     |
  |  3 |      3 |            8 | tom的朋友      |
  |  4 |      5 |          233 | mark的朋友     |
  +----+--------+--------------+----------------+
  ```
  
  ```sql
  mysql> desc workmates;
  +----------------+--------------+------+-----+---------+------------ 
  ----+
  | Field          | Type         | Null | Key | Default | Extra          
  |
  +----------------+--------------+------+-----+---------+------------ 
  ----+
  | id             | int(11)      | NO   | PRI | NULL    | 
  auto_increment |
  | belong         | int(11)      | YES  |     | NULL    |                
  |
  | workmate_count | int(11)      | YES  |     | 0       |                
  |
  | desc           | varchar(100) | YES  |     | NULL    |                
  |
  +----------------+--------------+------+-----+---------+------------ 
  ----+
  +----+--------+----------------+----------------+
  | id | belong | workmate_count | desc           |
  +----+--------+----------------+----------------+
  |  1 |      1 |             11 | astar的同事    |
  |  2 |      2 |             22 | jone的同事     |
  |  3 |      3 |             33 | tom的同事      |
  |  4 |      4 |             66 | jack的同事     |
  +----+--------+----------------+----------------+
  4 rows in set (0.00 sec)
  ```
  
  * 2张表中belong字段代表归属于某个用户id，各自的*_count字段是汇总数量，拥有多少个朋友/同事。现在需要统计各个用户的同事+朋友的总数。
  * 第一个反应的就是left/right join外连接的方式，用用户id做连接条件。但是某些用户只在其中一张表中有数据，比如mark在同事表中没有数据，jack在朋友表中没有数据。显然这种方式满足不了需求，需要2张表的全量数据，于是联想到了full join。
  * mysql数据库不支持full join(全连接)
  
  ```sql
  SELECT
    a.belong,
    sum(
      IFNULL( a.friend_count, 0 ) + IFNULL( b.workmate_count, 0 ) 
    ) 
  FROM
    friends a
    FULL JOIN workmates b ON a.belong = b.belong
    GROUP BY a.belong 
  ```
  
  ![alt](error.png)
---
## 解决方案
  * 2张表目前的情况如图：
  ![alt](intersection.png)
  需要的是一张表的全量(包含交集部分)+另一张表独有的数据
![alt](complete.png)
  * 一张表全量部分，用left/right join即可解决
  
  ```sql
  SELECT
     a.belong,
     sum(
       IFNULL( b.friend_count, 0 ) + IFNULL( a.workmate_count, 0 ) 
     ) count_all
  FROM
     workmates a
     LEFT JOIN friends b ON a.belong = b.belong
     GROUP BY a.belong;
  +--------+-----------+
  | belong | count_all |
  +--------+-----------+
  |      1 |       110 |
  |      2 |        99 |
  |      3 |        41 |
  |      4 |        66 |
  +--------+-----------+
  4 rows in set (0.00 sec)
  ```
  
  * 求另一张表独有数据，需要排除交集部分，所以需要使用where条件做过滤:
  
  ```sql
  SELECT
    a.belong,
    sum(
      IFNULL( a.friend_count, 0 ) + IFNULL( b.workmate_count, 0 ) 
    ) count_all 
  FROM
    friends a
    LEFT JOIN workmates b ON a.belong = b.belong 
  where 
    b.id is null -- 排除与workmates有交集部分
  GROUP BY
    a.belong;
  +--------+-----------+
  | belong | count_all |
  +--------+-----------+
  |      5 |       233 |
  +--------+-----------+
  1 row in set (0.00 sec)
  ```
  
  * 最后结果数据合并，使用union即可:
  
  ```sql
  SELECT
    a.belong,
    sum(
      IFNULL( b.friend_count, 0 ) + IFNULL( a.workmate_count, 0 ) 
    ) count_all 
  FROM
    workmates a
    LEFT JOIN  friends b ON a.belong = b.belong 
  GROUP BY
    a.belong
  UNION 
  SELECT
    a.belong,
    sum(
      IFNULL( a.friend_count, 0 ) + IFNULL( b.workmate_count, 0 ) 
    ) count_all 
  FROM
    friends a
    LEFT JOIN workmates b ON a.belong = b.belong 
  WHERE
    b.belong IS NULL 
  GROUP BY
    a.belong;
  +--------+-----------+
  | belong | count_all |
  +--------+-----------+
  |      1 |       110 |
  |      2 |        99 |
  |      3 |        41 |
  |      4 |        66 |
  |      5 |       233 |
  +--------+-----------+
  5 rows in set (0.00 sec)
  ```