# 二、Connection Pool
在后台程序中通常不会直接使用数据库连接，而是使用数据库连接池(pool)，线程从pool中获取一个可用的连接，再通过连接操控db，完成操作后需要释放连接。最常用的连接池就是PoolDB，这是DBUtils包下的一个模块。
```py
$ pip install DBUtils

from DBUtils.PooledDB import PooledDB
```

## 1.构造函数
首先看下PoolDB构建所需要的参数，这些参数和PoolDB的性能紧密相关：
```py
class PooledDB:
    def __init__(self,
            # 用来创建连接的对象，需要符合PEP249协议, 如MySQLdb, mysql.connector等
            creator,
            # 最小缓存的连接数, pool初始化时将会预先建立mincached个数的连接
            mincached=0,
            # 最大缓存的连接数, pool的连接个数超过maxcached个数的连接后，将会删除多余的连接。0表示不回收任何连接。
            maxcached=0,
            # 当连接数达到maxshared后，开始复用连接(0表示连接不进行复用)。该参数受数据库驱动的threadsafety参数影响。
            maxshared=0,
            # 连接数上限, 0为不设上限，每当连接达到这个数时，将会阻塞或是抛出异常。shared机制下，不受该参数影响。
            maxconnections=0,
            # 达到最大连接数，再申请连接抛出异常或阻塞
            blocking=False,
            # 单个连接可以被重用的次数。单个连接的使用次数达到maxusage，将会使连接重置(closed and reopened)。
            # execute和call开头的cursor方法的调用，将会递增一次usage
            # None或0不会启动该功能。
            maxusage=None,
            # setsession是sql语句的元组，每个连接创建的时候，都会预先执行一遍这批sql语句。
            setsession=None,
            # 连接归还给pool时，是否进行重置
            reset=True,
            failures=None,
            # connection进行检查连接的条件
            ping=1,
            *args,
            **kwargs):
        pass
```
### 1).*mincached机制*
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
### 2).*maxcached机制*
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
### 3).*maxconnections机制*
当pool所持有的连接数达到了maxconnections后, 继续获取连接, PooledDB将会有两种处理方式:
* pool.connection()方法阻塞, 直到有连接归还。
* pool.connection()方法抛出异常。

现在来看看PooledDB中是如何实现的:
```py
 def connection(self, shareable=True):
    if shareable and self._maxshared:
        # share机制
        while (not self._shared_cache and self._maxconnections
                and self._connections >= self._maxconnections):
            self._wait_lock()
        ...
    else:
        # dedicate机制
        while (self._maxconnections
                and self._connections >= self._maxconnections):
            self._wait_lock()
        ...

def _wait_lock(self):
    # 允许阻塞，则进行阻塞，否则抛出TooManyConnections异常。
    if not self._blocking:
        raise TooManyConnections
    self._lock.wait()
```
这里需要注意一下shared机制的处理，对于share机制其实是走不到`self._wait_lock()`逻辑的，因为需要`_shared_cache`为空是必要条件，而一旦有连接被取出，该连接`必然`是在`_shared_cache`中的，因此若有`_connections`过大，则`_shared_cache`必定不为空，因此必定不满足该条件。


### 4).*maxshared机制*
shared机制用来允许一个连接被多个线程复用，当非空闲连接个数达到maxshared时，将会使连接在线程间共享。在初始化时，将会根据threadsafety来计算pool可以接受的maxshared，如下代码:
```py
def __init__(...):
    ...
    # 获取threadsafety
    try:
        threadsafety = creator.threadsafety
    except AttributeError:
        try:
            if not callable(creator.connect):
                raise AttributeError
        except AttributeError:
            threadsafety = 2
        else:
            threadsafety = 0
    # 计算_maxshared
    if not threadsafety:
        raise NotSupportedError("Database module is not thread-safe.")
    if threadsafety > 1 and maxshared:
        self._maxshared = maxshared
        self._shared_cache = []
    else:
        self._maxshared = 0
    ...
```
很明显，只有当threadsafety大于1时，才会真正的使用__init__参数中的maxshared，否则maxshared为0.对于threadsafety的含义参考`DataBase API`中的描述。常用的MySQL数据库驱动库MySQLdb, mysql.connector等，该参数都是1(线程不安全，不应该被多线程共享同一个连接)，也就是说通常我们不会用到maxshared机制。

### 5).*setsession机制*
该机制最开始听名字没明白，但其实非常简单，就是创建连接完成后，预先执行一批sql语句。该机制的实现，交给类`SteadyDBConnection`, 这个类用来提供稳定的连接，也是在连接池中所保存的实际对象。该类是对实际的DataBase API的第一层封装:
```py
class SteadyDBConnection:
    def __init__(self, ..., setsession, ...):
        self._setsession_sql = setsession

    def _setsession(self, con=None):
        if con is None:
            con = self._con
        # 如果存在setsession的sql语句，遍历执行
        if self._setsession_sql:
            cursor = con.cursor()
            for sql in self._setsession_sql:
                cursor.execute(sql)
            cursor.close()

    # 创建连接的时候，将会调用该方法
    def _create(self):
        ...
        con = self._creator(*self._args, **self._kwargs)
        self._setsession(con)
        return con
```

## 2.如何归还连接
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

## 3.归还连接的时机
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

## 4.并发取出连接的性能问题
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
## 5.并发连接归还的性能问题
```py
```

## 6.share机制
在`构造函数`一节中有对shared机制有做一个简单的介绍，该机制在MySQL中并不常用，这里详细介绍该机制的原理和目的。

shared机制下，pool中维护了两个队列:
* _idle_cache，用于存放空闲连接的队列，通常在pool初始化时会放入一些空闲连接，以及当一个连接的被线程引用的个数为0时，将会被放入空闲连接队列。
* _shared_cache，用于存放非空闲连接的队列。

取连接时，_shared_cache已经满了，则会从_shared_cache中取出连接进行复用，_shared_cache未满(使用的连接未达到_maxshared)，则从_idle_cache取出空闲连接，若_idle_cache无空闲连接，则会新创建一个连接。从_idle_cache中取出来的或是新创建的连接，都会通过`SharedDBConnection`进行包装，放入_shared_cache中。
### 1).*连接的取出*
在`pool.connection()`操作中将会取出连接，在源码中将会通过_maxshared参数来进行判断:
```py
def connection(self, shareable=True):
    # 可以看到默认是shareable=True，因此通常时通过_maxshared进行判断，该参数在pool初始化时决定
    if shareable and self._maxshared:
        # share机制下的连接取出
```
接下来单独看shared机制的连接取出代码:
```py
# 对pool上锁，确保数据结构的线程安全
self._lock.acquire()
# _shared_cache为空时，如果连接数达到了上限，直接进行阻塞，知道连接被归还
# 如果_shared_cache不为空，是不会wait的，因为这时会去取_shared_cache中的连接来进行复用
# shared机制通常不会进行wait，因为一旦有连接，该连接就会放到_shared_cache中，直到该连接没有被任何线程所引用。
while (not self._shared_cache and
        self._maxconnections
        and self._connections >= self._maxconnections):
    self._wait_lock()
if len(self._shared_cache) < self._maxshared:
    # shared_cache未满，从空闲队列中取出连接，或是创建新的连接
    try:
        con = self._idle_cache.pop(0)
    except IndexError:
        con = self.steady_connection()
    else:
        con._ping_check()
    # 将一个连接用SharedDBConnection进行包装
    # SharedDBConnection主要提供连接的引用计数，以及为其排序提供函数。
    con = SharedDBConnection(con)
    self._connections += 1
else: # shared_cache满了，从shared_cache中取出连接进行复用
    # sort将会用cache中的引用次数生序排列，以便取出引用个数最少的连接
    self._shared_cache.sort()
    con = self._shared_cache.pop(0)
    # 如果连接在事务中，连接不应该进行共享，并等待，唤醒后重新取出连接个数最少的
    # 可以看出，如果使用shared机制，应该尽量不使用事务，避免阻塞降低并发能力
    while con.con._transaction:
        self._shared_cache.insert(0, con)
        self._wait_lock()
        self._shared_cache.sort()
        con = self._shared_cache.pop(0)
    con.con._ping_check()
    con.share()
self._shared_cache.append(con)
self._lock.notify()
self._lock.release()
# PooledSharedDBConnection进行包装，该包装主要提供conn回收的能力。
con = PooledSharedDBConnection(self, con)
```

### 2).*连接的归还*
调用`conn.close()`，将会归还连接，在shared机制下归还有两层含义：
* 连接的引用计数减少。
* 连接的引用计数为0时，将连接从shared_cache取出，并放入idle_cache(是否真的归还，还需要依靠maxcached参数决定时归还还是销毁)

shared机制下的连接被`PooledSharedDBConnection`包装，该类提供了归还的函数, 来看看如何归还的:
```py
class PooledSharedDBConnection:
    ...
    def close(self):
        if self._con:
            # 减少引用计数，并判断是否已经空闲
            self._pool.unshare(self._shared_con)
            self._shared_con = self._con = None

    def __del__(self):
        try:
            self.close()
        except Exception:
            pass

# pool的unshare函数:
def unshare(self, con):
    self._lock.acquire()
    try:
        # 减少引用计数
        con.unshare()
        shared = con.shared
        # 引用计数为0，从shared_cache中删除
        if not shared:
            self._shared_cache.remove(con)
    finally:
        self._lock.release()
    # 引用计数为0，将连接放回给idle_cache
    if not shared:
        self.cache(con.con)
```

## 6.DBUtils中封装了多少连接类
从pool中拿到的连接的类并不是简单的MySQLdb或mysql.connector所生成的连接，而是对这些连接类做了层层封装:
* 非shared机制下，从pool拿到的连接类的包装关系:
    * PooledDedicatedDBConnection( SteadyDBConnection( MySQLdb ) )
* share机制下，从pool拿到的连接类的包装关系:
    * PooledSharedDBConnection( SharedDBConnection( SteadyDBConnection( MySQLdb ) ) )

可以看到Shared中可以看到封装了4个连接的包装类，可以分为两类:
* 包装类提供归还连接到pool时的逻辑
    * PooledDedicatedDBConnection, 提供对专有连接的归还处理。
    * PooledSharedDBConnection, 提供对复用连接的归还处理：减少引用计数，为0时将连接从shared_cache移到idle_cache或是删除。
* 包装类对连接的强化
    * SteadyDBConnection, 用来提供稳定的连接，例如提供ping_check检查连接。后面对该类有更加具体的分析。
    * SharedDBConnection, 用来提供对连接的引用计数，一般包装满足Database API的类，在DBUtils中，封装的就是SteadyDBConnection。

## 7.DBUtils中封装了多少连接池类
* PooledPg.PooledDB, 提供线程间可共享的数据库连接，这个共享包含了两层含义，多个线程存取的连接来自同一个池子, 以及连接可以同时被多个线程复用(shared机制)
* PersistentDB.PersistentDB, 提供线程专用的数据库连接，每个线程都会有个LocalThread为其提供专用连接。其实该类并不提供池化，一个线程可以获得的连接始终是同一个，不同的线程获得的连接不同。
* SimplePooledDB.PooledDB, 比较简单的连接池，并不能提供丰富的参数，不建议使用。

## 8.SteadyDB的使用
SteadyDB用来对DataBase API的connection和cursor进行包装，确保连接的稳定可靠。SteadyDB包含了两个类:
* SteadyDBConnection
    * 魔法方法:
        * `__init__(creator, maxusage, setsession, failures, ping, closeable)`
        * `__enter__(), __exit__()`, 连接提供一个with语句的上下文，上下文的结束并不会关闭连接，但是会结束一个会话(commit或rollback)
        * `__del__()`, 稀构函数，将会调用`_close()`来关闭连接。
    * 公有方法:
        * dbapi(), 返回creator。
        * threadsafety(), 获得connection的线程安全等级。
        * close(), 继续调用`_close()`关闭连接，但`closeable=False`时，会忽略掉关闭操作，这种情况下若事务存在，会使得连接重置。
        * begin(), 开启一个事务。
        * commit(), 提交事务。
        * rollback(), 回滚事务。
        * cancel(), 取消事务, 需要底层驱动支持该方法, `mysql.connector`是不支持该方法的。
        * ping(), 单纯检查连接是否存活，直接返回`self._conn.ping(...)`的结果。
        * cursor(), 获取一个SteadyDBCursor对象。
    * 私有方法:
        * _create(), 使用creator创建一个新的连接。
        * _setsession(con), 创造连接时，预先执行一批sql语句。在_create时都会执行_setsession。
        * _store(con), 保存一个连接，并且初始化相关参数。
        * _close(), 关闭连接。
        * _reset(), 重置连接。
        * _ping_check(), 检查连接是否存活，若不存活会重试打开一个新的连接。
        * _cursor(), 获取一个原生的cursor对象。
* SteadyDBCursor
    * 魔法方法:
        * `__init__(con)`, 构造函数, 构造函数中将会获得原生cursor，并缓存起来, 也会把父连接缓存起来。
        * `__enter__(), __exit__()`, 提供cursor的上下文，退出上下文时，cursor将会关闭。
        * `__getattr__(name)`, 用来获得原生cursor的属性和方法，进行调用。该函数会对原生方法进行包装，执行特殊逻辑。
        * `__del__()`, cursor的析构函数，会调用`close()`方法。
    * 公有方法:
        * close(), 关闭游标，关闭后继续使用将会抛出异常。
        * setinputsizes(sizes), 将输入参数的缓冲信息记录在SteadyDBCursor中。
        * setoutputsize(size, column), 将输入参数的缓冲信息记录在SteadyDBCursor中。
    * 私有方法:
        * _clearsizes(), 清空SteadyDBCursor中的输入和输出缓冲区信息.
        * _setsizes(cursor), 将SteadyDBCursor中的输入输出信息交给真实的cursor, 某些驱动，如`mysql.connector`，没有对缓冲区方法做实现。
        * _get_tough_method(), 对原生cursor的方法进行包装, 提供鲁棒的数据库操作。

从所包装类的接口上来看，丰富connection和cursor的特性，以及更稳定持久的连接, 具体来说，提供了以下的能力:
* 上下文机制，SteadyDBCursor和SteadyDBConnection都提供了上下文机制，用with进行开发，简化代码，增强可读性。
* 多种connection驱动的支持及统一，使用更统一的接口来使用不同的数据库驱动。
* maxusage机制, 连接被使用超过一定次数后，将会被重置。对maxusage的检查主要是在获取cursor和用cursor进行execute/call操作时。
* 更强的连接检查。若连接断开，将会进行重连，重连成功同样视连接正常。
* 更稳定的提交/回滚，在操作失败后将会重新打开连接。注意，重新打开连接仍会抛出应有的异常。
* 更稳定的连接, 需要conn的操作，若失败了，都会重连并重试。
