# SQL

## SQL通用语法

1. SQL可以单行写可以多行写，以；作为结尾（这一点有点像c语言和java）
2. 多用空格/缩进来增强相关语句的可读性
3. MySQL语句不区分大小写，关键字建议使用大写
4. 注释相关内容：
   - 单行注释：--注释内容或#注释内容
   - 多行注释：/\*注释内容\*/

## SQL语句的分类

| 分类 | 全称                       | 作用                                                   |
| ---- | -------------------------- | ------------------------------------------------------ |
| DDL  | Data Definition Language   | 数据**定义**语言，定义数据库对象（数据库，表，字段）   |
| DML  | Data Manipulation Language | 数据**操作**语言，对数据库中的数据进行增删改           |
| DQL  | Data Query Language        | 数据**查询**语言，查找数据库中的表的记录               |
| DCL  | Data Control Language      | 数据**控制**语言，创建数据库用户，管理数据库的访问权限 |

## DDL

### 数据库操作

#### 查询

1. 查询所有数据库

   ```sql
   SHOW DATABASES;
   ```

2. 查询当前数据库

   ```sql
   SELECT DATABASE();
   ```

#### 创建

1. 创建数据库

   ```sql
   CREATE DATABASE [ IF NOT RXISTS ] 数据库名 [ DEFAULT CHARSET 字符集] [ COLLATE 排序规则];
   ```

#### 删除

1. 删除数据库

   ```sql
   DROP DATABASE [ IF EXISTS ] 数据库名;
   ```

#### 使用

1. 使用数据库

   ```sql
   USE 数据库名;
   ```

### 表操作

#### 数据类型

 [MySQL数据类型.xlsx](MySQL数据类型.xlsx) 

#### 查询

1. 查询当前数据库中所有表

   ```sql
   SHOW TABLES;
   ```

2. 查询表结构

   ```sql
   DESC 表名;
   ```

3. 查询指定表的建表语句（即创建该表时输入了哪些信息）

   ```sql
   SHOW CREATE TABLE 表名;
   ```

#### 创建

```sql
CREATE TABLE 表名(
	字段1 字段1类型[COMMENT 字段1注释]，
	字段2 字段2类型[COMMENT 字段2注释],
	字段3 字段3类型[COMMENT 字段3注释],
	......
	字段n 字段n类型[COMMENT 字段n注释]
)[COMMENT 表注释];
```

#### 修改

1. 添加字段

   ```sql
   ALTER TABLE 表名 ADD 字段名 类型 (长度) [COMMENT 注释] [约束];
   ```

2. 修改数据类型

   ```sql
   ALTER TABLE 表名 MODIFY 字段名 新数据类型(长度);
   ```

3. 修改字段名和字段类型

   ```
   ALTER TABLE 表名 CHANGE 旧字段名 新字段名 类型(长度) [COMMENT 注释] [约束];
   ```

4. 修改表名

   ```sql
   ALTER TABLE 表名 RENAME TO 新表名;
   ```

   

#### 删除

1. 删除字段

   ```sql
   ALTER TABLE 表名 DROP 字段名;
   ```

2. 删除表

   ```sql
   DROP TABLE[IF EXISTS] 表名;
   ```

3. 删除指定表，并重新创建该表[表里的数据就没了]

   ```sql
   TRUNCATE TABLE 表名;
   ```

   