---
layout: post
title: Project3-query execution
tags: [database]


---

Project3

```c++
bool SeqScanExecutor::Next(Tuple *tuple, RID *rid) {
  TableIterator iter_ = *table_iter_;
  while (iter_ != table_heap_->End()) {
    Tuple tup = *iter_;
    iter_++;
    bool eval = true;
    if (plan_->GetPredicate() != nullptr) {
      eval = plan_->GetPredicate()->Evaluate(&tup, GetOutputSchema()).GetAs<bool>();
    }
    if (eval) {
      *tuple = tup;
      return true;
    }
  }
  return false;
}
```

当我写出如上所示代码以后，程序就陷入了死循环。单看代码当然是没问题的……因为这里我们需要推进迭代器，所以复制的话每次都是Begin迭代器。只有引用才可以持续推进迭代器。

```c++
TableIterator &iter_ = *(table_iter_.get());
```



上一部分学完Access Methods以后，我们该学习Operator Execution了，先是Sorting & Aggregations。接下来两周我们的学习任务是

* Operator Algorithms
* Query Processing Models
* Runtime Architectures

Query Plan像是一棵树，数据从叶流向根，根的结果就是query的结果。

今天的目标是学完 External Merge Sort、Aggregations

### Sort

那么为什么需要排序？

首先Quries可能会要求tuples以特定的方式排序（ORDER BY）。但即便query没有要求这么做，我们依然可以依靠排序做其他事情：

* 支持去重（DISTINCT）
* 插入大批排序过的tuples进B+Tree更加快
* Aggregations（GROUP BY）

如果数据能够直接放入内存，那么毫无疑问我们直接用快速排序就完事了。可惜磁盘中的数据量太大了，无法直接用快速排序。用分治就完事了，先排序小块，再合并。循环往复。

拿2路外部排序来说，如果DBMS有一个Bpage大小的BufferPool。

Pass#0:每次读取Bpages进入内存，排序并写回

Pass#1，2，3，。。。使用3个buffer pages，2个输入，1个输出，不断merge

这个算法只要求3个buffer pages来进行排序B=3，但是如果B>3，我们也不能够有效利用多出来的bufferpages，因为工作的瓶颈在磁盘I/O。

针对上面的问题，Double Buffering Optimizatio：当前这趟正在处理处理的时候，预取下一趟的数据到第二个buffer。

使用B+树来排序，可以利用叶子节点。当然要分成聚集B+树、非聚集B+树。对于聚集B+树来说，总是好于外部排序因为没有计算开销并且所有的磁盘访问都是顺序的。对于非聚集B+树来说是个坏主意，基本上一次I/O一个record。

### Aggregations

接下来说说**聚集**。有两种方式实现聚集：排序和哈希。

排序其实相对于聚集来说又做了更多的工作，也带来了不必要的开销。所以有些情况下使用哈希更好。直接建立一个短暂的哈希表来扫描table。每个record，就查哈希表。

* Distinct：丢弃重复的
* Group by：执行聚合运算。

当然如何这些都可以放进内存来做的话，是很简单的……但是不能，所以另谋他路。所以来看外部哈希聚合

Phase#1-Partition：根据哈希函数h1把tuples划分到不同桶中，当桶满了把他们写到磁盘。

Phase#2-Rehash：建立一个或者多个哈希表（哈希函数h2），遍历每一个Partition（也就是第一步中的桶）并放入哈希表中，最后遍历哈希表（们）来取得符合要求的tuples。（假定每一个Partition都能放进内存）。

### Conclusion

选择sorting还是hashing要分情况，不能一概而论。

我们已经讨论过关于sorting的优化：分块来分摊开销、两个buffer来解决I/O瓶颈。

### NextClass

* Nested Loop Join

* Sort-Merge Join

* Hash Join

### Join operators

问题1：join操作需要给parent operator留下什么数据

问题2：如何判断两个join之间哪个更好

Early materialization：把outer和inner tuples拷贝进新的output tuple。缺点是可能最后的output tuple会很大很大。

Late materialization：把join keys连同record ids拷贝进来，如果后续还需要其他属性，通过record ids再获取。十分适合column stores，不会拷贝不需要的属性。

### cost analysis criteria

衡量cost的标准：IO次数。

假定 R有M pages，m tuples，S有N pages，n tuples。

### Nested Loop Join

* Simple/Stupid

  对R中的每一个tuple，遍历S中的每一个tuple，如果match，就emit。 

  Cost：M+m*N

* Block

  对R中每一个blockr，对S中的每一个blocks，对blockr中每一个tupler，对blocks中每一个tuples，如果match，就emit。

  Cost：M+M*N

  如果我们有B个buffer，可以如何利用呢？拿B-2个buffer来扫描outer table，1个扫描inner table，1个buffer输出。

  Cost：M+（M/（B-2） * N）

* Index

  为什么basic nested loop join看起来如此糟糕？对于outer table的每一个tuple，都要对inner table做一次sequential scan。因此我们可以使用索引来找inner table中符合的tuple。

  假设每次通过索引查找inner table的开销是C

  Cost：M+m*C

综上所述，Nested Loop Join要注意以下几点：

1.选择较小的table作为outer table（left table）

2.把outer table尽量多的放在内存buffer中

3.对inner table使用index

### Sort-Merge Join

Phase #1:Sort

Sort Cost（R）：2M *（1+logB-1（M/B）），复杂度注意是怎么回事，前面的2 * M是每一趟从磁盘读取，写入Mpages，后面是总共的趟数。

Sort Cost（S）：2N*（1+logB-1 （N/B））

Phase #2:Merge

Merge Cost：（M+N）

Total Cost： Sort + Merge

当然最差的情况就是outer table 和inner table里面所有join attribute值都相同。那么Cost就变成Nested loop join + sort cost=M*N+sortcost，这里推测使用的是Block为单位的Nested Loop Join。

那么什么情况下sort-merge有用呢？

当一个或者两个table都已经对join key排序。

当join key的输出必须是排序的。

### Hash Join

既然join属性相同，那么经过hash之后肯定也相同，因此可以使用hash join。

Phase #1:Build

扫描outer table并在join attributes使用h1，建立哈希表

Phase #2:Probe

扫描inner table并使用h1来跳到哈希表，检查是否匹配。

如何优化？在第一步哈希表find元素的时候使用bloom filter，bloom filter可以保证False negative不会发生，也就是说如果bloom filter告诉你这个元素不存在那就是不存在，如果告诉你存在那不一定存在。

如果没有足够的内存放下整个哈希表怎么办？我们当然也不想要buffer pool manager随机的将哈希表所在的page换出到磁盘。

这个Cost也是3（M+N），见Grace Hash Join

### Grace hash join

Recursive partitioning：桶中桶，使用不同的哈希函数。

Partitioning Phase：Read+Write Both tables 2（M+N） IO'S

Probing Phase：Read both tables M+N IO'S

当然如果DBMS知道outer table的大小的话，直接使用一个静态的哈希表就行，减少了build/probe的开销。如果不知道的话就只能动态哈希表或者允许page溢出。

### Join Algorithms：Summary

![image-20210504142431056](../image/image-20210504142431056.png)

### Conclusion

在聚合中使用sorting还是hashing要分情况。

下一节课我们要来着重讲讲组合operators来执行查询。

今天的课程内容

* Processing Models
* Access Methods
* Modification Queries
* Expression Evaluation

### Processing Models

Iterator model：就跟迭代器一样，每次输出一个tuple

materialization model：一次把全部结果输出

vectorized/batch model：像迭代器一样，只不过每次输出一个batch

### Access Methods

sequential Scan

optimizations

* zone maps：对于某个page预先计算出min，max，sum，DBMS然后就可以决定是否要访问该page
* Late materialization：延迟tuples的组合，在中间传输offset。

Index Scan

Multi-Index / “Bitmap” Scan

### Modification Queries

halloween problem：update改变了tuple的物理位置，导致scan operator访问该tuple多次（一般发生在update=delete+insert处），可能会在clustered tables 或者 索引扫描中出现。

### Expression Evaluation

就是说表达式树虽然很灵活，但是很满。



下一节课我们来讲讲 并行query execution，上一节课中我们都是假设只有一个worker在查询，那么现在来看看多个workers一起工作的情况。

parallel vs distributed

parallel：资源在物理上仅靠在一起，资源之间的通信很快，通信被认为是简单可靠的。

distributed正好相反。

今天的主要内容：process models、execution parallelism、I/O parallelism

### process models

#### process per DBMS worker

#### process pool

#### thread per dbms worker

使用多线程的好处：减少每次上下文切换的开销、不用管理共享内存。

10年内除了Redis和Postgres forks，Andy没发觉有其他DBMS不使用多线程。说明DBMS一般都使用多线程查询。

### execution parallelism

inter- vs intra-query parallelism

inter是不同queries之间执行。如果queries是只读的，那么不同queries之间只需要一点协同。如果要同时更新数据库，那么会有点困难（之后的课程会讲这部分）。

intra是一个queries被并行执行。

#### intra-operator（horizontal）

将operator分解成互相独立的fragments，对于数据的不同子集执行相同的函数。

引入了一个exchange operaotr：gather、distribute、repartition

#### inter-operator（Vertical）

operations之间有重叠的，目的是为了从一个stage pipeline到另一个stage，比如原先是先全部join完再project，现在可以让join完的一部分pipeline到project部分。

#### bushy

就是inter-operator的扩展版本，inter-operator有点像组内两个operator重叠，bushy就是组间重叠。

### I/O parallelism

#### multiple disks per database

#### one database per disk

#### one relation per disk

#### split relation across multiple disks

这节课到此结束，下节课我们来讲讲query optimization

Heuristics/Rules

重写query来移除低效的部分。

Cost-based Search

使用模型来计算query plan的开销，选择开销最小的plan。

### 今天的学习任务：启发式/规则：关系代数等式、逻辑查询优化、嵌套查询、表达式重写。

### relational algebra equivalences

Selections：尽早筛选，将复杂的谓词逻辑简单化执行。

Joins：交换律、结合律。

Projections：尽早执行投影，只剩下需要的属性，其他全都去掉。

### logical query optimization

转换成等价的逻辑查询，以便选出最优的计划。

split conjunctive predicates：就是分离连接谓词，一个一个执行，减少工作量。

predicate pushdown：将谓词移到尽可能的下方，同时也是对应的笛卡尔积的上方。

replace cartesian products with joins：可以把笛卡尔积和谓词替换成inner joins。

project pushdown：下沉project，去掉不必要的属性。

### nested sub-queries

rewrite to de-correlate and/or flatten them

decompose nested query and store result to temporary table

### Expression rewriting

就是说了一些没必要的谓词的删除或者化简。

## 今日学习任务：上一节课我们讲了启发式/规则来优化查询，这节课我们来讲基于开销的搜索

### Cost Estimation

Choice #1:Physics Costs

Choice #2:Logical Costs

Choice #3:Algorithmic Costs

对于基于磁盘的DBMS来说访问磁盘的次数的衡量一个query执行时间的主导因素。

### Plan Enumeration

在执行重写以后，DBMS会为这个quert列举不同的plans并且评价他们的开销。选择最好的plan。

single relation

multiple relations：只考虑left deep join。

在这种情况下其实还可以列举出很多。比如Left-deep tree就不止一种。aggregation、join又不止一种，aggregation可以是sort、merge，join可以是nested loop join、sort-merge join、grace sort-merge join。每个数据库的访问方式又不止一种，可以是index、seq scan。

nested sub-queries？没讲东西啊，略过算了。

dynamic programming：比较不同plan之间的cost，留下cost最小的那个plan

所以得到候选plan的步骤如下

1.枚举relation orderings

2.枚举join

3.枚举access

下一节课我们要讲事务了！数据库系统第二个最困难的部分。

