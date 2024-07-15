<div align="center">
    <h1>
        🔎ElasticSearch八股整理
    </h1>
</div>


**ES整体存储架构图**

![img](https://cdn.jsdelivr.net/gh/WangMinan/Pics/20210618035834144.png)



## 倒排索引

### 定义

#### 正排索引与倒排索引

- 正排索引：文档ID到文档内容和单词的关联

- 倒排索引：单词到文档ID的关系

  一种索引方法，被用来存储在全文搜索下某个单词在某一个文档或者一组文档中的存储位置的映射。也可以称他为反向索引。

  备注：ES对文档每个字段都有自己的倒排索引，可以指定某些字段不做索引，这样可以节省存储空间，缺点是这个字段无法被搜索。

  - **Term**：ES为每一个字段都建立了一个倒排索引，而每个字段下的值叫做Term。
  - **Posting List**：一个int类型的数组，存储了所有符合某一个Term的文档的Id，也可以理解为位置集合

  基本上会有三种文件去存储某一条数据：

  1. Term Dictionary 词典文件，存储一些分词的前缀
  2. frequencies 词频文件，存储这个分词出现的频率
  3. positions 位置文件，存储这个分词出现的位置



#### Term Dictionary和Term Index

**Term Dictionary：**
ES为了快速定位到某一个Term，会将所有的Term进行一个排序，需要查询的时候，用二分法的方式去查找。形如字典里先查偏旁
**Term Index：**
ES采用与B-Tree一样的想法，用内存去查找Term（快），而不是磁盘。Term Index像一棵树，包含了Term所在的地址。
形象的说，查询从Term Dictionary中查找偏旁，查到对应的偏旁后，可以看到这个偏旁下，有哪些字，分别在哪一页，这就是Term Index。

![在这里插入图片描述](https://cdn.jsdelivr.net/gh/WangMinan/Pics/20200711183647210.png)

因此这里说明一个很重要的点，就是Term Index不是存储所有的Term，而是**存储一个Term Dictionary的一个映射关系**。比如以字母A开头的存在哪些分词，在哪些位置，而Index存储的就是这些位置。



### 官方文档是这么写的

Elasticsearch 使用一种称为 ***倒排索引*** 的结构，它适用于快速的全文搜索。一个倒排索引由文档中所有不重复词的列表构成，对于其中每个词，有一个包含它的文档列表。

例如，假设我们有两个文档，每个文档的 `content` 域包含如下内容：

1. The quick brown fox jumped over the lazy dog (太经典了)
2. Quick brown foxes leap over lazy dogs in summer

为了创建倒排索引，我们首先将每个文档的 `content` 域拆分成单独的词（我们称它为 `词条` 或 `tokens` ），创建一个包含所有不重复词条的排序列表，然后列出每个词条出现在哪个文档。结果如下所示：

```
Term      Doc_1  Doc_2
-------------------------
Quick   |       |  X
The     |   X   |
brown   |   X   |  X
dog     |   X   |
dogs    |       |  X
fox     |   X   |
foxes   |       |  X
in      |       |  X
jumped  |   X   |
lazy    |   X   |  X
leap    |       |  X
over    |   X   |  X
quick   |   X   |
summer  |       |  X
the     |   X   |
------------------------
```

现在，如果我们想搜索 `quick brown` ，我们只需要查找包含每个词条的文档：

```
Term      Doc_1  Doc_2
-------------------------
brown   |   X   |  X
quick   |   X   |
------------------------
Total   |   2   |  1
```

两个文档都匹配，但是**第一个文档比第二个匹配度更高**。如果我们使用仅计算匹配词条数量的简单 ***相似性算法*** ，那么，我们可以说，对于我们查询的相关性来讲，第一个文档比第二个文档更佳。

但是，我们目前的倒排索引有一些问题：

- `Quick` 和 `quick` 以独立的词条出现，然而用户可能认为它们是相同的词。
- `fox` 和 `foxes` 非常相似, 就像 `dog` 和 `dogs` ；他们有相同的词根。
- `jumped` 和 `leap`, 尽管没有相同的词根，但他们的意思很相近。他们是同义词。

使用前面的索引搜索 `+Quick +fox` 不会得到任何匹配文档。（记住，`+` 前缀表明这个词**必须存在**。）只有同时出现 `Quick` 和 `fox` 的文档才满足这个查询条件，但是第一个文档包含 `quick fox` ，第二个文档包含 `Quick foxes` 。

我们的用户可以合理的期望两个文档与查询匹配。我们可以做的更好。

如果我们将词条规范为标准模式，那么我们可以找到与用户搜索的词条不完全一致，但具有足够相关性的文档。例如：

- `Quick` 可以小写化为 `quick` 。
- `foxes` 可以 *词干提取* --变为词根的格式-- 为 `fox` 。类似的， `dogs` 可以为提取为 `dog` 。
- `jumped` 和 `leap` 是同义词，可以索引为相同的单词 `jump` 。

现在索引看上去像这样：

```
Term      Doc_1  Doc_2
-------------------------
brown   |   X   |  X
dog     |   X   |  X
fox     |   X   |  X
in      |       |  X
jump    |   X   |  X
lazy    |   X   |  X
over    |   X   |  X
quick   |   X   |  X
summer  |       |  X
the     |   X   |  X
------------------------
```

这还远远不够。我们搜索 `+Quick +fox` *仍然* 会失败，因为在我们的索引中，已经没有 `Quick` 了。但是，如果我们对搜索的字符串使用与 `content` 域相同的标准化规则，会变成查询 `+quick +fox` ，这样两个文档都会匹配！



### 接下来我们做一点拓展

#### 倒排索引不可变性

倒排索引采用Immutable Design，**一旦生成，不可更改**。

- 优点
   （1）无需考虑并发写文件的问题，避免了锁机制带来的性能问题
   （2）一旦读入内核的文件系统缓存，便留在那里。只要文件系统存有足够大的空间。大部分请求就会直接请求内存，不会命中磁盘，提升了很大的性能
- 缺点
   不可变性也带来了另一个挑战，如果需要让一个新的文档可以被索引，需要重建整个索引。

在Lucene中，单个倒排索引文件被称作**Segment**。Segment是**不可变更**的。多个Segment汇总在一起，称为Lucene的Index，其对应的就是**ES中的Shard**

当有新文档写入时，会生成新Segment，查找时**会同时查找多个Segments，并且对结果进行汇总**。Lucene中有个Commit Point文件，用来记录所有Segments信息。

删除文档的信息，会保存在.del文件中。在搜索时，返回的结果会根据.del中记录的文档信息，将其过滤掉。



#### ES索引文档的过程

1. 请求发送到Coordinating Node，如果该节点不是Master节点，需要将该请求转发到Master.

   Master节点通过路由算法，确定该分片在哪个节点上：

   shard = hash(_routing) % (num_of_primary_shards)

   默认使用文档ID作为**routing值**，也可以通过API指定routing值

2. 分片节点收到请求后，会将请求写入到Memory Buffer，然后定时（默认是每隔1秒）写入到Filesystem Cache(磁盘缓存)

   这个从Momery Buffer到Filesystem Cache（磁盘缓存）的过程**就叫做Refresh**。在写入Buffer的同时，会同时写一个Transaction Log(每个分片会有一个Log文件)。当做refresh时，Buffer会被清空，Transaction Log不会清空。

3. ES每30分钟，会有一个Flush操作。

   该操作会调用fsync，将Filesystem Cache中的数据写入segment文件，旧的translog将被删除并开始一个新的translog

4. ES会自动进行Merge操作

   该操作会将多个Segment文件合并，在.del文件中被标记为删除的文档将不会被写入Segment，并且清空.del文件中的内容
   
   

## ES的索引压缩

### ES对索引的压缩

ES的索引是存储在内存当中的，也因此查询的速度非常的快，但是内存的大小是有限的，也因此ES会对他的索引进行压缩。
一般查询是**从Index中查找到Dictionary中对应的block，再去磁盘中去查找**，而不是直接去磁盘当中去随机的查询，这样有个好处就是：减少磁盘的随机IO次数，增加效率。
言归正传，ES采用FST的形式去把索引进行压缩的。
**FST以字节的形式去存储所有的Term Index**。用字节存储的话，肯定是占用空间很小的。因此能够有效的缩减存储所需的空间。
另外FST还有另外一个作用：快速确定某一个Term是否存在系统当中。



#### 补充知识：字典数据结构-FST(Finite State Transducers)

Lucene从4开始大量使用的数据结构是FST（Finite State Transducer）。FST有两个优点：

+ 空间占用小。通过对词典中单词前缀和后缀的**重复利用**，压缩了存储空间；(下面的案例就展现了重复利用的过程)
+ 查询速度快。$O(len(str))$的查询时间复杂度。

接下来演示FST插入过程

+ 插入“cat”

  插入cat，每个字母形成一条边，其中t边指向终点。

  ![img](https://pic2.zhimg.com/80/v2-faaaed78b9352325afaa9c4db03bc67d_720w.webp)

+ 插入“deep”

  与前一个单词“cat”进行最大前缀匹配，发现没有匹配则直接插入，P边指向终点。

  ![img](https://pic1.zhimg.com/80/v2-257571d300c8b82c5eba3efc5433d7ec_720w.webp)

+ 插入“do”

  与前一个单词“deep”进行最大前缀匹配，发现是d，则在d边后增加新边o，o边指向终点。

  ![img](https://pic3.zhimg.com/80/v2-06d65e079e2bf5a8fba86561d0c74fda_720w.webp)

+ 插入“dog”

  与前一个单词“do”进行最大前缀匹配，发现是do，**则在o边后增加新边g，g边指向终点**。

  ![img](https://pic1.zhimg.com/80/v2-a791b7a7785196b03571e2298823eb18_720w.webp)

+ 插入“dogs”

  与前一个单词“dog”进行最大前缀匹配，发现是dog，**则在g后增加新边s，s边指向终点**。

  ![img](https://pic2.zhimg.com/80/v2-b16af4e19803bd19b8fdfac9612e9e1d_720w.webp)

  

最终我们得到了如上一个**有向无环图**。利用该结构可以很方便的进行查询，如给定一个term “dog”，我们可以通过上述结构很方便的查询存不存在，甚至我们在构建过程中可以将单词与某一数字、单词进行关联，从而实现key-value的映射。



### ES对Posting List的压缩

举个例子：我们人，对于性别，只有男和女，如果一个Type下，有多条数据（Document），加入有一百万条，然后又以性别作为索引，那么想一想，这个Posting List是不是也有最短的是不是也有五十万条。这个数据量有点大。
因此，ES使用增量压缩编码，也就是**大数变小数，然后按照字节存储。**这种存储方式叫做Roaring BitMaps（RBM压缩）



ES索引小总结以及使用时注意的地方
**ES的索引思路总的来说为2点：**

+ 尽量把数据写到内存当中，减少磁盘的IO开销，增加查询的效率
  利用多种压缩算法，压缩数据。FST压缩索引啊，RBM压缩Posting List啊等等
  ES的索引在使用的时候应该注意哪些地方：

+ 因为索引是默认建立的，因此不需要使用索引的地方，一定要在定义的时候就声明出来。
  对于String类型的字段，在ES中分为text类型和keyword类型，不需要analysis的地方也要明确的声明出来。
  选择有规律的Id进行创建，否则默认创建的是随机的，没有规律的ID，这样今后进行数据压缩的时候，不方便，压缩的性价比低。



## Segment

### 定义

(参考置顶的ES架构图)

既然逆向索引是不可更改的，那么如何添加新的数据，删除数据以及更新数据？为了解决这个问题，**lucene将一个大的逆向索引拆分成了多个小的段segment。每个segment本质上就是一个逆向索引。**在lucene中，同时还会维护一个文件**commit point**，用来记录当前所有**可用的segment**，当我们在这个commit point上进行搜索时，就相当于在它下面的segment中进行搜索，每个segment返回自己的搜索结果，然后进行**汇总**返回给用户。

引入了segment和commit point的概念之后，数据的更新流程如下图：

<img src="https://cdn.jsdelivr.net/gh/WangMinan/Pics/20151118143354303" style="zoom:67%;" />

+ 新增的文档首先会被存放在内存的缓存中
+ 当文档数足够多或者到达一定时间点时，就会对缓存进行commit
  + 生成一个新的segment，并写入磁盘
  + 生成一个新的commit point，记录当前所有可用的segment
  + 等待所有数据都已写入磁盘

+ 打开新增的segment，这样我们就可以对新增的文档进行搜索了

+ 清空缓存，准备接收新的文档



### 文档的更新与删除

segment是不能更改的，那么如何删除或者更新文档？

每个commit point都会**维护一个.del文件**，文件内记录了在某个segment内某个文档已经被删除。在segment中，被删除的文档依旧是能够被搜索到的，不过在返回搜索结果前，会根据.del把那些已经删除的文档从搜索结果中过滤掉。

对于文档的更新，采用和删除文档类似的实现方式。当一个文档发生更新时，首先会在.del中声明这个文档已经被删除，同时新的文档会被存放到一个新的segment中。这样在搜索时，虽然新的文档和老的文档都会被匹配到，但是.del会把老的文档过滤掉，返回的结果中只包含更新后的文档。

 

#### Refresh

ES的一个特性就是提供实时搜索，新增加的文档可以在很短的时间内就被搜索到。在创建一个commit point时，为了确保所有的数据都已经成功写入磁盘，避免因为断电等原因导致缓存中的数据丢失，在创建segment时需要一个fsync的操作来确保磁盘写入成功。但是如果每次新增一个文档都要执行一次fsync就会产生很大的性能影响。在文档被写入segment之后，segment首先被写入了文件系统的缓存中，这个过程仅使用很少的资源。**之后segment会从文件系统的缓存中逐渐flush到磁盘，这个过程时间消耗较大。但是实际上存放在文件缓存中的文件同样可以被打开读取。**ES利用这个特性，在segment被commit到磁盘之前，就打开对应的segment，这样存放在这个segment中的文档就可以立即被搜索到了。

![img](https://cdn.jsdelivr.net/gh/WangMinan/Pics/20151118143425219)



上图中灰色部分即存放在缓存中，还没有被commit到磁盘的segment。此时这个segment已经可以进行搜索。

 

在ES中，**将缓存中的文档写入segment，并打开segment使之可以被搜索的过程叫做refresh。**默认情况下，分片的refresh频率是每秒1次。这就解释了为什么es声称提供实时搜索功能，新增加的文档会在1s内就可以进行搜索了。

Refresh的频率通过`index.refresh_interval:100s`参数控制，一条新写入es的日志，**在进行refresh之前，是在es中不能立即搜索不到的。**

通过执行

```bash
curl -XPOST127.0.0.1:9200/_refresh
```

可以手动触发refresh行为。

 

#### flush与translog

前面讲到，refresh行为会立即把缓存中的文档写入segment中，但是此时新创建的segment是写在文件系统的缓存中的。如果出现断电等异常，那么这部分数据就丢失了。所以es会定期执行flush操作，**将缓存中的segment全部写入磁盘并确保写入成功，同时创建一个commit point，整个过程就是一个完整的commit过程。**

但是如果断电的时候，缓存中的segment还没有来得及被commit到磁盘，那么数据依旧会产生丢失。为了防止这个问题，es中又引入了translog文件。

+ 每当es接收一个文档时，在把文档放在buffer的同时，都会把文档记录在translog中。

![img](https://cdn.jsdelivr.net/gh/WangMinan/Pics/20151118143536050)

+ 执行refresh操作时，会将缓存中的文档写入segment中，但是此时segment是放在缓存中的，并没有落入磁盘，此时新创建的segment是可以进行搜索的。

![img](https://cdn.jsdelivr.net/gh/WangMinan/Pics/20151118143545506)

+ 按照如上的流程，新的segment继续被创建，同时这期间新增的文档会一直被写到translog中。

![img](https://cdn.jsdelivr.net/gh/WangMinan/Pics/20151118143558387)

+ 当**达到一定的时间间隔，或者translog足够大时**，就会执行**commit**行为，将所有缓存中的segment写入磁盘。确保写入成功后，translog就会被清空。

![img](https://cdn.jsdelivr.net/gh/WangMinan/Pics/20151118143608305)

执行commit并清空translog的行为，在es中可以通过_flush api进行手动触发。

如：

```bash
curl -XPOST127.0.0.1:9200/tcpflow-2015.06.17/_flush?v
```

通常这个flush行为不需要人工干预，交给es自动执行就好了。同时，在重启es或者关闭索引之间，建议先执行flush行为，确保所有数据都被写入磁盘，避免照成数据丢失。通过调用

```bash
sh service.sh start/restart
```

会自动完成flush操作。

 

### Segment的合并

前面讲到es会定期的将收到的文档写入新的segment中，这样经过一段时间之后，就会出现很多segment。但是每个segment都会占用独立的文件句柄/内存/消耗cpu资源，而且，在查询的时候，需要在每个segment上都执行一次查询，这样是很消耗性能的。

> 每个 segment 是一个包含正排（空间占比90\~95%）+ 倒排（空间占比5\~10%）的完整索引文件，每次搜索请求会将所有 segment 中的倒排索引部分加载到内存，进行查询和打分，然后将命中的文档号拿到正排中召回完整数据记录。如果不对segment做配置，就会导致查询性能下降。

为了解决这个问题，es会自动定期的将多个小segment合并为一个大的segment。前面讲到删除文档的时候，并没有真正从segment中将文档删除，而是维护了一个.del文件，但是当segment合并的过程中，**就会自动将.del中的文档丢掉，从而实现真正意义上的删除操作**。

![img](https://cdn.jsdelivr.net/gh/WangMinan/Pics/20210616114218104.jpg)

当新合并后的segment完全写入磁盘之后，es就会自动删除掉那些零碎的segment，之后的查询都在新合并的segment上执行。Segment的合并会消耗大量的IO和cpu资源，这会影响查询性能。

默认情况下，归并线程的限速配置 `indices.store.throttle.max_bytes_per_sec` 是 20MB。对于写入量较大，磁盘转速较高，甚至使用 SSD 盘的服务器来说，这个限速是明显过低的。对于 ELK Stack (Elasticsearch、Logstash和Kibana) 应用，建议可以适当调大到 100MB或者更高。设置方式如下：

```http
PUT /_cluster/settings
{
    "persistent" : {
        "indices.store.throttle.max_bytes_per_sec" : "100mb"
    }
}
```

或者不限制：

```http
PUT /_cluster/settings
{
    "transient" : {
        "indices.store.throttle.type" : "none" 
    }
}
```

在es中，可以使用optimize接口，来控制segment的合并。

如：

```http
POST /logstash-2014-10/_optimize?max_num_segments=1
```

这样，es就会将`logstash-2014-10`中的segment合并为1个。但是对于那些更新比较频繁的索引，不建议使用optimize去执行分片合并，交给后台的es自己处理就好了。



## 数据一致性的保证

Elasticsearch是分布式的，当文档创建、更新、删除时，新版本的文档必须复制到集群中其他节点，同时，Elasticsearch也是异步和并发的。这就意味着这些复制请求被并行发送，并且到达目的地时也许顺序是乱的(老版本可能在新版本之后到达)。Elasticsearch需要一种方法确保文档的旧版本不会覆盖新的版本：Elasticsearch利用_version (版本号)的方式来确保应用中相互冲突的变更不会导致数据丢失。需要修改数据时，需要指定想要修改文档的version号，如果该版本不是当前版本号，请求将会失败。

这是一种乐观锁机制

### 1. **文档版本号**
每个文档都有一个版本号（version number），这是一个整数值，用于跟踪文档的更新次数。每次对文档进行写操作（如更新或删除）时，版本号都会递增。

可以使用如下代码查看

```http
GET /<index>/_doc/<document_id>
```

![5fe65d0227601050c82f616a49b03b7b](https://cdn.jsdelivr.net/gh/WangMinan/Pics/5fe65d0227601050c82f616a49b03b7b.png)

### 2. **写操作流程**
当执行写操作时，客户端会携带文档的当前版本号发送请求到Elasticsearch。写操作流程如下：
- **读取当前版本号**: Elasticsearch读取文档的当前版本号。
- **比较版本号**: Elasticsearch将请求中的版本号与当前版本号进行比较。
  - 如果请求中的版本号与当前版本号匹配，则执行写操作，并将版本号递增。
  - 如果请求中的版本号与当前版本号不匹配，则拒绝写操作，返回版本冲突错误。

### 3. **版本冲突**
版本冲突（Version Conflict）发生在两个并发写操作试图同时更新同一文档时。这种情况通常会导致以下两种处理方式：
- **重试机制**: 客户端可以捕获版本冲突错误，重新读取文档的最新版本，并尝试再次写入。这种机制被称为“乐观并发控制”。
- **应用逻辑处理**: 客户端可以根据具体的应用逻辑来决定如何处理版本冲突，例如放弃更新或合并修改。

### 4. **强制版本控制**
Elasticsearch还支持强制版本控制（Force Merge），允许在特殊情况下覆盖文档版本号。这种方式通常用于修复版本号错误或进行批量数据更新，但不建议在常规操作中使用。

### 5. **外部版本控制**
Elasticsearch中有内部版本号和外部版本号之分。使用内部版本号是要求指定的version字段和当前的version号相同。但在使用外部版本号时要求当前version号小于指定的版本号。如果请求成功，外部版本号作为文档新的version号进行存储。

除了内置的版本号机制，Elasticsearch还支持外部版本控制（External Versioning）。这种机制允许客户端自己管理版本号，适用于需要与外部系统同步数据的场景。使用外部版本控制时，客户端需要确保版本号的唯一性和递增性。

外部版本号命令：

```http
PUT /website/blog/2?version=5&version_type=external
```

内部版本号命令：

```http
PUT /website/blog/1?version=1 
```



### 6. **示例代码**
下面是一个使用版本号进行并发控制的示例：

```http
PUT /my_index/_doc/1?version=1
{
  "field": "value"
}
```

如果文档的当前版本号与请求中的版本号匹配（即都为1），Elasticsearch会执行更新并将版本号递增为2。如果版本号不匹配，则会返回一个版本冲突错误。

通过版本号机制，Elasticsearch能够有效地处理并发写操作，确保数据的一致性和完整性。