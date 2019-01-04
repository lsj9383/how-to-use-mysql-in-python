# 一、DataBase API
对于Python的数据库驱动API，首先需要关注和熟悉PEP(Python Enhancement Proposals)的建议，其中大部分常用的的Python-MySQL-API都会遵循[PEP249](https://legacy.python.org/dev/peps/pep-0249/)的建议，常见的数据库连接池也认为连接对象符合PEP249的规定。具体来说，PEP249定义了`数据库连接工厂(creator)`、`数据库连接(connection)`以及`数据库游标(cursor)`的接口:
```py
import mysql.connector
safety = mysql.connector.threadsafety       # 数据库连接的工厂
conn = mysql.connector(...)                 # 创造数据库连接
cursor = conn.
```

## 1.creator的全局变量
* apilevel, 字符串常量, 表示DataBase API的级别。只允许"1.0"和"2.0"，目前常用的库都是"2.0"。
* paramstyle, 字符串常量, 标识着sql语句的传餐格式:
    * "qmark", `...WHERE name=?`
    * "numeric", `...WHERE name=:1`
    * "named", `...WHERE name=:name`
    * "format", `...WHERE name=%s`
    * "pyformat", `...WHERE name=%(name)s`
* threadsafety, 表示DataBase API所支持的线程安全级别:
    * 0, creator不能在线程间共享
    * 1, creator可以线程共享，但connection不能线程复用
    * 2, creator和connection都可以在线程间共享
    * 3, creator、connection和cursor都可以在线程间共享

## 2.connection构造
connection对象代表client和MySQL服务器的一个TCP连接。通过以下构造函数进行创建:
```py
config = {
    'host': <host>,
    'user': <username>,
    'password': <password>,
    'port': <port>,
    'database': <database-name>,
    'charset': 'utf8'
}
connection = creator.connect(**config)
```

## 3.connection接口
connection管理了事务和游标, 并提供了相关的接口:
* commit(), 将事务提交到数据库。对于可以自动提交事务的database，需要将数据库关闭自动提交功能。
* rollback(), 回滚事务，可选，因为并非所有的数据库都支持事务。
* cursor(), 从连接获得一个新的cursor对象，若数据库没有游标的概念，则需要模块实现对游标的模拟。
* close(), 将会关闭client和MySQL的TCP连接，若connection存在未commit的操作，则事务将会被回滚。connection关闭后，connection和该connection生成的cursor的操作都会抛出异常。

## 4.cursor接口
游标对象用户操控数据库上下文，从同一个连接获得的cursor并不是孤立的，若一个cursor修改了数据库，则另一个相同连接的cursor会立即看到对应的修改，这和事务无关。不同连接之间的cursor的隔离性，则由事务提供。
* 提供的属性
    * description
    * rowcount, 该属性只读，是最后一次execute所生成或影响(SELECT/UPDATE/INSERT)的行数。如果cursor没有执行过execute或无法确定，该值为-1。
    * lastrowid, 该属性只读，属于DataBase API的扩展实现，并非所有都实现了该属性。提供上次修改或新增行的rowid(主键)，若db不支持rowid或没有设置rowid则返回None，若一次性有多个行的变更，则lastrowid是未定义的。
    * arraysize, 该属性可读可写，指明了每次执行`cursor.fetchmany()`时返回的行数，该属性默认为1。
* 接口方法
    * callproc(procname [, parameters]), 给定名称和参数，调用指定的存储过程。该方法可选，因为并非所有的db都支持存储过程，MySQL是支持的。
    * close(), cursor关闭，当关闭后若继续使用cursor，将会抛出异常。
    * execute(operation [, parameters]), 执行db命令，传参方式参考`creator.paramstyle`。parameters可以是一个集合，支持对db进行批量操作(需要传入参数multi=True，否则进行批量操作将会抛出异常)，但PEP249对于批量操作建议使用`executemany`
    * executemany(operation, seq_of_parameters)
    * fetchone(), 获取一行的结果。
    * fetchmany([size=cursor.arraysize]), 可以指定结果返回的行数，参数size默认为`cursor.arraysize`。PEP249建议最好使用arraysize参数，而不是在调用该函数时传入size参数，因为这样可以方便优化，但是`mysql.connector`源码中，并没有优化操作。
    * fetchall(), 获取所有行的结果。
    * nextset(), 使光标跳到下一个可用集。
    * setinputsizes(sizes), 设置execute中每个参数所需要的内存大小。
    * setoutputsize(size [, column]), 设置大列(数据可能很大的字段)的缓冲区大小。

### 1).*outputsize和inputsize的实现机制*
该文主要是讨论Python中使用MySQL，而常用的MySQL库基本上都没有实现该功能, 在`mysql.connector`中定义了该两个方法为空:
```py
def setinputsizes(self, sizes):
    """Not Implemented."""
    pass

def setoutputsize(self, size, column=None):
    """Not Implemented."""
    pass
```

### 2).*execute和executemany的批量操作机制*

### 3).*fetchmany实现机制*
在`mysql.connector`中，fetchmany就是反复调用fetchone，没有做任何优化。
```py
def fetchmany(self, size=None):
    res = []
    # 使用size参数或是arraysize属性，优先size参数。
    cnt = (size or self.arraysize)
    # 读取一行，直到行数达到size或arraysize，或是读取结果完毕。
    while cnt > 0 and self._have_unread_result():
        cnt -= 1
        row = self.fetchone()
        if row:
            res.append(row)
    return res
```

## 5.连接的事务

## 6.select的字典执行结果
cursor.fetch可以获取到结果中的一行数据，并且默认是以元组的形式进行保存。通常select返回的结果，以字典的形式更方便，这需要在获取游标对象的时候指定dictionary:
```py
cursor = connection.cursor(dictionary=True)
```
