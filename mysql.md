# 三、MySQL
## 1.字符问题
### 1) 各类字符集之间的关系
MySQL存在着多种字符集:
* character_set_client, 客户端的字符集, 默认latin1
* character_set_connection, 客户端的字符集, 默认latin1
* character_set_database, 数据库的字符集, 默认utf8
* character_results, 数据库输出字符集, 默认为latin1
* 相互关系:
    * 向mysql中写数据
        * mysql会认为客户端的编码为character_set_client
        * 将character_set_client转换为character_set_connection, 若编码一致不做转化
        * 将character_set_connection转换为实际存储的字符集，实际存储的字符集指的是:
            * 如果指定了字段的字符集，则使用该字符集
            * 如果没有指定字段字符集，而指定了表的字符集，则使用该字符集
            * 如果都没有指定则看db的字符集(character_set_database)
    * 从mysql中读数据
        * 将实际存储的编码转换为character_results的编码进行输出。若两者一致不进行转化直接输出。

### 2) 相关命令
首先可以看看MySQL关于字符集的相关命令:
* 查看MySQL支持的字符集
```sql
-- 查看支持的字符集
SHOW CHARACTER SET;

-- 查看支持的字符序
SHOW COLLATION;

-- 查看全局级的字符集配置
SHOW VARIABLES LIKE "%character%";

-- 查看表级的字符集配置

-- 查看字段级的字符集配置
```

### 3) set names
`set names utf8`会把character_set_client、character_set_connection以及character_results修改为`utf8字符集`。仅对当前连接有效。

### 4) python中的字符问题
python2和python3中的字符处理有很大的不同。

## 附录 常见问题
## 1.double UTF8
在工作，有一个模块使用的是默认字符集，这时候db中存放的二进制数据为实际字符的二次utf8编码:
```sql
SELECT HEX(nickname) FROM users WHERE uuid = 1;
```
若nickname的中文是`夏天`, 则这里返回的hex数据为:`c3a5c2a4c28fc3a5c2a4c2a9`，这其实是`夏天`进行了utf8编码以后，再对编码结果再进行一次utf8编码的结果。造成这个结果的原因是:
* 客户端传输到mysql的底层数据是utf8编码，但是mysql默认将其视为latin1进行处理，因为最后存储的字段是utf8的，所以做了一次latin1到utf8的编码。
* 采用默认的方式，进行读数据是没有问题的，因为set_results的字符集是latin1，因此做了一次utf8到latin1的编码转换给到客户端，这次转化刚好抵消了传进来的latin1->utf8的转换，因此客户端拿到的数据就是实际的一次utf8编码的数据，可以正常解码并使用。