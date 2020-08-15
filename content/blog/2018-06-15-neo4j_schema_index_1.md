---
title: Neo4j schema 索引对查询的影响
tags: Neo4j
category: Technology
excerpt: 经过实践，在我们给查询时的关键字段加上索引后，解决了读取速度过慢的问题
created: 2018-06-15
image: ./images/marco-marques-dJ_Zl5LpPto-unsplash.jpg
image_caption: Photo by Marco Marques on Unsplash
author: insutanto
---

Neo4j是一个高性能的NOSQL图形数据库，可以很方便的对图数据进行建模和计算。最近在使用Neo4进行数据分析时，发现查询的性能特别差，于是开始对Cypher查询语句和已有的 Neo4j 数据库进行性能优化。具体介绍可以自行百度。

``` cypher
profile match p = (exporter1:COMPANY {companyName: "APS AIRPARTS SERVICES& SUPPLIES"}) -[rel1:BOLCNT]-> (importer) <-[rel2:BOLCNT]- (exporter2) -[rel3:BOLCNT]-> (importer) <-[rel4:BOLCNT]- (exporter1) 
    where (id(importer) <> id(exporter1) <> id(exporter2) <> id(importer)) 
    and (rel1.hscode = rel2.hscode) 
    and (rel3.hscode = rel4.hscode) 
    return exporter1.companyName, exporter2.companyName, Company.sim(exporter1.companyName, exporter2.companyName)
```

我们通过 Cypher 中的 profile 关键字，对查询进行性能分析。发现查找节点 exporter1 时需要遍历大量节点，于是给 COMPANY 节点的 companyName 字段增加索引，预计可以减少查找的时间。[Neo4j 索引的相关文档](https://neo4j.com/docs/developer-manual/3.4/cypher/schema/index/)

执行Cypher语句: 

`create index on :COMPANY(companyName)`

执行结束后，Neo4j 会在后台增加索引，索引增加完成后会自动使用索引。我们隔一段时间之后，可以通过执行以下 Cypher 语句查看索引是否上线。

`:schema`

执行后如果返回类似下面带有 Indexes  的数据，并且对应索引最后是 ONLINE 状态，则表示索引建立完成，并且已经可以使用。

```
Indexes
   ON :BOLCNT(hscode) ONLINE 
   ON :COMPANY(companyName) ONLINE 

No constraints
```

PS: 这里最后一句"No constraints"，说明了我的数据库中没有约束存在。通常我们如果需要对数据进行唯一性、存在性等约束时，就需要在数据库中，针对相应的属性字段创建约束。[Neo4j 约束的相关文档](https://neo4j.com/docs/developer-manual/3.4/cypher/schema/constraints/)

![优化后的查询性能分析](http://ww1.sinaimg.cn/large/87c01ec7gy1fsrctww14gj205m0d474k.jpg)

经过实践，在我们给查询时的关键字段加上索引后，解决了读取速度过慢的问题；我们比较相同查询语句的性能，发现部分查询语句的节点遍历数减少了2000倍，查询速度提升了45倍。我们可以下一个结论，当我们查询无索引的节点属性时，Neo4j 会遍历数据库来查找数据，而当我们添加索引后，Neo4j 可以通过索引直接找到相应的节点。

通过查看文档，我们还发现，Neo4j 还支持在 Cypher 查询中显式(强制)使用索引。实际测试，在我们的查询下，使用两种方式在数据库中需要遍历的节点数是一样的，这种情况下，可以理解为，查询时显式声明和不声明没有差异。

``` cypher
profile match p = (exporter1:COMPANY {companyName: "APS AIRPARTS SERVICES& SUPPLIES"}) -[rel1:BOLCNT]-> (importer) <-[rel2:BOLCNT]- (exporter2) -[rel3:BOLCNT]-> (importer) <-[rel4:BOLCNT]- (exporter1) 
using index exporter1:COMPANY(companyName)
    where (id(importer) <> id(exporter1) <> id(exporter2) <> id(importer)) 
    and (rel1.hscode = rel2.hscode) 
    and (rel3.hscode = rel4.hscode) 
    return exporter1.companyName, exporter2.companyName, Company.sim(exporter1.companyName, exporter2.companyName)
```

通过文档我们可以了解，这是因为，我们的查询语句并不属于显示使用索引所适用的场景：

> When executing a query, Neo4j needs to decide where in the query graph to start matching. This is done by looking at the MATCH clause and the WHERE conditions and using that information to find useful indexes, or other starting points.</br>
> However, the selected index might not always be the best choice. Sometimes multiple indexes are possible candidates, and the query planner picks the wrong one from a performance point of view. Moreover, in some circumstances (albeit rarely) it is better not to use an index at all.</br>
> Neo4j can be forced to use a specific starting point through the USING keyword. This is called giving a planner hint. There are four types of planner hints: index hints, scan hints, join hints, and the PERIODIC COMMIT query hint. 

> 当我们执行一个查询时，Neo4j 需要决定从查询图中的哪个位置开始匹配。这是通过查看 MATCH 子句和 WHERE 条件去找到有用的索引，或者其他开始的点。</br>
> 然而，Neo4j 选择的索引可能不是总是最佳的选择。从性能的角度考虑，在有些时候，会存在多个索引成为可能的候选，而查询计划选择了错误的一个。</br>
> 此外，在一些很少的情况下，最好不要使用索引。</br>
> Neo4j 可以通过使用关键词 using 来强制使用一个特殊的起点。这叫做提供一个计划提示。一共有四种计划提示：索引提示，扫描提示，连接提示，以及定期提交的查询提示。

所以，如果查询语句中涉及多个索引，显示的使用索引会更有优势。

最后，Neo4j 其实还有自动索引等技术，接下来在使用过程中，如果用到，单独再进行探讨。