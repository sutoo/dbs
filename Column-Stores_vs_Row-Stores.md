基于这篇论文：
Column-Stores vs. Row-Stores: How Different Are They Really?


warehouse workloads

# optimizations
direct operation on compressed data

# Star Schema Benchmark
数据仓库中的星型模型：
一个 fact table + 多个 dimension tables

# 实现方案
## Vertical Partitioning
最直接的方式，完全的垂直分割。
为每一个column创建一个物理表，这个表里每个记录，除了value之外，还有一个“position”，来表示column之间的关联。（即相关position的，为同一行）
```
 Queries
are then rewritten to perform joins on the position attribute when
fetching multiple columns from the same relation
```
此方案的劣势：
1 position attribute 浪费存储空间
2 进一步，多数的列式存储，更偏向于有更多的列，这样会导致1的坏处被放大。

## Index-only plans
实现：数据还是以行的方式存储，然后为每一列构建一个B+tree的索引。
查询谓词的问题。如果某一列没有对应的查询谓词，就需要扫描全列：举例：
`SELECT AVG(salary) FROM emp WHERE age>40`
column 'age'是有谓词判断的，可以直接过滤出一部分 record ids，然后再与 column 'salary'的全量进行merge。
注：这里为什么扫描'salary'列，而不是去行存储中去找。因为只读取salary列更节省，虽然去行存储中昂去找，
可以用到基于record id的索引，但是需要权衡：读出的n行包含全部column的数据 vs 全量扫描'salary'一个column数据
所以应当这个谓词带来的问题：create indices with composite keys

## Materialized Views
create an optimal set of materialized views for every query flight in the workload, where the optimal view for a given flight has only the columns needed to answer
queries in that flight.
备注：
```
Materialized Views
在计算中，物化视图是包含查询结果的数据库对象。例如，它可能是远程数据的本地副本，也可能是表或联接结果的行和/或列的子集，或者可能是使用聚合函数的摘要。
建立实例化视图的过程有时称为实例化。这是一种缓存查询结果的形式，类似于功能语言中对函数值的记忆，有时将其描述为一种预计算形式。 维基百科（英文)
```

# 列式存储中的优化 optimizations
## 压缩
```
Compression algorithms
perform better on data with low information entropy(熵)
```
列数据中间更适合压缩。举例：电话号码之间有相似性，而电话号码与邮箱之间，就很难找到相似的地方。（有序）
压缩主要不是为了减少磁盘存储，而是为了减少磁盘IO时间，但会带来解压缩的CPU损耗（也即是时间损耗）。
所以如果可以直接对压缩数据进行操作（direct operation on compressed data）会更好。

## 延迟物化（Late Materialization ）
（从知乎摘抄）
往往一个SQL中filter和project只会涉及到整行数据的部分列，但在传统行式存储下这些计算过程需要先将整行数据（因为整行数据是放在一起的）解析和提炼出来（虽然可以只选择与SQL有相关性的列参与后续计算，但整行解析的过程是少不了的），然后再做一些丢弃；而在列存中，每个列存储上都是独立的，因而真的可以做到只解析与SQL有相关性的列并过滤，不同列之间以position（偏移量）作为行关联的依据。针对不同列的不同filter，最终产生多个position列表（列表可以是bitmap，也可以是range，类似于run-length-encoding）的中间结果集，然后多个结果集合并后，最后组装行结果时再查找真正需要project的列。

有4个方面的优点：

* 很多聚合与选择计算，压根不需要整行数据，过早物化会浪费严重；
* 很多列是压缩过的，过早物化会导致提前解压缩，但很多操作可以直接下推到压缩数据上的；
* 面向真正需要的列做计算，CPU的cache效率很高（100%），而行存因为非必要列占用了cache line中的空间，cache效率显然不高；
* 针对定长的列做块迭代处理，可以当成一个数组来操作，可以利用CPU的很多优势（SIMD加速、cache line适配、CPU pipeline等）；相反，行存中列类型往往不一样，长度也不一样，还有大量不定长字段，难以加速；

## 块迭代 （Block Iteration）
in all column-stores (that we are aware of), blocks of values from the same column are sent to
an operator in a single function call.
Operating on data as an array not only minimizes per-tuple overhead, but it also exploits potential
for parallelism on modern CPUs, as loop-pipelining techniques can be used 

## 隐式连接 (Invisible Join)
这里的“隐式”是指，没有通过传统的join方式（两两表迭代，生成两个表联合在一起的宽行数据，再做过滤）来实现join，
而是通过维持不同列的相同行之间的position对应关系来完成多个表join。与倒排索引很类似。
原文的解释参考：
https://cloud.tencent.com/developer/article/1694495
