# Mysql
+ 创建数据库
    - `CREATE DATABASE 数据库名`即可创建相应的数据库，命令不限定大小写，注意在创建含有特殊字符的数据库名的数据库的时候，需要使用反标记字符（\`)用于引用标识符，如 ' create database \`my.contacts\` '，数据表名同理。
    
    - 可以在连接数据库的同时就指定要连接的数据库，如`mysql -uroot -p 数据库名`，这样一连接上就进入了该数据库。

    - 进入mysql后，使用`use 数据库名`，进入和切换相应的数据库，使用`SELECT DATABASE()`可以查看当前进入的数据库是哪个。

    - 数据库被创建为数据目录中的一个目录，使用`SHOW VARIABLES LIKE 'datadir'`，可以查看到该目录所在的文件夹位置。

+ 创建表
    - 在表定义的时候，应该指定列的名称、数据类型（整型、浮点型、字符串等）和默认值（如果有的话）。

    - mysql支持的类型
        * 数字：tinyint、smallint、mediumint、int、bigint和big
        * 浮点型： decimal、float和double
        * 字符串： char、varchar、binary、varbinary、blob、text、enum和set
        * spatial数据类型
        * json数据类型
    
    - 例：
        ```
        CREATE TABLE IF NOT EXISTS `company`.`customers` (
            `id` int unsigned AUTO_INCREMENT PRIMARY KEY,
            `first_name` varchar(20),
            `last_name` varchar(20),
            `country` varchar(20)    
        ) ENGINE=InnoDB;
        ```  
        解释：</br>
        句点符号：表可以使用database.table来引用，如果使用use进入到了该database下，则可以直接使用table</br>
        IF NOT EXISTS: 若在创建表的时候已经存在了一个同名的数据表，则指定了IF NOT EXISTS的情况下，只会抛出一个警告，否则若不指定，则报异常。</br>
        ENGINE：用于指定创建表的数据引擎。InnoDB是唯一的事务引擎，也是默认引擎。</br>
        注意： 最后一个value后不能有逗号

    - 可以在同一的数据库中创建多个表
        ```
        CREATE TABLE payments (
            `customer_name` varchar(20) PRIMARY KEY,
            `payment` float
        );
        ```
    - 列出所有的表：`SHOW TABLES`

    - 查看表结构
        * `SHOW CREATE TABLE 表名\G`

        * `DESC 表名`

    - 创建表后，MySQL会在数据库目录内创建相应的`.ibd文件`，使用`SHOW VARIABLES LIKE 'datadir'`，可以查看到该目录所在的文件夹位置。

    - 克隆表结构（创建一个和已有的表具有相同结构的表）：`CREATE TABLE 新表名 LIKE 已存在的表的表名`

+ 操作数据库
    - 插入
        * 
        ```
        INSERT IGNORE INTO `company`.`customers`(first_name, last_name, country) 
        VALUES
        ('Mike', 'Christensen', 'USA'),
        ('Andy', 'Hollands', 'Australia'),
        ('Ravi', 'Vedantam', 'India'),
        ('Rajiv', 'Perera', 'Sri Lanka');
        ```
        解释：</br>
        IGNORE： 如果该行已经存在，此时加入了IGNORE,新添加的数据将被忽略，且INSERT操作被显示为成功，同时生成一个警告和重复数据的数目，否则报错。</br>
        若插入的是完整的数据内容，即不使用自增的值和默认值的话，可以不写(first_name, last_name, country)

    - 更新
        * 
        ```
        UPDATE customers SET first_name='Rajiv', country='UK' WHERE id=4;
        ```
        解释：</br>
        WHERE:这是用于过滤的字句。用于筛选出特定的数据。若不添加，则默认更改所有的数据。</br>
        **WHERE字句是强制性的。建议在事务中修改数据，以便在发现任何错误的时候轻松的回滚这些更改**
        
    - 删除
        ```
        DELETE FROM customers WHERE id=4 AND first_name='Rajiv';
        ```
        解释：</br>
        **WHERE字句是强制性的，若没有WHERE，则会删除表中所有的数据，建议在事务中修改数据，以便在发现任何错误的时候轻松的回滚这些更改**

    - REPLACE、INSERT和ON DUPLICATE KEY UPDATE
        * 在很多情况下，我们需要处理重复项。行的唯一性有主键标识。如果行已经存在，则REPLACE会简单地删除行并重新插入新行；如果该行不存在，则此时的REPLACE等同于INSERT。</br>
        使用REPLACE在插入重复项（主键是不能重复的）的时候，会显示更新了两行，这是因为，先删除了原先的记录，又添加了新的记录，所以显示改变了两行。
        
        * 如果想在行已经存在的情况下处理重复项，则需要使用ON DUPLICATE KEY UPDATE。如果指定了ON DUPLICATE KEY UPDATE选项，并且INSERT语句在PRIMARY KEY中引发了重复值，则MySQL会用新值更新已有行。</br>
        使用场景（类似于update操作，但是是以插入数据的形式进行的）：
        ```
        INSERT INTO payments VALUES ('Mike Christensen', 200) ON DUPLICATE KEY UPDATE payment=payment+VALUES(payment);
        INSERT INTO payments VALUES ('Mike Christensen', 300) ON DUPLICATE KEY UPDATE payment=payment+VALUES(payment);
        ```
        解释：</br>
        第一条语句直接在表中创建了一条新的数据，执行第二条语句时会发现customer_name这一项的值重复了，此时ON DUPLICATE KEY UPDATE就会生效，会覆盖原有的值，相当于对数据进行了更新，此时值应该是500。同时，我们发现显示修改了两条数据，这是因为，先删除了原先的记录，又添加了新的记录，所以显示改变了两行。</br>
        VALUES(payment)代表的是INSERT语句给的值，而payment是表中的列的值。

    - TRUNCATE TABLE
        * 删除整个表需要很长时间，因为MySQL需要逐行执行操作。删除表中的所有行（保留表结构）的最快方法是使用TRUNCATING TABLE语句。
        `TRUNCATE TABLE 表名`，是DDL操作，一旦数据被清楚，不能回滚。

+ 查询语句
    - select ...（输出） from ...（获取数据） where ...（过滤） group by ...（分组） having ...（过滤） order by ...（排序） limit ...（限定个数） ;
    
    - 执行的顺序：1.from 2.where 3.group by 4.select 5.having 6.order by 7.limit

    - group by:
        * 