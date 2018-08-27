---
title: mysql深入
date: 2018-08-23 11:02:11
tags: MySQL，SQL
categories: MySQL
---

## Replace into

Replace into是Insert into的增强版。在向表中插入数据时，我们经常会遇到这样的情况：1、首先判断数据是否存在；2、如果不存在，则插入；3、如果存在，则更新。

但是Replace也有坑，

### 删除记录并且数据会变化

1. Replace into的执行步骤其实就是，先去检查查询当前要插入的数据的唯一索引是否存在，如果存在，则将当前的数据删除，然后重新新增一条数据，新增的数据除了自己设置的值，其他的值都是，默认值，假设test表有三个字段

> `id`  ` key` `value`,id为自增主键，key为唯一索引，
>
>   1       1          1        
>
>   2       2          2      
>
>   3       3          3       

那么`replace into test (key,value) values (3,4);`的时候，则会去删除当前的记key=3的记录，并重新insert一条新的记录，此时表面看是将数据更新了，但是，自增id的值已经变了。

> `id`  ` key` `value`
>
>   1       1          1        
>
>   2       2          2      
>
>   4       3          4       

### 删除多条记录

在表中有超过一个的唯一索引。在这种情况下，REPLACE将考虑每一个唯一索引，并对每一个索引对应的重复记录都删除，然后插入这条新记录。

假设有一个table1表，有3个字段a, b, c。它们都有一个唯一索引。 

假设table1中已经有了3条记录   　　

> a	 b 	c  　　
>
> 1	 1 	1  　　
>
> 2 	2 	2  　　
>
> 3 	3 	3 

下面我们使用REPLACE语句向table1中插入一条记录。   　　

`REPLACE INTO table1(a, b, c) VALUES(1,2,3);`   　　

返回的结果如下   　　Query OK, 4 rows affected (0.00 sec)   　　

在table1中的记录如下   　

> 　a   	b 	 c  　　
>
>    1 		2 	3  

我们可以看到，REPLACE将原先的3条记录都删除了，然后将（1, 2, 3）插入。 



## 替代方法

1. 通过两条SQL语句

```mysql
SELECT COUNT(*) FROM xxx WHERE ID=xxx;
if (x == 0)
    INSERT INTO xxx VALUES;
else
    UPDATE xxx SET ;
```

2. 如果在INSERT语句末尾指定了ON DUPLICATE KEY UPDATE，并且插入行后会导致在一个UNIQUE索引或PRIMARY KEY中出现重复值，则在出现重复值的行执行UPDATE；如果不会导致唯一值列重复的问题，则插入新行。**不过（ 这语法效率超低。此语发是mysql独有）**

   ```mysql
   INSERT INTO TABLE (a,c) VALUES (1,3) ON DUPLICATE KEY UPDATE c=c+1;
   UPDATE TABLE SET c=c+1 WHERE a=1;
   ```

   

## 详细可以参考：

 **[mysql的replace into“坑” ](http://jjhpeopl.iteye.com/blog/2368927)** 







