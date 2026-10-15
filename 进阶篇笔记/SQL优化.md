# SQL优化

## 插入数据

- insert优化

  - 批量插入

    ```sql
    #一条一条插入
    INSERT INTO 表 VALUES(...);
    INSERT INTO 表 VALUES(...);
    INSERT INTO 表 VALUES(...);
    ...
    
    #批量插入
    INSERT INTO 表 VALUES(...),(...),(...);
    ```

  - 手动提交事务

    ```sql
    #一般的MySQL是自动提交事务的
    #我们可以手动提交
    START TRANSACTION;
    INSERT INTO 表 VALUES(...),(...),(...);
    INSERT INTO 表 VALUES(...),(...),(...);
    INSERT INTO 表 VALUES(...),(...),(...);
    INSERT INTO 表 VALUES(...),(...),(...);
    ...
    COMMIT;
    ```

  - 主键顺序插入

    ```sql
    #主键乱序插入：6 5 8 3 2 9 7 4 1；
    #主键顺序插入：1 2 3 4 5 6 7 8 9；
    顺序插入的性能明显好于乱序插入
    ```

- 大批量插入数据

  如果一次性需要插入大量数据，insert的插入性能就不够用了，此时可以使用MySQL数据库提供的load指令进行插入。具体操作如下：

  ```sql
  #客户端连接服务端时，加上一条参数 --local-infile
  mysql --local-infile -u root -p
  #设置全局参数local_infile为1，开启从本地加载文件导入数据的开关
  SET GLOBAL local_infile = 1;
  #执行load指令，将准备好的数据加载到表结构中
  LOAD DATA LOCAL INFILE '文件路径' INTO TABLE '表名' FIELDS TERMINATED BY '每个数据的分隔符（如果是逗号分隔的就写,）' LINES TERMINATED BY '每一行的分隔符（如果是回车换行的话就写\n）'
  ```

  这种方法插入时，也是**主键顺序插入性能高于乱序插入**

## 主键优化

- 数据的组织方式

  InnoDB存储引擎中，表数据都是个呢局主键顺序组织存放的，这种存储方式的表成为**索引组织表**（index organized table **IOT**）

  ![Snipaste_2026-10-15_14-49-21](..\images\Snipaste_2026-10-15_14-49-21.png)

  ![Snipaste_2026-10-13_08-44-44](..\images\Snipaste_2026-10-13_08-44-44.png)

- 页分裂

  页可以为空也可以填充一般，还可以填充百分之百。每个页包含了2-N行数据（如果一行数据过大，会行溢出），根据主键排列

  ![Snipaste_2026-10-15_15-00-39](..\images\Snipaste_2026-10-15_15-00-39.png)![Snipaste_2026-10-15_19-37-08](..\images\Snipaste_2026-10-15_19-37-08.png)
  
  如上图，乱序插入，50应该插到47后面，但是47后面空间不够了。这时候，会新建一个页，把第一页的后50%挪到新的第三页，再把50插到他后面，同时改变子节点之间的链表关系
  
  ![Snipaste_2026-10-15_19-46-19](..\images\Snipaste_2026-10-15_19-46-19.png)
  
  上述这种情况叫做**页分裂**

- 页合并

  在删除一行记录时，实际上记录并没有被物理删除，只是记录被标记（flaged）为删除并且它的空间变得允许被其他记录声明使用

  当页中删除的记录达到 MERGE_THRESHOLD（默认为页的50%），InnoDB会开始寻找最靠近的页（前或后）看看是否可以将两个页合并以优化空间使用

  ![Snipaste_2026-10-15_20-07-34](..\images\Snipaste_2026-10-15_20-07-34.png)

  > 图中禁止标致就是被删除的记录

  到达MERGE_THRESHOLD之后，就会优化空间使用

  ![Snipaste_2026-10-15_20-11-24](..\images\Snipaste_2026-10-15_20-11-24.png)

  > 把后继的合并了，优化空间

- 主键设计原则
  - 满足业务需求的情况下，尽量降低主键的长度
  - 插入数据时，尽量选择顺序插入，选择使用AUTO_INCREMENT自增主键
  - 尽量不要使用UUID（随机字符串）或者其他自然主键，如身份证号
  - 业务操作，尽量避免对主键进行修改

