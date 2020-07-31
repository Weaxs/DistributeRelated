# 分布式ID生成器

## UUID

优点：本地生成，足够简单，不用基于数据库且具有唯一性

缺点：长度过长，16字节128位，36位的字符串，做为主键存储以及查询的性能消耗较大；且UUID不具有有序性，会导致B+树索引在写的时候有过多的随机写操作 (连续的ID可以产生部分顺序写)，
还有，在写的时候不能产生有顺序的append操作，而需要进行insert操作，会读取整个B+树节点到内存，在插入这条记录后将整个节点回写磁盘，这种操作在记录占用空间比较大的情况下，性能下降明显。

适用场景：**文件名，编号等可以使用，但主键不能用

## 数据库自增ID

优点：实现简单，ID单调自增，数值类型查询速度快

缺点：每次插入时，都需要select last_insert_id()，导致性能低；同时当访问量激增的时候，数据库本身就是瓶颈，数据库宕机下线，会影响所有的业务系统，无法扛住高并发场景。

## 数据库多主(集群)模式

前面提到单点数据库的方式不可取，我们可以尝试主从模式集群，正常情况下可以解决数据库可靠性问题，但是当主库挂掉后，数据如果没有及时同步到从库，这个时候会出现ID重复的情况。

此时我们可以改成多主集群模式，也就是每个MySQL实例都能单独的产生自增ID。但同样多个Mysql实例会产生重复ID，解决方案：设置**起始值**和**自增步长**。例如：目前我有两个Mysql实例，一个自增偶数，一个自增奇数

对于这种生成分布式ID的方案，需要单独新增一个生成分布式ID应用，比如DistributIdService，该应用提供一个接口供业务应用获取ID，业务应用需要一个ID时，通过rpc的方式请求DistributIdService，DistributIdService随机去上面的两个Mysql实例中去获取ID。
实行这种方案后，就算其中某一台Mysql实例下线了，也不会影响DistributIdService，DistributIdService仍然可以利用另外一台Mysql来生成ID。

但是多主(集群)模式的扩容性很麻烦，当我们需要新增一个MySQL实例时，有可能需要修改所有MySQL实例的起始值和自增步长，同时对于新增实例的起始值需要根据当前所有实例ID的最大值判断

优点：解决了DB单点的问题

缺点：不利于后续扩容，且单个数据库自身压力依然大，无法满足高并发场景

## 数据库号段模式

号段模式可以理解为从数据库批量的获取自增ID，每次从数据库取出一个号段范围。

    CREATE TABLE id_generator (
        id int(10) NOT NULL,
        max_id bigint(20) NOT NULL COMMENT '当前最大id',
        step int(20) NOT NULL COMMENT '号段的布长',
        biz_type    int(20) NOT NULL COMMENT '业务类型',
        version int(20) NOT NULL COMMENT '版本号',
        PRIMARY KEY (`id`)
    ) ENGINE=InnoDB DEFAULT CHARSET=utf8;
    

比如DistributIdService从数据库获取ID时，如果能批量获取多个ID并缓存在本地的话，那样将大大提供业务应用获取ID的效率。
当这批号段ID用完，再次向数据库申请新号段，对max_id字段做一次update操作，update max_id= max_id + step，update成功则说明新号段获取成功，新的号段范围是(max_id ,max_id +step]。

这种方案不再强依赖数据库，就算数据库不可用，那么DistributIdService也能继续支撑一段时间。但是如果DistributIdService重启，会丢失一段ID，导致ID空洞。

为了提高DistributIdService的高可用，需要做一个集群，业务在请求DistributIdService集群获取ID时，会随机的选择某一个DistributIdService节点进行获取，对每一个DistributIdService节点来说，数据库连接的是同一个数据库，那么可能会产生多个DistributIdService节点同时请求数据库获取号段，那么这个时候需要利用乐观锁来进行控制，所以采用版本号version方式更新。

## 基于Redis模式

利用redis的 incr命令实现ID的原子性自增

但是在Redis宕机后可能会产生一下影响：
1. RDB会定时打一个快照进行持久化，假如连续自增但redis没及时持久化，而这会Redis挂掉了，重启Redis后会出现ID重复的情况
2. AOF会对每条写命令进行持久化，即使Redis挂掉了也不会出现ID重复的情况，但由于incr命令的特殊性，会导致Redis重启恢复的数据时间过长

## twitter的雪花 (Snowflake) 算法

Snowflake生成的是Long类型的ID，一个Long类型占8个字节，每个字节占8比特，也就是说一个Long类型占64个比特。

Snowflake ID组成结构：正数位（占1比特）+ 时间戳（占41比特）+ 机器ID（占5比特）+ 数据中心（占5比特）+ 自增值（占12比特），总共64比特组成的一个Long类型。

* 第一个bit位（1bit）：Java中long的最高位是符号位代表正负，正数是0，负数是1，一般生成ID都为正数，所以默认为0。
* 时间戳部分（41bit）：毫秒级的时间，不建议存当前时间戳，而是用（当前时间戳 - 固定开始时间戳）的差值，可以使产生的ID从更小的值开始；41位的时间戳可以使用69年，(1L << 41) / (1000L * 60 * 60 * 24 * 365) = 69年
* 工作机器id（10bit）：也被叫做workId，这个可以灵活配置，机房或者机器号组合都可以。
* 序列号部分（12bit），自增值支持同一毫秒内同一个节点可以生成4096个ID

根据这个算法的逻辑，只需要将这个算法用编程语言实现出来，封装为一个工具方法，那么各个业务应用可以直接使用该工具方法来获取分布式ID，只需保证每个业务应用有自己的工作机器id即可，而不需要单独去搭建一个获取分布式ID的应用。

在大厂里，其实并没有直接使用snowflake，而是进行了改造，因为snowflake算法中最难实践的就是工作机器id，原始的snowflake算法需要人工去为每台机器去指定一个机器id，并配置在某个地方从而让snowflake从此处获取机器id。
但是在大厂里，机器是很多的，人力成本太大且容易出错，所以大厂对snowflake进行了改造。

雪花算法存在的问题：
1. **时钟回拨：**雪花算法是强依赖时间戳的，如果时间发生回拨，有可能会生成重复的ID。
    解决方式：我们可以将生成过的ID存起来；存成过期时间；当发现时钟回拨的时间很短，也可以进行等待；如果时间回拨的时间较长，也可以抛错等处理。
2. **机器id的分配与回收：**机器id的分配的工作量比较大，且在服务器宕机或者扩容中，会存在机器id的分配和回收问题

## 百度的 uid-generator 算法

uid-generator使用的就是snowflake，只是在生产机器id，也叫做workId时有所不同。

uid-generator中的workId是由uid-generator自动生成的，并且考虑到了应用部署在docker上的情况，在uid-generator中用户可以自己去定义workId的生成策略，默认提供的策略是：
      
    应用启动时由数据库分配。
    即应用在启动时会往数据库表(uid-generator需要新增一个WORKER_NODE表)中去插入一条数据，数据插入成功后返回的该数据对应的自增唯一id就是该机器的workId
    数据由host，port组成。
    对于uid-generator中的workId，占用了22个bit位，时间占用了28个bit位，序列化占用了13个bit位
    需要注意的是，和原始的snowflake不太一样，时间的单位是秒，而不是毫秒，workId也不一样，同一个应用每重启一次就会消费一个workId。

## 美团的 Leaf 算法

美团的Leaf也是一个分布式ID生成框架。它非常全面，即支持号段模式，也支持snowflake模式。

Leaf中的snowflake模式和原始snowflake算法的不同点，也主要在workId的生成，Leaf中workId是基于ZooKeeper的顺序Id来生成的，每个应用在使用Leaf-snowflake时，在启动时都会都在Zookeeper中生成一个顺序Id，相当于一台机器对应一个顺序节点，也就是一个workId。

## 滴滴的 Tinyid 算法

Tinyid是基于号段模式原理实现，官方简介说类似与美团的Leaf

Tinyid扩展了leaf-segment算法，支持了多db(master)，同时提供了java-client(sdk)使id生成本地化，获得了更好的性能与可用性。
