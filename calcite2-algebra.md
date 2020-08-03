## Calcite学习笔记2-Algebra

### 概述

关系代数（relational algebra）是Calcite的核心。每一条query代表了一颗关系算子的树。你可以将SQL转换为关系代数，也可以直接构建关系算子树。

Planner rules利用保留语义的数学标识符来转换表达式树。例如：将filter下推导没有引用其他输入的列的inner join的输入是有效的（valid）。

> Calcite optimizes queries by repeatedly applying planner rules to a relational expression. A cost model guides the process, and the planner engine generates an alternative expression that has the same semantics as the original but a lower cost.

Calcite通过重复地将planner rules应用到关系表达式上来优化query（查询）。一个代价模型来指导这个过程，planner引擎生成具有相同语义但是代价更小的替代表达式，从而优化查询。

当然，planning过程是可以扩展的，你可以添加自己的关系算子、计划规则、代价模型以及数据统计（ relational operators, planner rules, cost model, and statistics）

### Algebra builder

构建关系表达式的最简单的方式是使用代数生成器（algebra builder）：`RelBuilder`。下面是一个例子

#### TableSan

```java
final FrameworkConfig config;
final RelBuilder builder = RelBuilder.create(config);
final RelNode node = builder
  .scan("EMP")
  .build();
System.out.println(RelOptUtil.toString(node));
```

可以在`RelBuilderExample.java`中找到完整的代码：

打印：

```shell
LogicalTableScan(table=[[scott, EMP]])
```

创建了一个对`EMP`表的扫描，等价于下面的SQL：

```sql
SELECT *
FROM scott.EMP;
```

#### 添加一个Project（投影）

SQL变为：

```sql
SELECT ename, deptno
FROM scott.EMP;
```

仅仅在调用`build`之前添加一个`project`方法就可以了：

```java
final RelNode node = builder
  .scan("EMP")
  .project(builder.field("DEPTNO"), builder.field("ENAME"))
  .build();
System.out.println(RelOptUtil.toString(node));
```

输出为：

```shell
LogicalProject(DEPTNO=[$7], ENAME=[$1])
  LogicalTableScan(table=[[scott, EMP]])
```

#### 添加一个Filter（过滤）和 Aggregate（聚合）

代码：

```java 
final RelNode node = builder
  .scan("EMP")
  .aggregate(builder.groupKey("DEPTNO"),
      builder.count(false, "C"),
      builder.sum(false, "S", builder.field("SAL")))
  .filter(
      builder.call(SqlStdOperatorTable.GREATER_THAN,
          builder.field("C"),
          builder.literal(10)))
  .build();
System.out.println(RelOptUtil.toString(node));
```

等价于下面的SQL：

```sql
SELECT deptno, count(*) AS c, sum(sal) AS s
FROM emp
GROUP BY deptno
HAVING count(*) > 10
```

输出：

```shell
LogicalFilter(condition=[>($1, 10)])
  LogicalAggregate(group=[{7}], C=[COUNT()], S=[SUM($5)])
    LogicalTableScan(table=[[scott, EMP]])
```

#### Push and Pop

builder采用堆栈的方式存储关系表达式，上一步的生成的关系表达式作为输入传递给下一步。这允许能够生成表达式的方法生成builder。

很多时候，你可能只会使用唯一的stack方法：`build()`，来获取最后的关系表达式，也就是树的根（root)

有时候堆栈嵌套地太深，容易产生误解。为了让事情保持简单，您可以从堆栈中删除表达式。例如，这里我们正在构建一个复杂的join:

```shell
.
               join
             /      \
        join          join
      /      \      /      \
CUSTOMERS ORDERS LINE_ITEMS PRODUCTS
```

我们可以通过三步来构建，存储中间结果通过`left`和`right`，在最后的`Join`时使用`use()`将它们放进堆栈。

```java
final RelNode left = builder
  .scan("CUSTOMERS")
  .scan("ORDERS")
  .join(JoinRelType.INNER, "ORDER_ID")
  .build();

final RelNode right = builder
  .scan("LINE_ITEMS")
  .scan("PRODUCTS")
  .join(JoinRelType.INNER, "PRODUCT_ID")
  .build();

final RelNode result = builder
  .push(left)
  .push(right)
  .join(JoinRelType.INNER, "ORDER_ID")
  .build();
```

#### 字段名和序号

你可以使用名字和序号来引用一个字段

序号从0开始。每个算子保证了字段在输出中的顺序。例如，`Project`返回由每一个标量表达式生成的字段。

每个算子中的每个字段名会被保证是独一无二的，但是有时候事情不是你想的那样。比如，当你join `EMP`和`DEPT`两张表时，你会发现输出字段有一个叫DEPTNO，另外一个可能是DEPTNO_1这种形式。

有些关系表达式的方法允许你在字段名上做更多的控制：

- `project`允许你用`alias(expr, fieldName)`来装饰表达式，去除了装饰器但是保持了建议的名字（只要是独一无二的）
- `values(String[] fieldName, Object... values)`接收一个字段名的数组。如果数组的某一个元素是nul的，那么这个builder会生成一个独一无二的名字。

如果一个表达式投影了一个输入字段，或者是输入字段的转换（cast），那么它会使用这个输入字段的名字。

一旦这个独一无二的字段名被分配了，那么它就不可以再被修改。如果你有一个特定的`RelNode`实例，你可以不用担心字段名会被更改。事实上，整个关系表达式都是不可变的（*the whole relational expression is immutable*）。

但是如果一个关系表达式通过了多个重写规则，那么结果表达式的字段名可能和原来的看起来差别很大，所以这种时候最好使用字段序号来引用字段。

当你在使用多个输入来构建关系表达式时，你最好考虑构建字段的引用。这在构建join条件时经常发生。

假设你在构建一个EMP表（包含8个字段：[EMPNO, ENAME, JOB, MGR, HIREDATE, SAL, COMM, DEPTNO] ）和DEPT表（包含3个字段：[DEPTNO, DNAME, LOC]）的join，那么Calcite会在内部通过一个长度为11 的序号来引用这个字段：从#0开始的前8个序号代表EMP表的8个字段，从#8开始的三个序号代表DEPT表的3个字段。

但是通过这个builder的API，你可以指定哪一个输入的哪一个字段。比如，为了引用“SAL“字段（内部的序号为#5），你可以写`builder.field(2, 0, "SAL")`、`builder.field(2, "EMP", "SAL")`或者`builder.field(2, 0, 5)`。这表明”SAL“字段是在#0输入的#5字段。（为什么需要知道有2（builder方法的第一个参数）个输入呢，因为他们是存在栈里面的，input #1是栈顶，input #0在它的下面，如果我们不告诉有2个输入，就无法找到input #0）

相同，可以引用"DNAME"（内部序号为#9（8+1）），使用`builder.field(2, 0, "DNAME")`、`builder.field(2, "DEPT", "DNAME")`或者`builder.field(2, 1, 1)`

#### 递归查询

警告：目前递归查询的API还处于试验阶段，如果更改了不会通知。一个SQL的递归查询的例子，生成1,2，...，10的数字序列：
SQL：

```sql
WITH RECURSIVE aux(i) AS (
  VALUES (1)
  UNION ALL
  SELECT i+1 FROM aux WHERE i < 10
)
SELECT * FROM aux
```

可以被生成：在一个TransientTable和一个RepeatUnion上使用scan：

```java
final RelNode node = builder
  .values(new String[] { "i" }, 1)
  .transientScan("aux")
  .filter(
      builder.call(
          SqlStdOperatorTable.LESS_THAN,
          builder.field(0),
          builder.literal(10)))
  .project(
      builder.call(
          SqlStdOperatorTable.PLUS,
          builder.field(0),
          builder.literal(1)))
  .repeatUnion("aux", true)
  .build();
System.out.println(RelOptUtil.toString(node));
```

生成的计划：

```shell
LogicalRepeatUnion(all=[true])
  LogicalTableSpool(readType=[LAZY], writeType=[LAZY], tableName=[aux])
    LogicalValues(tuples=[[{ 1 }]])
  LogicalTableSpool(readType=[LAZY], writeType=[LAZY], tableName=[aux])
    LogicalProject($f0=[+($0, 1)])
      LogicalFilter(condition=[<($0, 10)])
        LogicalTableScan(table=[[aux]])
```

### API

#### Relational operators（关系算子）

下面的方法可以创建一个关系表达式（`RelNode`），push进stack，返回`RelBuilder`。

| METHOD                                                       | DESCRIPTION                                                  |
| :----------------------------------------------------------- | :----------------------------------------------------------- |
| scan(tableName)                                              | 创建一个`TableScan`                                          |
| functionScan(operator, n, expr...) <br />functionScan(operator, n, exprList) | 创建一个`TableFunctionScan`，由最近使用的`n`个关系表达式构成 |
| transientScan(tableName [, rowType])                         | 在一个`TransientTable`上创建一个`TableScan`利用给定的类型（如果没有指明类型，就是用最近使用的关系表达式 |
| values(fieldNames, value...)<br />` `values(rowType, tupleList) | 创建一个`Values`                                             |
| filter([variablesSet, ] exprList)<br />` `filter([variablesSet, ] expr...) | 创建一个`Filter`，将给定的predicates通过交（AND）结合，如果`variablesSet`指定了，则predicates可能参考这些变量 |
| project(expr...)<br />` `project(exprList [, fieldNames])    | 创建一个`Project`。重写默认的名字，使用`alias`修饰表达式，或者指定`fieldNames`参数 |
| projectPlus(expr...)<br />` `projectPlus(exprList)           | `project`的变体，保留了原有字段，append给定的表达式          |
| projectExcept(expr...)<br />` `projectExcept(exprList)       | `project`的变体，保留了原有字段，remove给定的表达式          |
| permute(mapping)                                             | 创建一个`project`，通过`mapping`重排了字段                   |
| convert(rowType [, rename])                                  | 创建了一个`project`，转换字段到给定的类型，也可以重命名这些字段 |
| aggregate(groupKey, aggCall...)<br />` `aggregate(groupKey, aggCallList) | 创建一个`Aggregate`                                          |
| distinct()                                                   | 创建一个`Aggregate`，去除了重复的记录                        |
| sort(fieldOrdinal...)` `sort(expr...)` `sort(exprList)       | 创建一个`Sort`，第一种形式：字段序号从0开始，负数代表倒序，比如-2代表字段1倒序排列。其他形式：可以用`as`、`nullsFirst`或`nullsLast`修饰表达式 |
| sortLimit(offset, fetch, expr...)<br />` `sortLimit(offset, fetch, exprList) | 创建一个具有limit和offset的`Sort`                            |
| limit(offset, fetch)                                         | 创建一个只有limit和offset但是不排序的`Sort`                  |
| exchange(distribution)                                       | 创建一个`Exchange`                                           |
| sortExchange(distribution, collation)                        | 创建一个`SortExchange`                                       |
| correlate(joinType, correlationId, requiredField...)<br />` `correlate(joinType, correlationId, requiredFieldList) | 创建一个`Correlate`，由两个最近的表达式构成，具有一个变量名和左关联所需要的字段表达式 |
| join(joinType, expr...)<br />` `join(joinType, exprList)<br />` `join(joinType, fieldName...) | 创建一个`Join`，有最近的两个关系表达式构成，第一个形式通过一个布尔表达式构建joins（如果有多个通过AND结合）；最后一种形式通过命名的字段构建joins，对于每一个名字每一边要有一个字段对应 |
| semiJoin(expr)                                               | 使用两个最新的关系表达式的半连接类型（SEMI）创建`Join`       |
| antiJoin(expr)                                               | 使用两个最新的关系表达式的反连接类型（ANTI）创建`Join`       |
| union(all [, n])                                             | 创建一个`Unoin`，由最近使用的`n`（默认2个）个关系表达式构成  |
| intersect(all [, n])                                         | 创建一个`Intersect`，由最近使用的`n`（默认2个）个关系表达式构成 |
| minus(all)                                                   | 创建一个`Minus`，由最近使用的2个关系表达式构成               |
| repeatUnion(tableName, all [, n])                            | 创建一个与两个最新关系表达式的`TransientTable`相关联的`RepeatUnion`，迭代次数最多n次(默认为-1，即没有限制)。 |
| snapshot(period)                                             | 创建给定快照时期的`Snapshot`                                 |
| match(pattern, strictStart,` `strictEnd, patterns, measures,` `after, subsets, allRows,` `partitionKeys, orderKeys,` `interval) | 创建一个`Match`                                              |

参数类型：

- `expr`，`interval`：`RexNode`
- `expr...`, `requiredField...`  [RexNode](https://calcite.apache.org/javadocAggregate/org/apache/calcite/rex/RexNode.html) 数组
- `exprList`, `measureList`, `partitionKeys`, `orderKeys`, `requiredFieldList`  [RexNode](https://calcite.apache.org/javadocAggregate/org/apache/calcite/rex/RexNode.html) 的迭代器（Iterable）
- `fieldOrdinal` 字段的序号(从0开始)
- `fieldName` 字段名，独一无二
- `fieldName...` 字符串（字段名）数组
- `fieldNames` 字符串（字段名）的迭代器
- `rowType` [RelDataType](https://calcite.apache.org/javadocAggregate/org/apache/calcite/rel/type/RelDataType.html)
- `groupKey` [RelBuilder.GroupKey](https://calcite.apache.org/javadocAggregate/org/apache/calcite/tools/RelBuilder.GroupKey.html)
- `aggCall...` [RelBuilder.AggCall](https://calcite.apache.org/javadocAggregate/org/apache/calcite/tools/RelBuilder.AggCall.html) 数组
- `aggCallList` [RelBuilder.AggCall](https://calcite.apache.org/javadocAggregate/org/apache/calcite/tools/RelBuilder.AggCall.html)的迭代器
- `value...` 对象（Object）数组
- `value` 对象（Object）
- `tupleList` [RexLiteral](https://calcite.apache.org/javadocAggregate/org/apache/calcite/rex/RexLiteral.html)列表（list）的迭代器
- `all`, `distinct`, `strictStart`, `strictEnd`, `allRows` boolean
- `alias` String 别名
- `correlationId` [CorrelationId](https://calcite.apache.org/javadocAggregate/org/apache/calcite/rel/core/CorrelationId.html)
- `variablesSet`  [CorrelationId](https://calcite.apache.org/javadocAggregate/org/apache/calcite/rel/core/CorrelationId.html) 的迭代器
- `varHolder` [Holder](https://calcite.apache.org/javadocAggregate/org/apache/calcite/util/Holder.html) of [RexCorrelVariable](https://calcite.apache.org/javadocAggregate/org/apache/calcite/rex/RexCorrelVariable.html)
- `patterns` Map whose key is String, value is [RexNode](https://calcite.apache.org/javadocAggregate/org/apache/calcite/rex/RexNode.html)
- `subsets` Map whose key is String, value is a sorted set of String（值是字符串的排序集合）
- `distribution` [RelDistribution](https://calcite.apache.org/javadocAggregate/org/apache/calcite/rel/RelDistribution.html)
- `collation` [RelCollation](https://calcite.apache.org/javadocAggregate/org/apache/calcite/rel/RelCollation.html)（collation：校对）
- `operator` [SqlOperator](https://calcite.apache.org/javadocAggregate/org/apache/calcite/sql/SqlOperator.html)
- `joinType` [JoinRelType](https://calcite.apache.org/javadocAggregate/org/apache/calcite/rel/core/JoinRelType.html)

用于执行不同优化的builder方法：

- `project` 如果需要按顺序投影所有的列，那么直接返回input（输入）
- `filter` 平坦化条件（因此AND和OR可能有超过两个子元素）；简化（比如将  `x = 1 AND TRUE` 简化为 `x = 1`）
- 如果想使用`sort`和`limit`，效果和直接使用`sortlimit`是一样的

一些可以在**栈顶**的关系表达式添加信息的注解方法：

| METHOD              | DESCRIPTION                                   |
| ------------------- | --------------------------------------------- |
| as(alias)           | 给**栈顶**的关系表达式分配table alias(表别名) |
| variable(varHolder) | 给**栈顶**的关系表达式创建一个关联变量        |

#### 堆栈方法

| METHOD              | DESCRIPTION                                                  |
| ------------------- | ------------------------------------------------------------ |
| build()             | 从堆栈弹出最近的关系表达式（pop）                            |
| push(rel)           | 将关系表达式压入栈中，一般而言，像`scan`这样的关系表达式会调用，用户代码不调用 |
| pushAll(collection) | 将一个集合的关系表达式压入栈中                               |
| peek()              | 返回最新被压入栈中的关系表达式，但是不会从栈中remove         |

#### 标量表达式方法（Scalar expression methods)

下面的方法会返回标量表达式 ([RexNode](https://calcite.apache.org/javadocAggregate/org/apache/calcite/rex/RexNode.html)).

它们之中的许多方法需要使用到栈中的内容。例如，`field("DEPTNO")`返回刚刚放进栈的关系表达式的“DEPTNO”字段的引用。

| METHOD                                                       | DESCRIPTION                                                  |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| `literal(value)`                                             | 常量                                                         |
| `field(fieldName)`                                           | 通过字段名获取栈顶的关系表达式的字段                         |
| `field(fieldOrdinal)`                                        | 通过字段序号获取栈顶的关系表达式的字段                       |
| `field(inputCount, inputOrdinal, fieldName)`                 | 通过字段名引用栈中第(`inputCount` - `inputOrdinal`)个表达式的字段 |
| `field(inputCount, inputOrdinal, fieldOrdinal)`              | 通过字段序号引用栈中第(`inputCount` - `inputOrdinal`)个表达式的字段 |
| `field(alias, fieldName)`                                    | 通过字段名和表别名引用栈顶的关系表达式的字段                 |
| `field(expr, fieldName)`                                     | 通过字段名引用*值记录表达式*的字段                           |
| `field(expr, fieldOrdinal)`                                  | 通过字段序号引用*值记录表达式*的字段                         |
| `fields(fieldOrdinalList)`                                   | 按字段序号引用输入字段的关系表达式列表                       |
| `fields(mapping)`                                            | 通过给定映射引用输入字段的表达式列表                         |
| `fields(collation)`                                          | 表达式列表， `exprList`, `sort(exprList)` 会复制排序         |
| `call(op, expr...)` <br />`call(op, exprList)`               | 调用一个函数（function）或算子（operator）                   |
| `and(expr...)`<br /> `and(exprList)`                         | 逻辑AND. 铺平嵌套的 ANDs, 优化涉及到TRUE和FALSE的场景        |
| `or(expr...)` <br />`or(exprList)`                           | 逻辑OR. 铺平嵌套的 ORs, 优化涉及到TRUE和FALSE的场景          |
| `not(expr)`                                                  | 逻辑NOT                                                      |
| `equals(expr, expr)`                                         | 等于                                                         |
| `isNull(expr)`                                               | 检查表达式是否为空                                           |
| `isNotNull(expr)`                                            | 检查表达式是否非空                                           |
| `alias(expr, fieldName)`                                     | 重命名一个表达式（仅仅在作为`project`的一个参数时有效        |
| `cast(expr, typeName)` <br />`cast(expr, typeName, precision)` `cast(expr, typeName, precision, scale)` | 将一个表达式转换为指定的类型                                 |
| `desc(expr)`                                                 | 将排序方向改为逆序 (仅仅在作为 `sort` 或者 `sortLimit`的参数时有效) |
| `nullsFirst(expr)`                                           | 将排序方向改为null优先(仅仅在作为 `sort` 或者 `sortLimit`的参数时有效) |
| `nullsLast(expr)`                                            | 将排序方向改为null最后(仅仅在作为 `sort` 或者 `sortLimit`的参数时有效) |
| `cursor(n, input)`                                           | 对`TableFunctionScan` 的`n`个关系型输入中第`input`个 (从0开始)输入的引用(具体见 `functionScan`) |

#### 模式方法（Pattern methods)

下面方法返回在`match`中使用的模式（patterns)

| METHOD                               | DESCRIPTION  |
| ------------------------------------ | ------------ |
| `patternConcat(pattern...)`          | 连接patterns |
| `patternAlter(pattern...)`           | 替换patterns |
| `patternQuantify(pattern, min, max)` | 量化pattern  |
| `patternPermute(pattern...)`         | 重排pattern  |
| `patternExclude(pattern)`            | 去除pattern  |

#### Group key methods

下面的方法返回 [RelBuilder.GroupKey](https://calcite.apache.org/javadocAggregate/org/apache/calcite/tools/RelBuilder.GroupKey.html)

| METHOD                                                       | DESCRIPTION                                                  |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| `groupKey(fieldName...)` <br />`groupKey(fieldOrdinal...)` <br />`groupKey(expr...)` <br />`groupKey(exprList)` | 从给定的表达式创建group key                                  |
| `groupKey(exprList, exprListList)`                           | 从带有分组集合（group set）的给定表达式中创建group key       |
| `groupKey(bitSet [, bitSets])`                               | 如果`bitSets`被指定的话，从带有多个分组集合（group set）的给定表达式中创建group key |

#### Aggregate call methods

下面的方法返回 [RelBuilder.AggCall](https://calcite.apache.org/javadocAggregate/org/apache/calcite/tools/RelBuilder.AggCall.html).

| METHOD                                                       | DESCRIPTION                          |
| ------------------------------------------------------------ | ------------------------------------ |
| `aggregateCall(op, expr...)` <br />`aggregateCall(op, exprList)` | 由给定的聚合函数创建call（调用）     |
| `count([ distinct, alias, ] expr...)` <br />`count([ distinct, alias, ] exprList)` | 创建一个对 `COUNT` 聚合函数的调用    |
| `countStar(alias)`                                           | 创建一个对 `COUNT(*)` 聚合函数的调用 |
| `sum([ distinct, alias, ] expr)`                             | 创建一个对 `SUM` 聚合函数的调用      |
| `min([ alias, ] expr)`                                       | 创建一个对 MIN 聚合函数的调用        |
| `max([ alias, ] expr)`                                       | 创建一个对 MAX聚合函数的调用         |

为了进一步修改`AggCall`，可以调用它的方法：

| METHOD                           | DESCRIPTION                                        |
| -------------------------------- | -------------------------------------------------- |
| `approximate(approximate)`       | 允许 `approximate`聚合的近似值                     |
| `as(alias)`                      | 分配列的别名给表达式(see SQL `AS`)                 |
| `distinct()`                     | 在聚合之前去除重复的值 (see SQL `DISTINCT`)        |
| `distinct(distinct)`             | 如果`distinct`，则在聚合之前去除重复的值           |
| `filter(expr)`                   | 在聚合之前过滤一些行(see SQL `FILTER (WHERE ...)`) |
| `sort(expr...)` `sort(exprList)` | 在过滤之前排序一些行 (see SQL `WITHIN GROUP`)      |

