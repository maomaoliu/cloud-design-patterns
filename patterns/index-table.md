# 索引表模式

通过查询经常引用的数据存储区中的字段创建索引。该模式可以通过允许应用程序更快速从数据存储中检索定位数据来提高查询性能。

## 背景和问题

许多数据存储使用主键来组织实体集合的数据。应用程序可以使用该键来定位和检索数据。下图是存储客户信息的数据存储的例子。主键是Customer ID。该图显示了由主键（Customer ID）组织的客户信息。

![](https://docs.microsoft.com/en-us/azure/architecture/patterns/_images/index-table-figure-1.png)

虽然主键对于根据该键的值查询获取数据很有价值，但如果需要根据某个其它字段检索数据，应用程序可能无法使用主键。在上面的例子中，只有通过引用某个其它属性（例如客户所在的town）的值来查询数据时，应用程序无法使用Customer ID主键来检索客户。要执行这样的查询，应用程序可能必须获取并检查每个客户记录，这可能是一个缓慢的过程。

许多关系数据库管理系统支持二级索引。二级索引是由一个或多个非主要（次要）键字段组织的单独的数据结构，指示存储每个索引值的数据的位置。二级索引中的项目通常按二级主键的值进行排序，以便能够快速查找数据。这些索引通常由数据库管理系统自动维护。

你可以创建尽可能多的二级索引，因为需要支持应用程序执行的不同查询。例如，在关系数据库中的Customers表中，Customer ID是主键，如果应用程序经常在其所在的town查找客户，在town字段中添加二级索引会有所帮助。

然而，虽然二级索引在关系数据库中很常见，但云应用程序提供的大多数NoSQL数据存储无法提供相同的功能。

## 解决方案

如果数据存储不支持二级索引，可以通过创建自己的索引表来手动模拟。索引表通过特定的键组织数据。通常使用三种策略来构造索引表，具体取决于所需的二级索引的数量以及应用程序执行的查询的性质。

第一个策略是复制每个索引表中的数据，但是通过不同的键进行组织（完全非规范化）。下图显示了通过Town和LastName组织相同客户信息的索引表。
![](https://docs.microsoft.com/en-us/azure/architecture/patterns/_images/index-table-figure-2.png)

该策略适于数据与使用每个键查询的次数相比是相对静态的场景。如果数据更具动态性，则维护每个索引表的处理开销变得太大，不适于使用。此外，如果数据量非常大，则存储重复数据所需的空间量很大。
第二种策略是创建由不同键组织的归一化索引表，并使用主键而不是复制原始数据，如下图所示。 原始数据称为事实表。
![](https://docs.microsoft.com/en-us/azure/architecture/patterns/_images/index-table-figure-3.png)

这种技术可节省空间并减少维护重复数据的开销。缺点是应用程序必须执行两次查找操作以使用二级键查找数据。必须在索引表中找到数据的主键，然后使用主键查找事实表中的数据。
第三种策略是创建通过重复频繁检索的字段的不同键组织的部分归一化索引表。参考事实表来访问访问频度较低的字段。下图显示了频繁访问的数据是如何复制到每个索引表的。

![](https://docs.microsoft.com/en-us/azure/architecture/patterns/_images/index-table-figure-4.png)

这一策略可以在前两种方法之间取得平衡。普通查询的数据可以通过使用单个查找快速检索，而空间和维护开销没有复制整个数据集大。

如果应用程序经常通过指定组合值来查询数据（例如，“查找居住在Redmond的所有客户，姓氏为Smith”），则可以将索引表中的项目的键实现为级联的Town属性和LastName属性。下图展示了基于复合键的索引表。键按Town排序，然后按Town值相同的LastName排序。
![](https://docs.microsoft.com/en-us/azure/architecture/patterns/_images/index-table-figure-5.png)

索引表可以加快对分片数据的查询操作，并且在碎片密钥散列时特别有用。下图中的例子分片键是Customer ID的哈希值。索引表可以通过非标记值（Town和LastName）组织数据，并提供散列分片键作为查找数据。如果需要检索落在一个范围内的数据，或者按照非散列键的顺序获取数据取，则可以避免应用程序重复计算散列键（昂贵的操作）。例如，可以通过在索引表中找到匹配的项目，将它们全部存储在连续的块中来快速解决诸如“查找居住在Redmond的所有客户”之类的查询。 然后，使用存储在索引表中的分片键，跟随客户数据的引用。
![](https://docs.microsoft.com/en-us/azure/architecture/patterns/_images/index-table-figure-6.png)

## 问题和注意事项

实现此模式时，请考虑以下几点：
* 维持二级索引的开销可能很大。必须分析和了解应用程序使用的查询。只有在经常使用时才能创建索引表。不要创建推测索引表来支持应用程序不执行或仅偶尔执行的查询。
* 在索引表中复制数据可能会增加存储成本巨大开销和维护多份数据副本的工作量。
* 用引用原始数据的规范化结构实现索引表要求应用程序执行两次查找操作找到数据。第一个操作搜索索引表以检索主键，第二个操作使用主键来获取数据。
* 如果系统在非常大的数据集中包含多个索引表，则很难在索引表与原始数据之间保持一致性。可能围绕最终一致性模型设计应用程序。例如，要插入，更新或删除数据，应用程序可以将消息发布到队列，并让单独的任务执行操作并维护异步引用此数据的索引表。有关实现最终一致性的更多信息，请参阅[数据一致性入门](https://msdn.microsoft.com/library/dn589800.aspx)。
> 微软Azure存储表支持对同一分区（又称实体组事务）中保存的数据所做更改的事务更新。如果可以将事实表和一个或多个索引表的数据存储在同一分区中，则可以使用此功能来确保一致性。
* 索引表本身可能分区或分片。

## 何时使用该模式

当应用程序经常需要使用除主（或分片）键之外的键来检索数据时，使用此模式可以提高查询性能。
此模式可能不适于以下场景：

* 数据不稳定。索引表可能很快过时，使其无效或使维护索引表的开销大于使用索引表所节省的成本。
* 索引表选择的二级索引字段无差别，只能具有一小组值（例如，性别）。
* 索引表的二级键选择的字段的数据值的平衡偏差较高。例如，如果90％的记录在字段中包含相同的值，则创建和维护索引表，根据此字段查找数据可能会比依次扫描数据开销更大。但是，如果查询频繁访问的目标值位于剩余的10％中，该索引可能是有用的。应该了解应用程序执行的查询以及执行的频率。

## 案例
Azure存储表为在云中运行的应用程序提供了高度可扩展的键/值数据存储。应用程序通过指定键来存储和检索数据值。数据值可以包含多个字段，但数据项的结构对表存储是不透明的，它只是将数据项作为字节数组处理。

Azure存储表还支持分片。分片键包括两个元素，分区键和行键。具有相同分区键的项目存储在相同的分区（分片）中，并且项目以分片中的行键顺序存储。优化表存储用于执行查询，以获取落在分区内的行键值的连续范围内的数据。如果构建的应用程序正在使用Azure表存储信息，应考虑使用此功能来构建数据。

例如，存储有关电影的信息的应用程序。应用程序经常根据类型（动作，纪录片，历史，喜剧，戏剧等）查询电影。可以通过使用类型作为分区键为每个类型创建一个具有分区的Azure表，并将电影名称指定为行键，如下图所示。

![](https://docs.microsoft.com/en-us/azure/architecture/patterns/_images/index-table-figure-7.png)

如果应用程序还需要通过主演查询电影，使用这种方法效果较差。在这种情况下，可以创建一个独立的Azure表作为索引表。分区键是演员，行键是电影名称。每个演员的数据将存储在不同的分区中。如果电影有超过一个电影明星，同一部电影将出现在多个分区中。
可以通过采用上面“解决方案”部分中描述的第一种方法，复制每个分区所保存的值中的电影数据。然而，每个电影很可能复制好几次（每个演员一次），所以可能会更有效地将数据反规范化以支持最常见的查询（例如其他演员的名称），并使应用程序通过包括在类型分区中找到完整信息所必需的分区键来检索任何剩余的详细内容。此方法为“解决方案”部分中的第三个策略。下图展示了这种方法。
![](https://docs.microsoft.com/en-us/azure/architecture/patterns/_images/index-table-figure-8.png)

## 相关模式和指南

以下模式和指南在实现此模式时也可能相关：
* [数据一致性入门](https://msdn.microsoft.com/library/dn589800.aspx)。索引表必须在数据更改时同步其索引。在云中，将执行更新索引的操作作为修改数据的相同事务不太可能也不合适。在这种情况下，采用最终一致更为合适。文章提供了有关最终一致性问题的相关内容。
可能作为修改数据的相同事务的部分
* [分片模式](sharding.md)。索引表模式经常与使用分片分割的数据结合使用。分片模式提供了有关如何将数据存储分成一组分片的相关内容。
* [物化视图模式](materialized-view.md)。相比索引数据来支持汇总数据的查询，可能创建数据的物化视图更合适。物化视图模式中介绍了如何通过在数据上生成预填充视图来支持高效的汇总查询。