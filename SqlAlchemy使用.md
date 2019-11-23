## SqlAlchemy 使用

### 简介

### 安装
+ 使用 `pip install sqlalchemy`进行安装
+ 默认情况下，sqlalchemy是直接支持SQLite3的，不需要安装额外的驱动程序，但是想要支持其他的数据库需要安装相应的驱动程序
	- 若想要支持mysql，需要安装pymysql驱动

### 使用（以使用MySql为例）
+ 连接数据库
	- 要连接数据库，首先需要建立一个SqlAlchemy引擎，为数据库创建一个公共接口来执行SQL语句
	- ```python
        from sqlalchemy import create_engine

        # 引擎
        engine = create_engine(
            'mysql+pymysql://root:739230854@localhost:3306/test', pool_recycle=3600
        )
        collection = engine.connect()  
        ```
	- 注意，`create_engine`函数会返回一个引擎的实例。但是，在调用需要使用链接的操作（比如查询）之前，它实际上并不会打开连接。


## SQLAlchemy Core

### 模式和类型

+ SqlAlchemy提供了四种类型，通用类型、SQL类型、厂商自定义类型、用户定义类型
+ 定义了大量的通用类型，可以从sqlalchemy.type模块中导入，为了方便起见，在sqlalchemy中也可以直接使用
+ 除了通用类型，还可以使用SQL标准类型和厂商自定义类型，是为了方便在通用类型无法满足时调用的
	- SQL标准类型在sqlalchemy.type中可用，且为了和通用类型区分，都要大写
	- 厂商自定义类型只使用于特定的后端数据库类型，可以通过所选方言文档以及Sqlalchemy站点确定可用的类型，都在sqlalchemy.dialects模块中，且每个数据库方言都有若干子模块，如使用postgresql数据库中的json格式，可以使用`from sqlalchemy.dialects.postgresql import json`

+ 完整代码：
    ```python
    from sqlalchemy import create_engine
    from sqlalchemy import MetaData
    from sqlalchemy import Table, Column, Integer, Numeric, String, ForeignKey, Boolean

    # 引擎
    engine = create_engine(
        'mysql+pymysql://root:739230854@localhost:3306/test', pool_recycle=3600
    )
    collection = engine.connect()  # 打开数据库的链接，创建表的时候并不需要

    # 元数据
    metadata = MetaData()

    # 表
    cookies = Table('cookies', metadata,
        Column('cookie_id', Integer(), primary_key=True),  # 作为主键
        Column('cookie_name', String(50), index=True),  # 创建索引，加快查找的速度
        Column('cookie_recipe_url', String(50)),  
        Column('cookie_sku', String(55)),
        Column('quantity', Integer()),
        Column('unit_cost', Numeric(12, 2))  # 长度以及精度
    )

    # 带有更多选项的表
    from datetime import datetime
    from sqlalchemy import DateTime

    user = Table('users', metadata,
        Column('user_id', Integer(), primary_key=True),
        Column('username', String(15), nullable=False, unique=True),  # 必须有，且不能为空
        Column('email_address', String(255), nullable=False),
        Column('phone', String(20), nullable=False),
        Column('password', String(25), nullable=False),
        Column('created_on', DateTime(), default=datetime.now),
        Column('updated_on', DateTime(), default=datetime.now, onupdate=datetime.now)  # 设置onupdate使得每次更新的时候将当前时间设置给列
    )

    from sqlalchemy import ForeignKey

    orders = Table('orders', metadata,
        Column('order_id', Integer(), primary_key=True),
        Column('user_id', ForeignKey('users.user_id')),  # 使用的是字符串，而不是对列的实际引用
        Column('shipped', Boolean(), default=False),
    )

    line_items = Table('line_items', metadata,
        Column('line_items)id', Integer(), primary_key=True),
        Column('order_id', ForeignKey('orders.order_id')),
        Column('cookie_id', ForeignKey('cookies.cookie_id')),
        Column('quantity', Integer()),
        Column('extended_cost', Numeric(12, 2))
    )

    # 表的持久化-即在数据库中创建表
    metadata.create_all(engine)
    ```


### 插入数据

+ `insert`插入
    ```python 
    # 插入数据
    ins = cookies.insert().values(
        cookie_name = 'chocolate chip',
        cookie_recipe_url = "http://xxx",
        cookie_sku = 'CC01',
        quantity = '12',
        unit_cost = '0.50'
    )
    # print(str(ins))
    '''
    INSERT INTO cookies 
        (cookie_name, cookie_recipe_url, cookie_sku, quantity, unit_cost) 
    VALUES 
        (:cookie_name, :cookie_recipe_url, :cookie_sku, :quantity, :unit_cost)
    '''
    # print(ins.compile().params)  # 可以查询随查询一起发送的实际参数
    '''
    {'cookie_name': 'chocolate chip', 'cookie_recipe_url': 'http://xxx', 
    'cookie_sku': 'CC01', 'quantity': '12', 'unit_cost': '0.50'}
    '''
    ```

+ 执行插入语句，执行前需要先添加引擎并连接
    ```python 
    from sqlalchemy import create_engine

    from sqlalchemy_test import cookies

    # 引擎
    engine = create_engine(
        'mysql+pymysql://root:739230854@localhost:3306/test', pool_recycle=3600
    )
    connection = engine.connect()  # 打开数据库的链接

    result = connection.execute(ins) # 使用链接执行

    print(result)  # <sqlalchemy.engine.result.ResultProxy object at 0x7f1d30fe5f10>
    print(result.inserted_primary_key)  # 产看刚刚添加的数据的ID
    ```

+ `insert`除了可以作为表实例cookies的方法来使用外，还可以作为单独的函数来使用，此时表实例cookies作为函数的参数，事实上，这种方法更加贴近于数据库的insert写法，因此更加推荐这种用法
    ```python 
    from sqlalchemy import insert

    ins = insert(cookies).values(
        cookie_name = 'chocolate chip',
        cookie_recipe_url = "http://xxx",
        cookie_sku = 'CC01',
        quantity = '12',
        unit_cost = '0.50'
    )
    ```

+ 连接对象的`execute`方法不仅可以接受语句，还可以在语句之后接受关键字参数值，在编译语句时，它会向列列表添加每个关键字参数键，并将它们的每个值添加到SQL语句的VALUES部分，但是这种用法不常用，但是他很好的说明了语句在发送到数据库服务器之前是如何编译和组装的。可以通过使用一个字典列表一次插入很多条记录，字典里面包含我们要提交的数据。
    ```python 
    ins = cookies.insert()
    result = connection.execute(
        ins,  # 插入语句任然是execute方法的第一个参数
        cookie_name = 'dack chocolate chip',
        cookie_recipe_url = "http://xxx/qqq",
        cookie_sku = 'CC02',
        quantity = '1',
        unit_cost = '0.75'
    )  # 这样在连接的同时，添加了数据
    ```

+ 进阶版，通过使用一个字典列表一次插入很多条记录，字典里面包含我们要提交的数据</br>
*注意创建字典列表的时候，字典必须拥有完全相同的键，sqlalchemy会根据列表中的第一个字典编译语句，如果后续的字典不同会出错*
    ```python
    inventory_list = [
        {
            'cookie_name': 'peanut butter',
            'cookie_recipe_url': 'http://xxx/1',
            'cookie_sku': 'PB01',
            'quantity': '24',
            'unit_cost': '0.25'
        },
        {
            'cookie_name': 'oatmeal raisin',
            'cookie_recipe_url': 'http://xxx/2',
            'cookie_sku': 'EWW01',
            'quantity': '100',
            'unit_cost': '1.00'
        }
    ]
    result = connection.execute(ins, inventory_list)
    ```

### 查找数据

+ `select`查询:</br>
select 需要一个列的列表来选择，为了方便，还可以直接接受一个Table实例（这里的cookies表），则此时选中该表中的所有的内容
    ```python
    from sqlalchemy import select

    s = select([cookies])  
    # print(str(s))  # 任然可以使用str(s)查看数据库看到的SQL语句 
    rp = connection.execute(s)  # rp是ResultProxy的缩写
    # print('rp:',type(rp))  # <sqlalchemy.engine.result.ResultProxy object at 0x7f2dc9f608d0>
    result = rp.fetchall()
    '''
    print(result)
    [
        (5, 'chocolate chip', 'http://xxx', 'CC01', 12, Decimal('0.50')), 
        (6, 'dack chocolate chip', 'http://xxx/qqq', 'CC02', 1, Decimal('0.75')), 
        (7, 'peanut butter', 'http://xxx/1', 'PB01', 24, Decimal('0.25')), 
        (8, 'oatmeal raisin', 'http://xxx/2', 'EWW01', 100, Decimal('1.00'))
    ]
    ```

+ 与`insert()`方法一致，Table实例对象也提供了select方法
    ```python
    s = cookies.select()  # 直接使用cookies的select方法
    rp = connection.execute(s)
    result = rp.fetchall()
    ```

### `ResultProxy`
+ ResultProxy 是DBAPI游标对象的包装器，主要目的是让语句返回的结果更容易使用和操作，比如，ResultProxy允许使用索引，名称或column对象进行访问，从而额简化了对查询结果的处理
    ```python 
    first_row = result[0]  # 获取ResultProxy的第一行
    print(first_row)
    print(first_row[1])  # 通过索引访问列
    print(first_row.cookie_name)  # 通过名称访问列
    print(first_row[cookies.c.cookie_name])  # 通过column对象访问列
    '''
    (5, 'chocolate chip', 'http://xxx', 'CC01', 12, Decimal('0.50'))
    chocolate chip
    chocolate chip
    chocolate chip
    '''
    ```

+ 迭代的方式获得ResultProxy的项
    ```python
    s = select([cookies])
    rp = connection.execute(s)
    print(rp.keys())  
    # ['cookie_id', 'cookie_name', 'cookie_recipe_url', 'cookie_sku', 'quantity', 'unit_cost']
    for record in rp:
        print(record)  # (9, 'chocolate chip', 'http://xxx', 'CC01', 12, Decimal('0.50'))
        print(type(record))  # <class 'sqlalchemy.engine.result.RowProxy'>  注意是 RowProxy
        print(record.cookie_name)  # chocolate chip
    ```

+ 除了将ResultProxy作为可迭代对象和调用fetchal()方法之外，还用其他通过ResultProxy访问数据的方式：
    - 每种对数据库的不同的操作对应的ResultProxy的方法不同
    - 使用rowcount()
    - 使用inserted_primary_key()获得插入记录的ID--使用insert插入数据的时候使用
    - first()，若有记录，则返回第一个记录并关闭连接，此时想再操作会报异常`sqlalchemy.exc.ResourceClosedError: This result object is closed.`
    - fetchone()，返回一行，并保持光标为打开的状态，以便你做更多的获取调用
    - scalar()，如果查询结果是包含一个列的单条记录，则返回单个值
    - 如果想产看结果集中的多个列，可以使用keys()方法来获得列名列表

+ 最佳实践，在编写生产代码时，应该遵循如下方针：
    1. 获取单条记录时，要多用first()方法，尽量不要使用fetchone()和scalar()方法，因为对程序员来说，first方法更加清晰，需要注意，使用first()返回第一条记录后将自动关闭连接
    2. 尽量使用可迭代对象ResultProxy，而不要用fetchall和fetchone方法，因为前者的内存效率更高，而且我们往往一次只对一条记录进行操作
    3. 避免使用fetchone方法，因为如果不小心，他会一直让连接处在打开状态，因为使用fetchone会返回一行，并保持光标为打开的状态，以便你做更多的获取调用
    4. 谨慎使用scalar方法，因为如果查询返回多行多列，就会引发错误，多行多列在测试过程中会经常丢失


