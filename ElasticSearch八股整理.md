<div align="center">
    <h1>
        ElasticSearch八股整理
    </h1>
</div>

## 倒排索引

- 正排索引：文档ID到文档内容和单词的关联
- 倒排索引：单词到文档ID的关系
  备注：ES对文档每个字段都有自己的倒排索引，可以指定某些字段不做索引，这样可以节省存储空间，缺点是这个字段无法被搜索。

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