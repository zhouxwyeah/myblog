ES

####倒排索引

doc id

分词 term

term关联  post list

term dict  排序 二分查找  偏于查找 term,查找读磁盘 【任意byte数组】

term index  tier树      可加载到内存，快速定位到term的 offset区间，然后查找term dict，减少IO次数



mysql 只有term dict，查找的磁盘IO次数大于 ES，ES多出了term index二级索引，减少随机读的次数



term dict还有公共前缀压缩，可以减少空间



####联合索引

mysql必须建立

ES 支持单索引自动合并 AND OR 操作

skip list（Frame of Reference 压缩）   bitmap(Roaring Bitmap压缩)

 Frame of Reference 编码是如此 高效，对于简单的相等条件的过滤缓存成纯内存的 bitset 还不如需要访问磁盘的 skip list 的方式要快。



####减少文档数

将压缩存储序列压缩成一行，分行-列表 ，减少索引尺寸

父子文档内嵌

使用了嵌套文档之后，对于 term 的 posting list 只需要保存父文档的 doc id 就可以了，可以比保存所有的数据点的 doc id 要少很多

#### LOAD加载

非索引 要顺序扫描真个存储

索引   可能导致大量碎片的随机读



没有所谓完美的解决方案。MySQL 支持索引，一般索引检索出来的行数也就是在 1~100 条之间。如果索引检索出来很多行，很有可能 MySQL 会选择不使用索引而直接扫描主存储，这就是因为用 row id 去主存储里读取行的内容是碎片化的随机读操作，这在普通磁盘上很慢。

大量顺序读 不一定比很少的随机读慢



Elasticsearch/Lucene 的解决办法是让主存储的随机读操作变得很快，从而可以充分利用索引，而不用惧怕从主存储里随机读加载几百万行带来的代价



#### 分布式计算

Elasticsearch/Lucene 从最底层就支持数据分片，查询的时候可以自动把不同分片的查询结果合并起来。Elasticsearch 的 document 都有一个 uid，默认策略是按照 uid 的 hash 把文档进行分片。













