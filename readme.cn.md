在广告投放中,广告主需要通过实时的投放数据来优化投放策略.

我们在测试环境使用了`druid.io`一段时间, 发现一些问题

#### druid在处理数据更新时很笨重
这个在广告投放中很常见,比如二跳/转化的数据往往延迟于点击很长时间, 在更新时,需要整个进行Reindex ,这个需要很长时间.
并且我们不可能对每一条二跳/转化日志的日志都进行Reindex.
> 使用hive的分区,将曝光/点击数据保存在logtype='ic'的分区中,二跳/转化的数据保存在logtype='jc'中,这样聚合时只需要处理logtype='ic'中的,同时由于logtype='jc'的数据很小,往往不需要额外处理~

#### druid在处理非常多维度的数据实时ingest时,结果很差
广告日志中往往需要处理60+dimension,20+metric的数据,druid在内存中会进行group,很容易引发GC,造成同ZK的消息中断,导致不断的Reindex
> 测试证明,对于实时的数据,我们不需要进行直接的聚合操作,已经可以通过presto的分布式查询来得到满意的结果, 这样聚合的粒度我们可以放宽到小时的级别

#### druid不支持sql
sql对于大多数数据分析师来说是更好的选择
> presto 支持

抛开目前的流行框架(eg:kylin,pinot,druid), 目前流行的各种数据仓库的设计其实都可以用下面的一句话来概括
> 对高可用的文件系统上高效存储的数据进行分布式查询操作

问题就变的的很简单了

1. ORC,Parquet在列存储上已经是很好的实现,不需要再额外实现新的文件格式
2. HDFS已经是被证明的高可用文件存储方案,不需要额外自己调度实现
3. Presto是非常优秀的查询引擎,做了很多查询优化
4. Hive Sql可以很方便的GroupBy来对数据进行聚合操作,`HadoopIndexer`完全没有必要 ; 同时hive sql非常灵活,可以在建立多个cube来加速查询
5. 

问题就变的非常简单
> 对高可用的文件系统上(hdfs)高效存储的(orc)数据进行分布式查询(presto)操作

实际需要做的事情就变成了: 生成ORC文件放到hdfs上~

同时为了查询效率,每隔一段时间,从之前的数据进行聚合操作 
```sql
insert overwite my_orc_table partition(pday =20151201 , phour = 4) select a,b,sum(imp),sum(click) from my_orc_table where log_type='ic' and pday =20151201 and phour = 4
```

1. source(kafka) -> orc file -> hdfs
2. crontab 


于是,[eaux](https://github.com/zhenchuan/eaux)就产生了,
https://github.com/zhenchuan/eaux


---------


目前流行的各种数据仓库的设计都可以用下面的一句话来概括
> 对高可用的文件系统上高效存储的数据进行分布式查询操作

作为分布式查询引擎,Presto已经可以通过`hive connector`处理位于hdfs上的数据 , 
剩下的步骤就是生成文件到HDFS了~
这里选择的是`ORC`格式的文件存储.

所以上面的定义就变成了
> 对高可用的文件系统上(hdfs)高效存储的(orc)数据进行分布式查询(presto)操作

`eaux` 的作用就是从数据源中读取文件,然后生成`ORC`格式的文件, 上传到HDFS中~

在实际应用中,有几个问题需要解决
1. 何时生成本地的`ORC`文件. 
```
`eaux`中提供了以下配置用于生成`ORC`文件
1. `rotation.period` , PT10m表示每隔10 minutes生成一个文件. 如果需要更实时的数据,可以考虑设置更低的值
2. `rotation.gran` , hour ,表示在hour的临界点,强制生成.例如12.58 -> 13.00
3. `rotation.file.size.mb` , 在用户配置使用`CommitLog`时,如果commitLog的大小超过设置的值,会生成文件.
```
2. 需要根据不同的条件来生成不同的`ORC`文件
```
`eaux`提供了`GroupFileWriter`来提供自定义逻辑. 

同时,在根据日志中的时间(非系统时间)进行rotate时,需要处理关闭
```
3. 由于hive提供的`orc file wirter`是在内存中处理生成的,需要保证在持久化到磁盘之前数据不会因为程序终止而造成数据丢失
```
`eaux`采用了`CommitLog`的机制,来保证数据的写入安全. 如果在生成ORC文件失败时,会从CommitLog中恢复~
```


## [example](https://github.com/zhenchuan/eaux/tree/master/example)


### 创建hive表
根据应用的特点,创建数据表

#### 创建表`eaux_grouped` 
使用日志中的日期字段分区,同时对广告主按`adv_id %2 ==0`进行分组~

```sql
CREATE  TABLE eaux_grouped(
  day INT ,
  hour INT ,
  id_advertiser_id INT ,
  action_platform STRING ,
  bid_policy_data STRING ,
  bid_rank_data STRING ,
  imp BIGINT ,
  click BIGINT
) partitioned by (pDay int,pHour int,adv int) stored as orc ;
```

#### 创建表eaux_plain
使用系统时间创建分区
```sql
CREATE  TABLE eaux_plain(
  day INT ,
  hour INT ,
  id_advertiser_id INT ,
  action_platform STRING ,
  imp BIGINT ,
  click BIGINT
) partitioned by (pDay int,pHour int,adv int) stored as orc ;
```
创建完数据表后,其hdfs为 
`/apps/hive/warehouse/eaux_grouped` 和 `/apps/hive/warehouse/eaux_plain` , 我们只需要把生成的`ORC`格式的文件按照分区方式放进来就可以了~

### 创建

#### 构造`GroupedHdfsSink`,用于写入`eaux_grouped`
```java
//sink的配置
Configuration groupedConfiguration = new PropertiesConfiguration("report.properties");
....
//用于保存本地生成的文件
FileRegistry groupedFileRegistry = new FileRegistry() ;
//确定分组条件,构建本地路径和hdfs的路径
FileNameGenerator groupedFileNameGenerator = new DateAdvertiserNameGenerator(groupedConfiguration) ;
final RecoverableWriter groupFileWriter = new GroupFileWriter(groupedConfiguration,
                groupedFileRegistry,
                groupedFileNameGenerator);
//恢复上次关闭时没有来的及处理的文件,必须在设置使用`CommitLog`时才生效                
groupFileWriter.recoverWith(null,null);
//用于上传文件
UploadService groupedUploadService = new AsyncUploadService("grouped",groupedFileRegistry,
                groupedFileNameGenerator,groupedConfiguration);
//构建Sink                
HdfsSink groupedHdfsSink = new GroupedHdfsSink(groupedConfiguration,groupFileWriter,groupedUploadService);
```
`FileNameGenerator`接口用于自定义文件的本地路径和hdfs路径, 最好将分组方式也放到其实现类中~

如果设置了使用`CommitLog`,  `GroupFileWriter`在会自动恢复相关数据,不用再额外传入相关参数, 所以这里的参数都是null,`groupFileWriter.recoverWith(null,null)`

#### 构造`PlainHdfsSink`,用于写入`eaux_plain`
```java
//plain sink writer
FileRegistry plainFileRegistry = new FileRegistry() ;
Configuration plainConfiguration = new PropertiesConfiguration("plain.properties");
...
HiveTableFileNameGenerator plainNameGenerator = new PlainNameGenerator(plainConfiguration);
//当参数commitLog为null时,表示不使用`CommitLog`
final RecoverableWriter plainWriter = new FileWriter(plainConfiguration, plainFileRegistry,
                plainNameGenerator,null);  //without commit log
UploadService plainUploadService = new AsyncUploadService("plain",plainFileRegistry,
                plainNameGenerator,plainConfiguration);
HdfsSink plainSink = new PlainHdfsSink(plainConfiguration,plainWriter,plainUploadService) ;
```

目前支持提供了 `kafka`,`rocketmq`,`file` 三种数据源,你也可以自己扩展`Source`来实现~
```java
//kafka source
Configuration inputConfiguration = new PropertiesConfiguration("input.kafka.properties");
final TestKafkaSource kafkaSource = new TestKafkaSource(inputConfiguration);
kafkaSource.via(Lists.newArrayList(groupedHdfsSink,plainSink)).start();
```






### 配置
#### kafka配置
key | value
---|---
group.id |  MyConsumerGroup 
zookeeper.connect |192.168.146.69:2181/kafka
zookeeper.session.timeout.ms | 150000
zookeeper.connection.timeout.ms | 120000
auto.offset.reset | largest
consumer.timeout.ms | 60000
socket.receive.buffer.bytes | 1048576
fetch.message.max.bytes | 1048576
consumerTopics | 格式: topic:thread_num,多个之间用 `,` 分割 例 imp:2,click:1

#### sink 配置
key | value
---|---
name | nrt_orc
work.dir | /tmp/multi/
hdfs.upload | false/true ##是否上传到hdfs 
hdfs.resources | /tmp/core-site.xml,/tmp/hdfs-site.xml
rotation.period | PT5m 
rotation.file.size.mb | 60
rotation.gran | hour/day
writer.parallel | 2
*commitlog.enable | true/false  ##是否对每个`group`都使用CommitLog
*segment.expiry.check.interval.seconds | 76  ##定期检查`group`是否有数据更新
*segment.force.persist.interval.rp | 5   ##如果`group` 在 `5 * 76` 内没有新的数据,就强制`close`并生成orc文件

注意: `*`开头的配置用于`GroupFileWriter`

#### graphite 配置
key | value
---|---
graphite.prefix | eaux144126
graphite.host | 192.168.152.21
graphite.port | 2003
graphite.poll.interval.seconds | 60

