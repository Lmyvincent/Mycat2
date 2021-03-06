

# mycat 2.0-readme

author:junwen  2020-1-10

技术支持qq:  294712221

qq群:332702697

[![Creative Commons License](https://i.creativecommons.org/l/by-sa/4.0/88x31.png)](http://creativecommons.org/licenses/by-sa/4.0/)
This work is licensed under a [Creative Commons Attribution-ShareAlike 4.0 International License](http://creativecommons.org/licenses/by-sa/4.0/).

项目地址:<https://github.com/MyCATApache/Mycat2>

## 特点

1.proxy透传报文,使用buffer大小与结果集无关

2.proxy透传事务,支持XA事务,jdbc本地事务

3.支持分布式查询

## 限制

暂不支持预处理(客户端可以开启客户端预处理解决这个问题),游标等



测试版本的mycat2无需账户密码即可登录



## Mycat2流程

客户端发送SQL到mycat2,mycat2拦截对应的SQL执行不同的命令,对于不需要拦截处理的SQL,透传到有逻辑表的mysql,这样,mycat2对外就伪装成mysql数据库





## MetaData配置

#### 存储节点


dataNode是数据节点,库名,表名组成的三元组

targetName是目标名字,它可以是数据源的名字或者集群的名字

分片必然分库
分库必然分表


| 目标 targetName| 库schemaName   | 表tableName   | 类型   |
| ---- | ---- | ---- | ------ |
| 唯一 | 唯一 | 唯一 | 非分片 |
| 唯一 | 多目标 | 相同名字 |   非分片分库分表     |
|  唯一    |  唯一    |  不同名字    |   非分片单库分表     |
| 跨实例 | 多目标 | 不同名字 | 分片分库分表 |









## MetaData配置



### 概念

#### 分片类型

##### 自然分片

单列或者多列的值映射单值,分片算法使用该值计算数据节点范围

##### 动态分片

单列或者多列的值在分片算法计算下映射分片目标,目标库,目标表,得到有效的数据节点范围



```yaml
select * from db1.address1 where userid = 1 and  id = '122' and addressname ='abc';
```

即根据userid,id,addressname得出分片目标,目标库,目标表

例子1

id ->集群名字

id->库名

id->表名



例子2

任意值->固定集群名字

address_name->库名

user_name->表名



例子3

任意值->固定集群名字,固定库名

address_name->表名

user_name->表名



总之能通过条件中存在的字段名得出存储节点三元组即可

多列值映射到一个值暂时不支持,后续支持



##### 配置

```yml
metadata: #元数据 升级计划:通过创建表的sql语句提供该信息免去繁琐配置,
  schemas:
    db1: #逻辑库名
      tables:
        travelrecord: #逻辑表名
          columns:
            - columnName: id #分片字段信息,显式提供,
              shardingType: NATURE_DATABASE_TABLE #类型:自然分片,即根据一列(支持)或者多个列(暂不支持)的值映射成一个值,再根据该值通过单维度的分片算法计算出数据分片范围
              function: { clazz: io.mycat.router.function.PartitionByLong , name: partitionByLong, properties: {partitionCount: '4', partitionLength: '256'}, ranges: {}}
              #提供表的字段信息,升级计划:通过已有数据库拉取该信息
          createTableSQL: |-
            CREATE TABLE `travelrecord` ( `id` bigint(20) NOT NULL,`user_id` varchar(100) CHARACTER SET utf8 DEFAULT NULL,`traveldate` date DEFAULT NULL,`fee` decimal(10,0) DEFAULT NULL,`days` int(11) DEFAULT NULL,`blob` longblob DEFAULT NULL) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
          dataNodes: [{targetName: defaultDs ,schemaName: db1, tableName: travelrecord},
                      {targetName: defaultDs ,schemaName: db1, tableName: travelrecord2},
                      {targetName: defaultDs ,schemaName: db1, tableName: travelrecord3},
                      {targetName: repli ,schemaName: db2, tableName: travelrecord}] #9999999999
        address1:
          columns:
            - columnName: id
              shardingType: MAP_TARGET #动态分片类型,通过任意值映射分片目标,目标库,目标表,要求查询条件要求包列信息,否则可能路由失败,如果配置了dataNode,则会使用dataNode校验
              function: { clazz: io.mycat.router.function.PartitionConstant , properties: {defaultNode: '0'}} #映射到第一个元素,也就是指向map中的defaultDs
              map: [defaultDs]
            - columnName: addressname
              shardingType: MAP_DATABASE
              function: { clazz: io.mycat.router.function.PartitionConstant , properties: {defaultNode: '0'}}
              map: [db1]
            - columnName: addressname
              shardingType: MAP_TABLE
              function: { clazz: io.mycat.router.function.PartitionConstant , properties: {defaultNode: '0'}}
              map: [address]
          createTableSQL: CREATE TABLE `address` (`id` int(11) NOT NULL,`addressname` varchar(20) DEFAULT NULL,PRIMARY KEY (`id`))
      comany: #普通表
          columns:
           - columnName: id
             shardingType: NATURE_DATABASE_TABLE
             function: { clazz: io.mycat.router.function.PartitionConstant , properties: {defaultNode: '0'}} #映射到第一个元素
             dataNodes: [{targetName: defaultDs ,schemaName: db1, tableName: comany}]
         createTableSQL: CREATE TABLE `company` (`id` int(11) NOT NULL AUTO_INCREMENT,`companyname` varchar(20) DEFAULT NULL,`addressid` int(11) DEFAULT NULL,PRIMARY KEY (`id`)) ENGINE=InnoDB AUTO_INCREMENT=32 DEFAULT CHARSET=utf8mb4

```

shardingType类型MAP_TARGET,MAP_DATABASE,MAP_TABLE和map必须一起配置

当分片算法根据分片值计算出0的时候,也就是指向map中第一个元素,当分片算法根据分片值计算出1的时候,也就是指向map中第二个元素



NATURE_DATABASE_TABLE是单独配置,不与其他配置混合



## MetaData支持的SQL

##### 查询SQL

```yaml
query:
      values
  |   WITH withItem [ , withItem ]* query
  |   {
          select
      |   selectWithoutFrom
      |   query UNION [ ALL | DISTINCT ] query
      |   query EXCEPT [ ALL | DISTINCT ] query
      |   query MINUS [ ALL | DISTINCT ] query
      |   query INTERSECT [ ALL | DISTINCT ] query
      }
      [ ORDER BY orderItem [, orderItem ]* ]
      [ LIMIT [ start, ] { count | ALL } ]
      [ OFFSET start { ROW | ROWS } ]
      [ FETCH { FIRST | NEXT } [ count ] { ROW | ROWS } ONLY ]

withItem:
      name
      [ '(' column [, column ]* ')' ]
      AS '(' query ')'

orderItem:
      expression [ ASC | DESC ] [ NULLS FIRST | NULLS LAST ]

select:
      SELECT [ STREAM ] [ ALL | DISTINCT ]
          { * | projectItem [, projectItem ]* }
      FROM tableExpression
      [ WHERE booleanExpression ]
      [ GROUP BY { groupItem [, groupItem ]* } ]
      [ HAVING booleanExpression ]

selectWithoutFrom:
      SELECT [ ALL | DISTINCT ]
          { * | projectItem [, projectItem ]* }

projectItem:
      expression [ [ AS ] columnAlias ]
  |   tableAlias . *

tableExpression:
      tableReference [, tableReference ]*
  |   tableExpression [ NATURAL ] [ ( LEFT | RIGHT | FULL ) [ OUTER ] ] JOIN tableExpression [ joinCondition ]
  |   tableExpression CROSS JOIN tableExpression
  |   tableExpression [ CROSS | OUTER ] APPLY tableExpression

joinCondition:
      ON booleanExpression
  |   USING '(' column [, column ]* ')'

tableReference:
      tablePrimary
      [ FOR SYSTEM_TIME AS OF expression ]
      [ matchRecognize ]
      [ [ AS ] alias [ '(' columnAlias [, columnAlias ]* ')' ] ]

tablePrimary:
      [ [ catalogName . ] schemaName . ] tableName
      '(' TABLE [ [ catalogName . ] schemaName . ] tableName ')'
  |   tablePrimary [ EXTEND ] '(' columnDecl [, columnDecl ]* ')'
  |   [ LATERAL ] '(' query ')'
  |   UNNEST '(' expression ')' [ WITH ORDINALITY ]
  |   [ LATERAL ] TABLE '(' [ SPECIFIC ] functionName '(' expression [, expression ]* ')' ')'

columnDecl:
      column type [ NOT NULL ]

values:
      VALUES expression [, expression ]*

groupItem:
      expression
  |   '(' ')'
  |   '(' expression [, expression ]* ')'
  |   CUBE '(' expression [, expression ]* ')'
  |   ROLLUP '(' expression [, expression ]* ')'
  |   GROUPING SETS '(' groupItem [, groupItem ]* ')'

```



以下命令请使用之前使用explian查看执行计划,检查sql拆分是否正确

##### 插入SQL

```yaml
{sql: 'insert {any}',command: execute, tags: {executeType: INSERT,metaData: true,needTransaction: true }},
```



##### 更新SQL

```yaml
{sql: 'update {any}',command: execute,tags: {executeType: UPDATE,metaData: true ,needTransaction: true }},
```



##### 删除SQL

```yaml
{sql: 'delete {any}',command: execute,tags: {executeType: UPDATE,metaData: true,needTransaction: true  }}
```



## 拦截器配置



#### SQL匹配

```yaml
interceptor: #拦截器,如果拦截不了,尝试use schema,试用explain可以看到执行计划,查看路由
  defaultHanlder: {command: execute , tags: {targets: defaultDs,forceProxy: true}}
  schemas: [{
              tables:[ 'db1.travelrecord','db1.address1'],#sql中包含一个表名,就进入该匹配流程
              sqls: [
              {sql: 'select {selectItems} from {any}',command: distributedQuery },
              {sql: 'delete {any}',command: execute,....}
              ],
            },
            {
              tables:[ 'db1.company'],
              sqls: [
              {sql: 'select {selectItems} from {any}',command: execute ,....}
              ],
            },
  ]
  sqls: [
  {name: useStatement; ,sql: 'use {schema};',command: useStatement}
  ]
  transactionType: xa #xa.proxy
```

开启事务实现的方式,可以选择xa或者proxy

defaultHanlder

当匹配器无法匹配的时候,进入defaultHanlder处理流程

sqls中每项的sql是匹配的模式.

schemas中每项的tables,以 schema.table配置多个表名,当sql中出现此table,则进入匹配流程,如果sql中配置的匹配模式也匹配上,就执行命令.



#### 模式语法参考

https://github.com/MyCATApache/Mycat2/blob/master/doc/29-mycat-gpattern.md



#### 参数提取

sql中{name}是通配符,基于mysql的词法单元上匹配

同时把name保存在上下文中,作为命令的参数,所以命令的参数是可以从SQL中获得

```yaml
tags: {targets: defaultDs,forceProxy: true}
```

tags是配置文件中定下的命令参数

sql中的参数的优先级比tags高



#### SQL再生成

```yaml
  {name: 'mysql set names utf8', sql: 'SET NAMES {utf8}',explian: 'SET NAMES utf8mb4'  command: execute , tags: {targets: defaultDs,forceProxy: true}}
```

SQL被'SET NAMES utf8mb4'替换



```yaml
  {name: 'select n', sql: 'select {n}',explain: 'select {n}+1' command: execute , tags: {targets: defaultDs,forceProxy: true}},
```



拦截器会对use {schema}语句处理,得出不带schema的sql的table是属于哪一个schema.当mycat2发生错误的时候,会关闭连接,此时保存的schema失效,重新连接的时候请重新执行use schema语句



## 命令

##### explain

查看执行计划

```yaml
{name: explain,sql: 'EXPLAIN {statement}' ,command: explain}, #explain一定要写库名
```



##### commit

```yaml
{name: commit,sql: 'commit',command: commit},{name: commit;,sql: 'commit;',command: commit},
```



##### begin

```yaml
{name: begin; ,sql: 'begin',command: begin},{name: begin ,sql: 'begin;',command: begin},
```



##### rollback

```yaml
{name: rollback ,sql: 'rollback',command: rollback},{name: rollback;,sql: 'rollback;',command: rollback},
```



##### 切换数据库

```yaml
{name: useStatement ,sql: 'use {schema}',command: useStatement},
```



##### 开启XA事务

```yaml
{name: setXA ,sql: 'set xa = on',command: onXA},
```



##### 关闭XA事务

```yaml
{name: setProxy ,sql: 'set xa = off',command: offXA},
```



##### 关闭自动提交

关闭自动提交后,在下一次sql将会自动开启事务,并不会释放后端连接

```yaml
{name: setAutoCommitOff ,sql: 'set autocommit=off',command: setAutoCommitOff},
```



##### 开启自动提交

```yaml
{name: setAutoCommitOn ,sql: 'set autocommit=on',command: setAutoCommitOn},
```



##### 设置事务隔离级别

```
READ UNCOMMITTED,READ COMMITTED,REPEATABLE READ,SERIALIZABLE
```

```yaml
{name: setTransactionIsolation ,sql: 'SET SESSION TRANSACTION ISOLATION LEVEL {transactionIsolation}',command: setTransactionIsolation},
```



##### execute执行SQL

forceProxy:true|false

强制SQL以proxy上运行,忽略当前事务



needTransaction:true|false

根据上下文(关闭自动提交)自动开启事务



metaData:true|false

true的时候不需要配置targets,自动根据sql路由

false的时候要配置targets



targets

sql发送的目标:集群或者数据源的名字



balance

当targets是集群名字的时候生效,使用该负载均衡策略



executeType

QUERY执行查询语句,在proxy透传下支持多语句,否则不支持

QUERY_MASTER执行查询语句,当目标是集群的时候路由到主节点

INSERT执行插入语句

UPDATE执行其他的更新语句,例如delete,update,set



##### 返回error信息

```yaml
{command: error , tags: {errorMessage: "错误!",errorCode: -1}}
```



## 事务

XA事务使用基于JDBC数据源实现,具体请参考Java Transaction API

Proxy事务即通过Proxy操作MySQL进行事务操作,本质上与直接操作MySQL没有差异.

为了方便上层逻辑操作事务,所以统一JDBC和Proxy操作,,参考JDBC的接口,定下mycat2的事务接口
在proxy事务下,开启自动提交,没有事务,遇上需要跨分片的非查询操作,会自动升级为通过jdbc操作



## 数据源配置

```yaml
datasource:
  datasources: [{name: defaultDs, ip: 0.0.0.0,port: 3306,user: root,password: 123456,maxCon: 10000,minCon: 0,
   maxRetryCount: 1000000000, #连接重试次数
   maxConnectTimeout: 1000000000, #连接超时时间
   dbType: mysql, #
   url: 'jdbc:mysql://127.0.0.1:3306?useUnicode=true&serverTimezone=UTC',
   weight: 1, #负载均衡权重
   initSQL: 'use db1', #建立连接后执行的sql,在此可以写上use xxx初始化默认database
    jdbcDriverClass:, #jdbc驱动
   instanceType:,#READ,WRITE,READ_WRITE ,集群信息中是主节点,则默认为读写,副本则为读,此属性可以强制指定可写
  }
  ]
  datasourceProviderClass: io.mycat.datasource.jdbc.datasourceProvider.AtomikosDatasourceProvider
  timer: {initialDelay: 1000, period: 5, timeUnit: SECONDS}
```



maxConnectTimeout:单位millis

配置中的定时器主要作用是定时检查闲置连接



## 集群配置

```yaml
cluster: #集群,数据源选择器,既可以mycat自行检查数据源可用也可以通过mycat提供的外部接口设置设置数据源可用信息影响如何使用数据源
  close: true #关闭集群心跳,此时集群认为所有数据源都是可用的,可以通过mycat提供的外部接口设置数据源可用信息达到相同效果
  clusters: [
  {name: repli ,
   replicaType: SINGLE_NODE , # SINGLE_NODE:单一节点 ,MASTER_SLAVE:普通主从 GARELA_CLUSTER:garela cluster
   switchType: NOT_SWITCH , #NOT_SWITCH:不进行主从切换,SWITCH:进行主从切换
   readBalanceType: BALANCE_ALL  , #对于查询请求的负载均衡类型
   readBalanceName: BalanceRoundRobin , #对于查询请求的负载均衡类型
   writeBalanceName: BalanceRoundRobin ,  #对于修改请求的负载均衡类型
   masters:[defaultDs], #主节点列表
   replicas:[defaultDs2],#从节点列表
   heartbeat:{maxRetry: 3, #心跳重试次数
              minSwitchTimeInterval: 12000 , #最小主从切换间隔
              heartbeatTimeout: 12000 , #心跳超时值,毫秒
              slaveThreshold: 0 , # mysql binlog延迟值
              reuqestType: 'mysql' #进行心跳的方式,mysql或者jdbc两种
   }}
  ]
  timer: {initialDelay: 1000, period: 5, timeUnit: SECONDS} #心跳定时器
```

只有GARELA_CLUSTER能在masters属性配置多个数据源的名字

reuqestType是进行心跳的实现方式,使用mysql意味着使用proxy方式进行,能异步地进行心跳,而jdbc方式会占用线程池



## 服务器配置

```yaml
server:
  ip: 0.0.0.0
  port: 8066
  reactorNumber: 1
  #用于多线程任务的线程池,
  worker: {close: false, #禁用多线程池,jdbc等功能将不能使用
           maxPengdingLimit: 65535, #每个线程处理任务队列的最大长度
           maxThread: 2,
           minThread: 2,
           timeUnit: SECONDS, #超时单位
           waitTaskTimeout: 5 #超时后将结束闲置的线程
  }
```



## 分片算法配置

```yaml
function: { clazz: io.mycat.router.function.PartitionByLong , name: partitionByLong, properties: {partitionCount: '4', partitionLength: '256'}, ranges: {}}
```

具体参考以下链接

https://github.com/MyCATApache/Mycat2/blob/master/doc/17-partitioning-algorithm.md



## 负载均衡配置

```yaml
plug:
  loadBalance:
    defaultLoadBalance: balanceRandom
    loadBalances: [
    {name: BalanceRunOnMaster, clazz: io.mycat.plug.loadBalance.BalanceRunOnMaster},
    {name: BalanceLeastActive, clazz: io.mycat.plug.loadBalance.BalanceLeastActive},
    {name: BalanceRoundRobin, clazz: io.mycat.plug.loadBalance.BalanceRoundRobin},
    {name: BalanceRunOnMaster, clazz: io.mycat.plug.loadBalance.BalanceRunOnMaster},
    {name: BalanceRunOnRandomMaster, clazz: io.mycat.plug.loadBalance.BalanceRunOnRandomMaster}
    ]
```

具体参考以下链接

https://github.com/MyCATApache/Mycat2/blob/master/doc/16-load-balancing-algorithm.md