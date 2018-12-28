# How to Use MySQL in Python
记录本人在python中使用MySQL所遇到过的问题，问题主要又分为三大部分：
* DataBase API, 讨论Python数据库API
* Connection Pool, 讨论连接池使用中遇到的问题.
* MySQL, 讨论MySQL本身所带来的问题.

## 一、DataBase API
对于Python的数据库驱动API，首先需要关注和熟悉PEP(Python Enhancement Proposals)的建议，其中大部分常用的的Python-MySQL-API都会遵循[PEP249](https://legacy.python.org/dev/peps/pep-0249/)的建议，常见的数据库连接池也认为连接对象符合PEP249的规定。
### 1.*构造函数*

### 2.*连接的事务*

## 二、Connection Pool
在后台程序中通常不会直接使用数据库连接，而是使用数据库连接池(pool)，线程从pool中获取一个可用的连接，再通过连接操控db，完成操作后需要释放连接。最常用的连接池就是PoolDB，这是DBUtils包下的一个模块。
```py
$ pip install DBUtils

from DBUtils.PooledDB import PooledDB
```

### 1.*构造函数*
首先看下PoolDB构建所需要的参数，这些参数和PoolDB的性能紧密相关：
```py
class PooledDB:
    def __init__(self,
            creator,            # 用来创建连接的对象，需要符合PEP249协议, 如MySQLdb, mysql.connector等
            mincached=0,        # 最小缓存的连接数, pool初始化时将会预先建立mincached个数的连接
            maxcached=0,        # 最大缓存的连接数, pool的连接个数超过maxcached个数的连接后，将会删除多余的连接。0表示不回收任何连接。
            maxshared=0,
            maxconnections=0,   # 连接数上限, 0为不设上线，每当连接
            blocking=False,     # 是否允许达到最大连接数后进行阻塞
            maxusage=None,
            setsession=None,
            reset=True,
            failures=None,
            ping=1,             # connection进行检查连接的条件
            *args,
            **kwargs):
        pass
```
#### 1).mincached机制的实现
mincache是预先建立连接的个数， 在pool的__init__函数中进行:
```py
class PooledDB:
    def __init__(...):
        ...
        # 连接池队列
        self._idle_cache = []
        self._lock = Condition()
        self._connections = 0
        # dedicated_connection是建立新连接的函数， 这里就会建立mincached数量的连接。
        idle = [self.dedicated_connection() for i in range(mincached)]
        # 建立的连接并没有放到_idle_cache中，需要通过close()函数，会自动将连接放到池子中。
        while idle:
            idle.pop().close()
```
#### 2).maxcached机制的实现
在回收一个从pool中获取的连接时，将会检查pool是否超过了_maxcached，若超过了_maxcached则会丢弃掉该连接。若_maxcached为0，则不会回收任何连接。
```py
class PooledDedicatedDBConnection:
    def close(self):
        if self._con:
            # 连接对象的关闭，将会把连接放回到pool中
            self._pool.cache(self._con)
            self._con = None

class PooledDB:
    ...
    def cache(self, con):
        self._lock.acquire()
        try:
            # 这里判断，如果_maxcached为0，或是_idle_cache没达到_maxcached，则进行放回pool，否则丢弃改连接。
            if not self._maxcached or len(self._idle_cache) < self._maxcached:
                con._reset(force=self._reset)
                self._idle_cache.append(con)
            else:
                con.close()
            self._connections -= 1
            self._lock.notify()
        finally:
            self._lock.release()
```
#### 3).maxconnections机制的实现

### 2.*如何归还连接*
最开对于连接的归我是懵逼的，虽然理所应当是`conn.close()`，但PEP249规定这个是关闭连接的操作，而非归还连接。观察源码后才发现，PoolDB对连接做了一层包装对象，包装会将除了`close()`方法全部路由给实际的连接，但是对于close函数，只会把连接归还给线程池。
```py
class PooledDedicatedDBConnection:
    # 归还给连接池
    def close(self):
        if self._con:
            self._pool.cache(self._con)
            self._con = None

    def __getattr__(self, name):
        # 将会代理除了close的其他所有方法和属性
        if self._con:
            return getattr(self._con, name)
        else:
            raise InvalidConnection
```
另外一点，pool中实际保存的是连接对象，而不是连接的包装对象，在每次调用`pool.connection()`时，将会从队列中获得一个连接，并且对该连接进行包装：
```py
# 下面是【伪码】
def connection(self):
    # 操作线程池的锁
    self._lock.acquire()
    # 若达到了最大的线程，直接阻塞
    while (self._maxconnections
            and self._connections >= self._maxconnections):
        self._wait_lock()
    # 从队列中取出一个连接
    con = self._idle_cache.pop(0)
    # 包装连接
    con = PooledDedicatedDBConnection(self, con)
    self._connections += 1
    self._lock.release()
    return con
```
上述只是一个简化的实现，并非PoolDB源码。

### 3.*归还连接的时机*
我们的代码中通常会编写如下的代码来`显示释放`连接:
```py
conn, cursor = None, None
conn = pool.connection()
cursor = conn.cursor()
try:
    execute_db(conn, cursor)
finally:
    cursor and cursor.close()
    conn and conn.close()
```
但是我们会发现，就算不进行`conn.close()`，连接也会被归还给pool。这是因为conn是一个被包装的对象，包装类实现了`__del__`函数，会在对象被垃圾回收的时候触发:
```py
class PooledDedicatedDBConnection:
    ...
    # 垃圾回收时将会触发
    def __del__(self):
        try:
            self.close()
        except Exception:
            pass
```

### 4.*取出连接的耗时*
这是一个相当重要的问题，PoolDB在面临大量的线程申请获得连接时，将会有非常严重的性能问题，类似如下的数据库操作:
```py
def execute():
    conn, cursor = None, None
    conn = pool.connection()
    cursor = conn.cursor()
    try:
        execute_db(conn, cursor)
    finally:
        cursor and cursor.close()
        conn and conn.close()

def execute_parallel(num):
    threads = []
    # 开启大量线程同时获取连接对象，并进行sql操作
    for i in range(num):
        t = threading.Thread(target=query)
        threads.append(t)
        t.start()
    # 等待线程结束
    for t in threads:
        t.join()

execute_parallel(10)
```
执行上述的代码，将会发现一个非常诡异的现象: 无论pool首先启动了多少个连接，每个线程的sql处理耗时，都会随着num的增大而线性增大。这种场景在后台面临高并发的时候将会经常遇到，上千个请求同时打到后台服务器，后台服务器无论是多线程或是协程进行db处理，都需要向pool申请连接。这主要是因为`pool.connection()`的耗时非常大，100个请求就有几百毫秒的获取请求的阻塞，并发性能怎么也上不去。分析源码后得知，pool中通过一个队列来缓存连接对象:
* 为了保证队列的线程安全性，会申请独占锁
* 为了保证连接的可用性，会进行ping_check(), 毫秒级耗时(少则几毫秒)。

这两步操作锁导致获取连接阻塞的关键因素: 一开始大量的线程建立起来，获取连接对象并被阻塞，每次只能一个线程获取一个连接, 每次获取连接都会进行几毫秒的延时，这会导致耗时的积累，并且越到后面的线程阻塞的时间就越长:
```py
def connection(self, shareable=True):
    self._lock.acquire()
    try:
        while (self._maxconnections
                and self._connections >= self._maxconnections):
            self._wait_lock()
            con = self._idle_cache.pop(0)
        except IndexError:
            con = self.steady_connection()
        else:
            # 检查连接可用性 耗时较大
            con._ping_check()
        con = PooledDedicatedDBConnection(self, con)
        self._connections += 1
    finally:
        self._lock.release()
    return con
```
阻塞的耗时主要是对连接的检查, 可以通过初始化时指定`ping=0`来避免对连接的可用性进行检查, 减少阻塞时间, 但是解决的并不彻底，一方面这不能保障连接的可用性，另一方面连接在归还给pool的时候也会上锁阻塞, 阻塞期间会进行连接的reset操作(后面将会有专门提及), 导致高并发sql操作在释放给pool也会导致较为严重的阻塞.

为了解决对pool操作的串行化问题, 可以采用pools(创建多个pool)的方案, 将串行压力从一个池分摊给多个池, 如果需要初始化pool设置`ping=0`:
```py
import random

pools_size = 10
pools = []
for idx in range(pools_size):
    pools.append(PooledDB(..., ping=0))

def execute():
    conn, cursor = None, None
    # 最简单的负载均衡方案，随机选择一个
    random_idx = random.randint(0, pools_size-1)
    conn = pools[random_idx].connection()
    cursor = conn.cursor()
    try:
        execute_db(conn, cursor)
    finally:
        cursor and cursor.close()
        conn and conn.close()

def execute_parallel(num):
    threads = []
    # 开启大量线程同时获取连接对象，并进行sql操作
    for i in range(num):
        t = threading.Thread(target=query)
        threads.append(t)
        t.start()
    # 等待线程结束
    for t in threads:
        t.join()

execute_parallel(10)
```
