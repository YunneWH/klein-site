---
title: SQL
icon: config
---

CnosDB SQL 的灵感来自于 [DataFusion](https://arrow.apache.org/datafusion/user-guide/introduction.html)，我们支持DataFusion的大部分SQL语法。



# **DDL**

## **创建数据库**

```sql
CREATE DATABASE [IF NOT EXISTS] db_name [WITH db_options]

db_options:
    db_option ...

db_option: {
	  TTL value
	| SHARD value
	| VNODE_DURATION value
	| REPLICA value
	| PRECISION {'ms' | 'us' | 'ns'}
}
```
### 参数说明
1. TTL： 表示数据文件保存的时间，默认为365天，用带单位的数据表示，支持天（d），小时（h），分钟（m），如TTL 10d，TTL 50h，TTL 100m，当不带单位时，默认为天，如TTL 30
2. SHARD：表示数据分片个数，默认为1
3. VNODE_DURATION：表示数据在shard中的时间范围，默认为365天，同样使用带单位的数据来表示，数据意义与TTL的value一致
4. REPLICA： 表示数据在集群中的副本数，默认为1
5. PRECISION：数据库的时间戳精度，ms 表示毫秒，us 表示微秒，ns 表示纳秒，默认为ns纳秒

## **修改数据库参数**
```sql
todo!()
```

## **创建外部表**

```sql
-- Column definitions can not be specified for PARQUET files

CREATE EXTERNAL TABLE [ IF NOT EXISTS ] table_name ( field_defination [, field_defination] ... ) tb_option

field_defination: 
    column_name data_type [ NULL ]

tb_option:
    STORED AS { PARQUET | NDJSON | CSV | AVRO }
    [ WITH HEADER ROW ]
    [ DELIMITER 'a_single_char' ]
    [ PARTITIONED BY ( column_name, [, ... ] ) ]
    LOCATION '/path/to/file'
```

### 参数说明
1. STORE AS：表示文件以什么格式储存，目前支持 PARQUET，NDJSON，CSV，AVRO格式
2. WITH HEADER ROW：仅在csv文件格式下生效，表示带有csv表头
3. DELIMITER：仅在csv格式下生效，表示列数据的分隔符 
4. PARTITIONED BY：使用创建表时指定的列来进行分区
5. LOCATION：表示关联的文件的位置

## **创建表**
```sql
CREATE TABLE [IF NOT EXISTS] tb_name (field_defination [, field_defination] ...TAGS(tg_name [, tg_name] ...))

field_defination:
   column_name data_type [field_codec_type]
```
### 使用说明：
1. 创建表时无需创建timestamp列，系统自动添加名为"time"的timestamp列
2. 各列的名字需要互不相同
3. 创建表时如果不指定压缩算法，则使用系统默认的压缩算法
4. 目前各种类型支持的压缩算法如下，每种类型第一个为默认指定的算法

    * BIGINT/BIGINT UNSIGNED：DELTA，QUANTILE，NULL
    * DOUBLE：GORILLA，QUANTILE，NULL
    * STRING：SNAPPY，ZSTD，GZIP，BZIP，ZLIB，NULL
    * BOOLEAN：BIPACK，NULL


## **修改表**
```sql
todo!()
```

## **删除表**

```sql
-- We don't support cascade and purge for now.
DROP TABLE [ IF EXISTS ] table_name
```

## 删除数据库
```sql
DROP DATABASE [IF EXISTS] db_name
```



# **DML**

## 插入数据
```sql
INSERT INTO table_item VALUES (TIME, ...) [, ...]
```
目前只支持插入常量,TIME列为必选


> CnosDB SQL 不支持其他DML。


# **DQL**

## **SELECT**

```sql
[ WITH with_query [, ...] ]
SELECT [ ALL | DISTINCT ] select_expression [, ...]
[ FROM from_item [, ...] ]
[ WHERE condition ]
[ GROUP BY [ ALL | DISTINCT ] grouping_element [, ...] ]
[ HAVING condition ]
[ { UNION | INTERSECT | EXCEPT } [ ALL | DISTINCT ] select ]
[ ORDER BY expression [ ASC | DESC ] [, ...] ]
[ OFFSET count ]
[ LIMIT { count | ALL } ]

-- from_item
-- 1.
table_name [ [ AS ] alias [ ( column_alias [, ...] ) ] ]
-- 2.
from_item join_type from_item
{ ON join_condition | USING ( join_column [, ...] ) }

-- join_type
[ INNER ] JOIN
LEFT [ OUTER ] JOIN
RIGHT [ OUTER ] JOIN
FULL [ OUTER ] JOIN
CROSS JOIN

-- grouping_element
()
```

## **WITH clause**

```sql
-- eg.
SELECT a, b
FROM (
SELECT a, MAX(b) AS b FROM t GROUP BY a
) AS x;

WITH x AS (SELECT a, MAX(b) AS b FROM t GROUP BY a)
SELECT a, b FROM x;
```



## **SELECT clause**

```sql
SELECT [ ALL | DISTINCT ] select_expression [, ...]
```

### **Select expressions**

```sql
expression [ [ AS ] column_alias ]
```
```sql
row_expression.* [ AS ( column_alias [, ...] ) ]
```
```sql
relation.*
```
```sql
*
```



## **GROUP BY clause**

### **复杂的分组操作**

### **GROUPING SETS**

```sql
-- 原始数据
SELECT * FROM shipping;
--  origin_state | origin_zip | destination_state | destination_zip | package_weight
-- --------------+------------+-------------------+-----------------+----------------
--  California   |      94131 | New Jersey        |            8648 |             13
--  California   |      94131 | New Jersey        |            8540 |             42
--  New Jersey   |       7081 | Connecticut       |            6708 |            225
--  California   |      90210 | Connecticut       |            6927 |           1337
--  California   |      94131 | Colorado          |           80302 |              5
--  New York     |      10002 | New Jersey        |            8540 |              3
-- (6 rows)
```
```sql
-- 如下查询演示了GROUPING SETS的语义
SELECT origin_state, origin_zip, destination_state, sum(package_weight)
FROM shipping
GROUP BY GROUPING SETS (
(origin_state),
(origin_state, origin_zip),
(destination_state));
--  origin_state | origin_zip | destination_state | _col0
--  --------------+------------+-------------------+-------
--   New Jersey   | NULL       | NULL              |   225
--   California   | NULL       | NULL              |  1397
--   New York     | NULL       | NULL              |     3
--   California   |      90210 | NULL              |  1337
--   California   |      94131 | NULL              |    60
--   New Jersey   |       7081 | NULL              |   225
--   New York     |      10002 | NULL              |     3
--   NULL         | NULL       | Colorado          |     5
--   NULL         | NULL       | New Jersey        |    58
--   NULL         | NULL       | Connecticut       |  1562
--  (10 rows)
```
```sql
-- 上述查询等价于
SELECT origin_state, NULL, NULL, sum(package_weight)
FROM shipping GROUP BY origin_state

UNION ALL

SELECT origin_state, origin_zip, NULL, sum(package_weight)
FROM shipping GROUP BY origin_state, origin_zip

UNION ALL

SELECT NULL, NULL, destination_state, sum(package_weight)
FROM shipping GROUP BY destination_state;
```

### **CUBE**

```sql
SELECT origin_state, destination_state, sum(package_weight)
FROM shipping
GROUP BY CUBE (origin_state, destination_state);
-- 上述语句等价于
SELECT origin_state, destination_state, sum(package_weight)
FROM shipping
GROUP BY GROUPING SETS (
(origin_state, destination_state),
(origin_state),
(destination_state),
()
);
```

### **ROLLUP**

```sql
SELECT origin_state, origin_zip, sum(package_weight)
FROM shipping
GROUP BY ROLLUP (origin_state, origin_zip);
SELECT origin_state, origin_zip, sum(package_weight)
FROM shipping
GROUP BY GROUPING SETS ((origin_state, origin_zip), (origin_state), ());
```

### **组合多个分组表达式**

```sql
-- grouping set、cube、rollup可以在同一个sql中出现多次
```

### **GROUPING operation**

```sql
SELECT origin_state, origin_zip, destination_state, sum(package_weight),
grouping(origin_state, origin_zip, destination_state)
FROM shipping
GROUP BY GROUPING SETS (
(origin_state),
(origin_state, origin_zip),
(destination_state)
);

-- origin_state | origin_zip | destination_state | _col3 | _col4
-- --------------+------------+-------------------+-------+-------
-- California   | NULL       | NULL              |  1397 |     3
-- New Jersey   | NULL       | NULL              |   225 |     3
-- New York     | NULL       | NULL              |     3 |     3
-- California   |      94131 | NULL              |    60 |     1
-- New Jersey   |       7081 | NULL              |   225 |     1
-- California   |      90210 | NULL              |  1337 |     1
-- New York     |      10002 | NULL              |     3 |     1
-- NULL         | NULL       | New Jersey        |    58 |     6
-- NULL         | NULL       | Connecticut       |  1562 |     6
-- NULL         | NULL       | Colorado          |     5 |     6
-- (10 rows)
```



## **HAVING clause**

```sql
SELECT count(*), mktsegment, nationkey,
CAST(sum(acctbal) AS bigint) AS totalbal
FROM customer
GROUP BY mktsegment, nationkey
HAVING sum(acctbal) > 5700000
ORDER BY totalbal DESC;
```



## **WINDOW clause**

```sql
select c1, first_value(c1) over (partition by c2)
from aggregate_test_100
```



## **Set operations**

### **UNION clause**

### **INTERSECT clause**

### **EXCEPT clause**



#### ORDER BY 子句

按引用的表达式对结果进行排序。默认情况使用升序 (ASC)。通过在 order by 的表达式后添加 DESC 按降序排序。



```sql
SELECT age, person FROM table ORDER BY age;
SELECT age, person FROM table ORDER BY age DESC;
SELECT age, person FROM table ORDER BY age, person DESC;
```

### LIMIT 子句

限制行数为count，count必须非负

例子：

```sql
SELECT age, person FROM table
LIMIT 10
```

## **OFFSET clause**



## **Join子句**

CnosDB支持`INNER JOIN`、`LEFT OUTER JOIN`、`RIGHT OUTER JOIN`、`FULL OUTER JOIN`和`CROSS JOIN`。



示例：

以下是基表x,y

```sql
select * from x;
+----------+----------+
| column_1 | column_2 |
+----------+----------+
| 1        | 2        |
+----------+----------+

select * from y;
+----------+----------+
| column_1 | column_2 |
+----------+----------+
| 1        | 2        |
+----------+----------+
```
#### INNER JOIN

关键字`JOIN`或`INNER JOIN`定义了一个只显示两个表中匹配的行的连接。

```sql
select * from x inner join x y ON x.column_1 = y.column_1;
+----------+----------+----------+----------+
| column_1 | column_2 | column_1 | column_2 |
+----------+----------+----------+----------+
| 1        | 2        | 1        | 2        |
+----------+----------+----------+----------+
```

#### LEFT JOIN

用关键字`LEFT JOIN`或`LEFT OUTER JOIN`定义一个右连接。该连接包括左表中的所有行，如果右表没有匹配行，则连接的右侧为空值。

```sql
select * from x left join x y ON x.column_1 = y.column_2;
+----------+----------+----------+----------+
| column_1 | column_2 | column_1 | column_2 |
+----------+----------+----------+----------+
| 1        | 2        |          |          |
+----------+----------+----------+----------+
```

#### RIGHT JOIN

用关键字`RIGHT JOIN`或`RIGHT OUTER JOIN`定义一个右连接。该连接包括右表中的所有行，如果左表没有匹配行，则连接的左侧为空值。

```sql
select * from x full outer join x y ON x.column_1 = y.column_2;
+----------+----------+----------+----------+
| column_1 | column_2 | column_1 | column_2 |
+----------+----------+----------+----------+
| 1        | 2        |          |          |
|          |          | 1        | 2        |
+----------+----------+----------+----------+
```



#### FULL JOIN 

关键字`FULL JOIN`或`FULL OUTER JOIN`定义了一个全连接，实际上它是 LEFT OUTER JOIN 和 RIGHT OUTER JOIN 的联合。 它会显示连接左侧和右侧的所有行，并将在连接的任一侧不匹配的地方产生空值。

```sql
select * from x full outer join x y ON x.column_1 = y.column_2;
+----------+----------+----------+----------+
| column_1 | column_2 | column_1 | column_2 |
+----------+----------+----------+----------+
| 1        | 2        |          |          |
|          |          | 1        | 2        |
+----------+----------+----------+----------+
```

#### CROSS JOIN

交叉连接产生一个笛卡尔积，它将连接左侧的每一行与连接右侧的每一行相匹配。

```sql
select * from x cross join x y;
+----------+----------+----------+----------+
| column_1 | column_2 | column_1 | column_2 |
+----------+----------+----------+----------+
| 1        | 2        | 1        | 2        |
+----------+----------+----------+----------+
```


### **Qualifying column names**



## **Subqueries**

### **EXISTS**

### **IN**

### **Scalar subquery**



# **DCL (无)**



# **EXPLAIN**

```sql
{ EXPLAIN | DESCRIBE } [ ANALYZE ] [ VERBOSE ] <statement>
```



# **DESCRIBE**

```sql
DESCRIBE table_name
```



# **SHOW**

## **SHOW VARIABLE**

```sql
-- only support show tables
-- SHOW TABLES is not supported unless information_schema is enabled
SHOW TABLES
```

## **SHOW COLUMNS**

```sql
-- SHOW COLUMNS with WHERE or LIKE is not supported
-- SHOW COLUMNS is not supported unless information_schema is enabled
-- treat both FULL and EXTENDED as the same
SHOW [ EXTENDED ] [ FULL ]
{ COLUMNS | FIELDS }
{ FROM | IN }
table_name
```

## **SHOW CREATE TABLE**

```sql
SHOW CREATE TABLE table_name
```
