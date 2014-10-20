---
title: Hbase模式（schema）设计时的tips
date: 2014-07-30 16:00:00 +0800
tags:
- hbase
---

### 表的属性

- 最大版本数：通常是3，如果对于更新比较频繁的应用完全可以设置为1，能够快速的淘汰无用数据，对于节省存储空间和提高查询速度有效果。不过这类需求在海量数据领域比较小众。
- 压缩算法：可以尝试一下最新出炉的snappy算法，相对lzo来说，压缩率接近，压缩效率稍高，解压效率高很多。
- inmemory：表在内存中存放，一直会被忽略的属性。如果完全将数据存放在内存中，那么hbase和现在流行的内存数据库memorycached和redis性能差距有多少，尚待实测。
- bloomfilter：根据应用来定，看需要精确到rowkey还是column。不过这里需要理解一下原理，bloomfilter的作用是对一个region下查找记录所在的hfile有用。即如果一个region下的hfile数量很多，bloomfilter的作用越明显。适合那种compaction赶不上flush速度的应用。

## 列族的数量及列族的势

建议HBASE列族的数量设置的越少越好。由于HBASE的FLUSHING和压缩是基于REGION的当一个列族所存储的数据达到FLUSHING阀值时，该表的所有列族将同时进行FLASHING操作，这将带来不必要的I/O开销。同时还要考虑到同一个表中不同列族所存储的记录数量的差别，即列族的势。当列族数量差别过大将会使包含记录数量较少的列族的数据分散在多个Region之上，而Region可能是分布是不同的RegionServer上。这样当进行查询等操作系统的效率会受到一定影响。






### rowkey 行键

- 应避免使用时序或单调行键。因为当数据到来时，HBASE首先需要根据记录的行键来确定存储位置，即Region的位置。如果使用时序或单调行建，那么连续到来的数据将会被分配到同一个Region当中，而此时系统化中的其他Region/RegionServer将处于空闲状态，这是分布式系统最不希望看到的。
- 数字rowkey的从大到小排序：原生hbase只支持从小到大的排序，这样就对于排行榜一类的查询需求很尴尬。那么采用rowkey = Integer.MAX_VALUE-rowkey的方式将rowkey进行转换，最大的变最小，最小的变最大。在应用层再转回来即可完成排序需求。
- rowkey的散列原则：如果rowkey是类似时间戳的方式递增的生成，建议不要使用正序直接写入rowkey，而是采用reverse的方式反转rowkey，使得rowkey大致均衡分布，这样设计有个好处是能将regionserver的负载均衡，否则容易产生所有新数据都在一个regionserver上堆积的现象，这一点还可以结合table的预切分一起设计。

### 尽量最小化行键和列族的大小

Hbase中一条记录是由存储该值的行键，对应的列以及该值的时间戳决定。HBase 中索引是为了加速随机访问的速度。该索引的创建是基于“行键+列族：列+时间戳+值”的，如果行键和
列族的大小过大，将会增加索引的大小，加重系统的存储负担。

### 版本数量

HBase在进行数据存储时，新数据不会直接覆盖旧的数据，而是进行追加操作，不同的数据通过时间戳进行区分。默认每行数据存储三个版本，建议不要将其设置过大。

### Compact & Split

在HBase中，数据在更新时首先写入WAL 日志(HLog)和内存(MemStore)中，MemStore中的数据是排序的，当MemStore累计到一定阈值时，就会创建一个新的MemStore，并且将老的MemStore添加到flush队列，由单独的线程flush到磁盘上，成为一个StoreFile。于此同时， 系统会在zookeeper中记录一个redo point，表示这个时刻之前的变更已经持久化了(minor compact)。

StoreFile是只读的，一旦创建后就不可以再修改。因此Hbase的更新其实是不断追加的操作。当一个Store中的StoreFile达到一定的阈值后，就会进行一次合并(major compact)，将对同一个key的修改合并到一起，形成一个大的StoreFile，当StoreFile的大小达到一定阈值后，又会对 StoreFile进行分割(split)，等分为两个StoreFile。

由于对表的更新是不断追加的，处理读请求时，需要访问Store中全部的StoreFile和MemStore，将它们按照row key进行合并，由于StoreFile和MemStore都是经过排序的，并且StoreFile带有内存中索引，通常合并过程还是比较快的。

实际应用中，可以考虑必要时手动进行major compact，将同一个row key的修改进行合并形成一个大的StoreFile。同时，可以将StoreFile设置大些，减少split的发生。

### 存储结构

对于HBase的存储设计，要考虑它的存储结构是：rowkey+columnFamily:columnQualifier+timestamp(version)+value = KeyValue in HBase，一个KeyValue依次按照rowkey，columnkey和timestamp有序。一个rowkey加一个column信息定位了hbase表的一个逻辑的行结构。

从逻辑存储结构到实际的物理存储结构要经历一个fold过程，所有的columnFamily下的内容被有序的合并，因为HBase把一个ColumnFamily存储为一个StoreFile。

### 模式趋向单一，建议使用tall narrow模式

把HBase的查询等价为一个逐层过滤的行为，那么在设计存储时就应该明白，使设计越趋向单一的keyvalue性能会越好；如果是因为复杂的业务逻辑导致查询需要确定rowkey、column、timestamp，甚至更夸张的是用到了HBase的Filter在server端做value的处理，那么整个性能会非常低。

因此在表结构设计时，HBase里有tall narrow和flat wide两种设计模式，前者行多列少，整个表结构高且窄；后者行少列多，表结构平且宽；但是由于HBase只能在行的边界做split，因此如果选择flat wide的结构，那么在特殊行变的超级大（超过file或region的上限）时，那么这种行为会导致compaction，而这样做是要把row读内存的~~因此，强烈推荐使用tall narrow模式设计表结构，这样结构更趋近于key value，性能更好。

### partial row scan

一种优雅的行设计叫做partial row scan，我们一般rowkey会设计为--...，每个key都是查询条件，中间用某种分隔符分开，对于只想查key1的所有这样的情况，在不使用filter的情况下（更高性能），我们可以为每个key设定一个起始和结束的值，比如key1作为开始，key1+1作为结束，这样scan的时候可以通过设定start row和stop row就能查到所有的key1的value，同理迭代，每个子key都可以这样被设计到rowkey中。

### 分页

对于分页查询，推荐的设计方式也不是利用filter，而是在scan中通过offset和limit的设定来模拟类似RDBMS的分页。具体过程就是首先定位start row，接着跳过offset行，读取limit行，最后关闭scan，整个流程结束。

### 时间范围查询

对于带有时间范围的查询，一种设计是把时间放到一个key的位置，这样设计有个弊端就是查询时一定要先知道查询哪个维度的时间范围值，而不能直接通过时间查询所有维度的值；另一种设计是把timestamp放到前面，同时利用hashcode或者MD5这样的形式将其打散，这样对于实时的时序数据，因为将其打散导致自动分到其他region可以提供更好的并发写优势。

### 利用column达到类似二级索引的效果

一种高级的设计方式是利用column来当做RDBMS类似二级索引的应用设计，rowkey的存储达到一定程度后，利用column的有序，完成类似索引的设计，比如，一个CF(Column Family)叫做data存放数据本身，Column Qualifier是一个MD5形式的index，而value是实际的数据；再建一个CF叫做index存储刚才的MD5，这个index的CF的ColumnQualifier是真正的索引字段（比如名字或者任意的表字段，这样可以允许多个），而value是这个索引字段的MD5。每次查询时就可以先在index里找到这个索引（查询条件不同，选择的索引字段不同），然后利用这个索引到data里找到数据，两次查询实现真正的复杂条件业务查询。

实现二级索引还有其他途径，比如：

1. 客户端控制，即一次读取将所有数据取回，在客户端做各种过滤操作，优点自然是控制力比较强，但是缺点在性能和一致性的保证上；
2. Indexed-Transactional HBase，这是个开源项目，扩展了HBase，在客户端和服务端加入了扩展实现了事务和二级索引；
3. Indexed-HBase；
4. Coprocessor。

### 集成搜索

HBase集成搜索的方式有多种：

- 客户端控制，同上；
- Lucene；
- HBasene，
- Coprocessor。

### 集成事务

HBase集成事务的方式：

- ITHBase；
- ZooKeeper，通过分布式锁。

### timestamp
timestamp虽然叫这个名字，但是完全可以存放任何内容来形成用户自定义的版本信息。
