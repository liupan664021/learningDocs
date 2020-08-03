## Apache Calcite: A Foundational Framework for Optimized Query Processing Over Heterogeneous Data Sources

作者：Julian Hyde（Calcite创始人）

#### Calcite的优点

- **Open source friendliness**：开源
- **Multiple data models**：多数据模型
- **Flexible query optimizer**：灵活的查询优化器
- **Cross-system support**：支持跨系统——不同的查询处理系统以及数据源
- **Reliability**：可靠性
- **Support for SQL and its extensions**：支持SQL及其扩展

### 类似的框架

- Orca
- Spark SQL
- Algebricks
- Carlic
- FORWARD
- BigDAWG
- Myria

### 架构

![image-20200530223312284](C:\Users\liupan\AppData\Roaming\Typora\typora-user-images\image-20200530223312284.png)

- 有一些系统支持SQL，但是很少有查询优化，比如Hibe和Spark都支持SQL，但是没有包含优化器。这种情况下， 在查询被优化后，Calcite会将关系表达式转化回SQL，再进行查询。这样的话，对于那些提供了SQL接口，但是没有优化器的数据管理系统，Calcite就是一个在其之上的独立存在的系统（工具）。
- Calcite架构不只是针对SQL进行优化的项目。更加普遍的情况是，数据处理系统选择他们自己的查询语言的解析器，然后Calcite也可以对这些查询进行优化。甚至，Calcite也允许直接实例化关系算子来构建算子树，可以使用内置的关系表达式构建器（*relational expressions builder*）来完成这种操作。

### 查询代数

#### Operators

Calcite包含最常见的数据操作表达式，比如：filter、join、project等等，也包含了更加复杂的表达式，用于更加高效地识别优化机会。比如，流计算里面的window。

#### Traits

Calcite不包含不同的条目来表征逻辑和物理的算子。相反，使用特质（traits）来描述一个算子的物理属性。这些特质可以用来评估不同的备选计划（plan）的代价（cost）。改变特质并不会改变被评估的逻辑表达式，也就是说给定的算子始终会生成相同的行（rows）。

在优化的过程中，Calcite总是尝试将特定的traits强加到关系表达式上，比如：特定列的排列顺序。关系算子通过实现一个*converter*接口，将一个表达式的特质从一个值改变为另一个值。

Calcite包含了常规的traits来描述关系表达式生成的数据的物理属性，比如：ordering、grouping、partitioning等。Calcite可以推导出这些属性并且利用他们避免不必要的表达式出现在计划当中。例如，如果一个排序算子的输入已经被正确排序了，那么这个排序表达式就会被去掉。

除了这些属性以外，Calcite还有一个主要的特征叫：calling convention 特质。本质上讲，calling convention 特质代表的是这个关系表达式应该在哪个数据处理系统（数据源）被执行。这样，Calcite就可以将那些在不同的处理系统上执行代价差异较大的表达式进行优化，选择更加合适的处理系统进行执行。

>  For example, consider joining a Products table held in MySQL to an Orders table held in Splunk (see Figure 2). Initially, the scan of Orders takes place in the splunk convention and the scan of Products is in the jdbc-mysql convention. The tables have to be scanned inside their respective engines. The join is in the logical convention, meaning that no implementation has been chosen. Moreover, the SQL query in Figure 2 contains a filter (where clause) which is pushed into splunk by an adapter-specific rule (see Section 5). One possible implementation is to use Apache Spark as an external engine: the join is converted to spark convention, and its inputs are converters from jdbc-mysql and splunk to spark convention. But there is a more efficient implementation: exploiting the fact that Splunk can perform lookups into MySQL via ODBC, a planner rule pushes the join through the splunk-to-spark converter, and the join is now in splunk convention, running inside the Splunk engine.

例子中，对来自不同数据源的数据进行扫描之后会有个join，join之后会进行filter，一种处理方法是scan之后的表达式操作在一个外部的spark引擎中进行，这样在scan之后会使用spark的convention，但是由于filter的对象只来自splunk，所以可以把join下推到filter之后，在splunk之中完成filter之后再join，这样的话，开销明显变小，cost降低，达到优化的目的。

![image-20200530225829208](C:\Users\liupan\AppData\Roaming\Typora\typora-user-images\image-20200530225829208.png)

### 适配器

适配器就是为了适配不同的数据源而诞生的，结构：

![image-20200530225943935](C:\Users\liupan\AppData\Roaming\Typora\typora-user-images\image-20200530225943935.png)

模式工厂从model文件里面读取参数配置，构建模式实例，然后模式实例里面包含了函数、子模式、表达式以及表。

最基本的适配器需要实现表的扫描（scan），也可以实现更多的功能。

为了拓展适配器提供的功能，Calcite定义了一种名叫*enumerable*的convention，简单地通过迭代接口对元组里面的数据进行操作，这样，Calcite就可以实现那些在某些数据源中不支持的操作。

如果某些查询只涉及表当中很少的一部分数据，那么遍历所有的元组就很低效了。基于规则的优化器可以实现adapter-specific rules（基于适配器的规则）。例如，要过滤和排序一张表的数据，那么可以在数据源上进行过滤的适配器将实现一条规则：匹配LogicalFilter，并将它转换为这个适配器的convention上面，这样的话，这个规则就将LogicalFilter转换为了另一个Filter的实例。这个新的Filter实例代价更小，Calcite就可以实现跨适配器的查询优化了。

> For example, suppose a query involves filtering and sorting on a table. An adapter which can perform filtering on the backend can implement a rule which matches a LogicalFilter and converts it to the adapter’s calling convention. This rule converts the LogicalFilter into another Filter instance. This new Filter node has a lower associated cost that allows Calcite to optimize queries across adapters.

Calcite可以利用上面的规则实现多数据源的查询优化：将尽可能多的逻辑下推到数据源，这样就能执行joins和aggregations更加高效。

![image-20200530231402434](C:\Users\liupan\AppData\Roaming\Typora\typora-user-images\image-20200530231402434.png)

### 查询处理和优化

#### Planner rules



#### Metadata providers

两个目的：

1. 指导planner实现降低整体查询计划的cost的目标；
2. 提供信息给正在被执行的规则

Janino实现的，轻量化的Java编译器，由于缓存元数据结果，显著提高查询效率。比如，当需要计算诸如一个join的基数性、平均行大小和选择性等元数据时，是很方便的。

#### Planner engines

CBO和RBO

CBO基于代价和数据对planner进行动态的优化，直到得到一个比较低的cost的计划。CBO比较复杂，比较慢，但是更加有效。

RBO更快，但是可能出现死循环，来回转换。而且会出现局部最优解。

#### Materialized views

- view substitution
- lattices



![image-20200530232640356](C:\Users\liupan\AppData\Roaming\Typora\typora-user-images\image-20200530232640356.png)





![image-20200530232654711](C:\Users\liupan\AppData\Roaming\Typora\typora-user-images\image-20200530232654711.png)