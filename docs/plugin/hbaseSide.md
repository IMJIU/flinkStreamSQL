
## 1.格式
```
 CREATE TABLE tableName(
     columnFamily:columnName type as alias,
     ...
     PRIMARY KEY(keyInfo),
     PERIOD FOR SYSTEM_TIME
  )WITH(
     type ='hbase',
     zookeeperQuorum ='ip:port',
     zookeeperParent ='/hbase',
     tableName ='tableNamae',
     cache ='LRU',
     cacheSize ='10000',
     cacheTTLMs ='60000',
     parallelism ='1',
     partitionedJoin='false'
  );
```
## 2.支持版本
 hbase2.0

## 3.表结构定义
   
|参数名称|含义|
|----|---|
| tableName | 注册到flink的表名称(可选填;不填默认和hbase对应的表名称相同)|
| columnFamily:columnName | hbase中的列族名称和列名称 |
| alias | hbase 中的列对应到flink中注册的列名称 |
| PERIOD FOR SYSTEM_TIME | 关键字表明该定义的表为维表信息|
| PRIMARY KEY(keyInfo) | 维表主键定义;hbase 维表rowkey的构造方式:可选择的构造包括 md5(alias + alias), '常量',也包括上述方式的自由组合 |
  
## 4.参数

参数详细说明请看[参数详细说明](sideParams.md)

|参数名称|含义|是否必填|默认值|
|----|---|---|----|
| type | 表明维表的类型[hbase&#124;mysql]|是||
| zookeeperQuorum | hbase 的zk地址;格式ip:port[;ip:port]|是||
| zookeeperParent | hbase 的zk parent路径|是||
| tableName | hbase 的表名称|是||
| cache | 维表缓存策略(NONE/LRU)|否|NONE|
| partitionedJoin | 是否在維表join之前先根据 設定的key 做一次keyby操作(可以減少维表的数据缓存量)|否|false|

--------------

## 5.样例
### LRU维表示例
```
CREATE TABLE sideTable (  
        wtz:message varchar as message,
        wtz:info varchar as info , 
        PRIMARY KEY (rowkey),
        PERIOD FOR SYSTEM_TIME
) WITH (
        type = 'hbase',
        zookeeperQuorum = '192.168.80.105:2181,192.168.80.106:2181,192.168.80.107:2181',
        zookeeperParent = '/hbase',
        tableName = 'testFlinkStreamSql',
        parallelism = '1',
        cache = 'LRU',
        cacheSize ='10000',
        cacheTTLMs ='60000',
        parallelism ='1',
        partitionedJoin='false'
);
```

### ALL维表示例
```
CREATE TABLE sideTable (  
        wtz:message varchar as message,
        wtz:info varchar as info , 
        PRIMARY KEY (rowkey),
        PERIOD FOR SYSTEM_TIME
) WITH (
        type = 'hbase',
        zookeeperQuorum = '192.168.80.105:2181,192.168.80.106:2181,192.168.80.107:2181',
        zookeeperParent = '/hbase',
        tableName = 'testFlinkStreamSql',
        parallelism = '1',
        cache = 'ALL',
        cacheTTLMs ='60000',
        parallelism ='1',
        partitionedJoin='false'
);
```

## 6.hbase完整样例

### 数据说明

在hbase中，数据是以列簇的形式存储，其中rowKey作为主键，按字典排序。

在样例中，wtz为列族名，message, info为列名，数据在hbase中的存储情况为：
```
hbase(main):002:0> scan 'testFlinkStreamSql'
ROW     COLUMN+CELL                                                                                                                
 cfcd208495d565ef66e7dff9f98764dazhangsantest      column=wtz:info, timestamp=1587089266719, value=hadoop                                                                     
 cfcd208495d565ef66e7dff9f98764dazhangsantest      column=wtz:message, timestamp=1587089245780, value=hbase                                                                   
 c4ca4238a0b923820dcc509a6f75849blisitest      column=wtz:info, timestamp=1587088818432, value=flink                                                                      
 c4ca4238a0b923820dcc509a6f75849blisitest      column=wtz:message, timestamp=1587088796633, value=dtstack                                                                 
 c81e728d9d4c2f636f067f89cc14862cwangwutest      column=wtz:info, timestamp=1587088858564, value=sql                                                                        
 c81e728d9d4c2f636f067f89cc14862cwangwutest      column=wtz:message, timestamp=1587088840507, value=stream
```
在hbase中，rowKey是一个二进制码流，可以为任意字符串，flinkStreamSql读取rowKey并通过rowKey唯一确定数据，对rowKey没有任何限制,对rowKey可选择的构造包括 md5(alias + alias), '常量',也可以它们的自由组合。
在本次案例中，rowKey为了简单，设置成了"0,1,2"这样的数值型字符，若有必要，也可以设计得更为复杂。
### hbase异步维表关联完整案例
```
CREATE TABLE MyTable(
        id varchar,
        name varchar,
        address varchar
)WITH(
        type = 'kafka10',
        bootstrapServers = '172.16.101.224:9092',
        zookeeperQuorm = '172.16.100.188:2181/kafka',
        offsetReset = 'latest',
        topic = 'tiezhu_test_in',
        groupId = 'flink_sql',
        timezone = 'Asia/Shanghai',
        topicIsPattern = 'false',
        parallelism = '1'
);

CREATE TABLE MyResult(
        id varchar,
        name varchar,
        address varchar,
        message varchar,
        info varchar
)WITH(
        type = 'console'
 );
 
 CREATE TABLE sideTable (  
        wtz:message varchar as message,
        wtz:info varchar as info , 
        PRIMARY KEY (md5(rowkey1) + rowkey2 + 'test'),
         PERIOD FOR SYSTEM_TIME
) WITH (
        type = 'hbase',
        zookeeperQuorum = '192.168.80.105:2181,192.168.80.106:2181,192.168.80.107:2181',
        zookeeperParent = '/hbase',
        tableName = 'testFlinkStreamSql',
        parallelism = '1',
        cache = 'LRU',
        cacheSize ='10000',
        cacheTTLMs ='60000',
        parallelism ='1',
        partitionedJoin='false'
);

insert
into
    MyResult
    select
        a.name,
        a.id,
        a.address,
        b.message,
        b.info
    from
        MyTable a
    left join
        sideTable b
            on a.id=b.rowkey1 and a.name = b.rowkey2;
```

