## Calcite 学习笔记3-Adapters

### Adapters

#### Schema adapters

schema adapter 允许Calcite读取特定类型的数据，并且以在schema中的tables的形式表示数据。

- [Cassandra adapter](https://calcite.apache.org/docs/cassandra_adapter.html) ([calcite-cassandra](https://calcite.apache.org/javadocAggregate/org/apache/calcite/adapter/cassandra/package-summary.html))
- CSV adapter ([example/csv](https://calcite.apache.org/javadocAggregate/org/apache/calcite/adapter/csv/package-summary.html))
- [Druid adapter](https://calcite.apache.org/docs/druid_adapter.html) ([calcite-druid](https://calcite.apache.org/javadocAggregate/org/apache/calcite/adapter/druid/package-summary.html))
- [Elasticsearch adapter](https://calcite.apache.org/docs/elasticsearch_adapter.html) ([calcite-elasticsearch](https://calcite.apache.org/javadocAggregate/org/apache/calcite/adapter/elasticsearch/package-summary.html))
- [File adapter](https://calcite.apache.org/docs/file_adapter.html) ([calcite-file](https://calcite.apache.org/javadocAggregate/org/apache/calcite/adapter/file/package-summary.html))
- [Geode adapter](https://calcite.apache.org/docs/geode_adapter.html) ([calcite-geode](https://calcite.apache.org/javadocAggregate/org/apache/calcite/adapter/geode/package-summary.html))
- JDBC adapter (part of [calcite-core](https://calcite.apache.org/javadocAggregate/org/apache/calcite/adapter/jdbc/package-summary.html))
- MongoDB adapter ([calcite-mongodb](https://calcite.apache.org/javadocAggregate/org/apache/calcite/adapter/mongodb/package-summary.html))
- [OS adapter](https://calcite.apache.org/docs/os_adapter.html) ([calcite-os](https://calcite.apache.org/javadocAggregate/org/apache/calcite/adapter/os/package-summary.html))
- [Pig adapter](https://calcite.apache.org/docs/pig_adapter.html) ([calcite-pig](https://calcite.apache.org/javadocAggregate/org/apache/calcite/adapter/pig/package-summary.html))
- [Redis adapter](https://calcite.apache.org/docs/redis_adapter.html) ([calcite-redis](https://calcite.apache.org/javadocAggregate/org/apache/calcite/adapter/redis/package-summary.html))
- Solr cloud adapter ([solr-sql](https://github.com/bluejoe2008/solr-sql))
- Spark adapter ([calcite-spark](https://calcite.apache.org/javadocAggregate/org/apache/calcite/adapter/spark/package-summary.html))
- Splunk adapter ([calcite-splunk](https://calcite.apache.org/javadocAggregate/org/apache/calcite/adapter/splunk/package-summary.html))
- Eclipse Memory Analyzer (MAT) adapter ([mat-calcite-plugin](https://github.com/vlsi/mat-calcite-plugin))
- [Apache Kafka adapter](https://calcite.apache.org/docs/kafka_adapter.html)

##### 其他语言接口

- Piglet ([calcite-piglet](https://calcite.apache.org/javadocAggregate/org/apache/calcite/piglet/package-summary.html)) 在[Pig Latin](https://pig.apache.org/docs/r0.7.0/piglatin_ref1.html) 的子集上运行查询

#### Engines

许多项目使用Calcite作为SQL解析、查询优化、数据可视化/整合以及可视化视图的重写。其中的一些在 [“powered by Calcite”](https://calcite.apache.org/docs/powered_by.html) 页中展示。

#### Drivers

driver用来在自己的程序中连接Calcite

- [JDBC driver](https://calcite.apache.org/javadocAggregate/org/apache/calcite/jdbc/package-summary.html)

JDBC driver由[Avatica](https://calcite.apache.org/avatica/docs/) 提供。连接可以是本地的也可以是远程的（JSON over HTTP or Protobuf over HTTP）

最基本的JDBC连接字符串的形式为：

jdbc:calcite:property=value;property2=value2

属性在下面中描述。连接字符串符合OLEDB连接字符串的语法，由Avatica的

 [ConnectStringParser](https://calcite.apache.org/avatica/apidocs/org/apache/calcite/avatica/ConnectStringParser.html) 实现。

####JDBC连接字符串属性说明

| PROPERTY                                                     | DESCRIPTION                                                  |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| [approximateDecimal](https://calcite.apache.org/javadocAggregate/org/apache/calcite/config/CalciteConnectionProperty.html#APPROXIMATE_DECIMAL) | `DECIMAL`类型上的聚合函数的近似结果是否可接受                |
| [approximateDistinctCount](https://calcite.apache.org/javadocAggregate/org/apache/calcite/config/CalciteConnectionProperty.html#APPROXIMATE_DISTINCT_COUNT) | `COUNT(DISTINCT ...)` 聚合函数的近似结果是否可以接受         |
| [approximateTopN](https://calcite.apache.org/javadocAggregate/org/apache/calcite/config/CalciteConnectionProperty.html#APPROXIMATE_TOP_N) | “Top N” 查询的形式 (`ORDER BY aggFun() DESC LIMIT n`) 的近似结果是否可以接受 |
| [caseSensitive](https://calcite.apache.org/javadocAggregate/org/apache/calcite/config/CalciteConnectionProperty.html#CASE_SENSITIVE) | 是否标识符对大小写敏感，如果没有指明，使用`lex`的值          |
| [conformance](https://calcite.apache.org/javadocAggregate/org/apache/calcite/config/CalciteConnectionProperty.html#CONFORMANCE) | SQL 一致性等级. 取值: DEFAULT (默认的，类似于PRAGMATIC_2003), LENIENT, MYSQL_5, ORACLE_10, ORACLE_12, PRAGMATIC_99, PRAGMATIC_2003, STRICT_92, STRICT_99, STRICT_2003, SQL_SERVER_2008. |
| [createMaterializations](https://calcite.apache.org/javadocAggregate/org/apache/calcite/config/CalciteConnectionProperty.html#CREATE_MATERIALIZATIONS) | 是否允许Calcite创建实体，默认不可以                          |
| [defaultNullCollation](https://calcite.apache.org/javadocAggregate/org/apache/calcite/config/CalciteConnectionProperty.html#DEFAULT_NULL_COLLATION) | 查询中的NULL应该被如何排序，在 NULLS FIRST 和 NULLS LAST都没有被指明的情况下。默认的是HIGH，和Oracle数据库中对NULL的处理一样。 |
| [druidFetch](https://calcite.apache.org/javadocAggregate/org/apache/calcite/config/CalciteConnectionProperty.html#DRUID_FETCH) | 使用Druid adapter时，一次SELECT返回多少行                    |
| [forceDecorrelate](https://calcite.apache.org/javadocAggregate/org/apache/calcite/config/CalciteConnectionProperty.html#FORCE_DECORRELATE) | 是否planner应该尽可能的解关联（*de-correlating*），默认是    |
| [fun](https://calcite.apache.org/javadocAggregate/org/apache/calcite/config/CalciteConnectionProperty.html#FUN) | 内建函数和算则的集合。 有效的值是：“标准的”（默认）“oracle”, “spatial”, 多个结合使用逗号，比如oracle,spatial”. |
| [lex](https://calcite.apache.org/javadocAggregate/org/apache/calcite/config/CalciteConnectionProperty.html#LEX) | 词汇策略. 取值可以使ORACLE (默认的), MYSQL, MYSQL_ANSI, SQL_SERVER, JAVA. |
| [materializationsEnabled](https://calcite.apache.org/javadocAggregate/org/apache/calcite/config/CalciteConnectionProperty.html#MATERIALIZATIONS_ENABLED) | 是否Calcite可以使用实体化（matrialization），默认是不可以    |
| [model](https://calcite.apache.org/javadocAggregate/org/apache/calcite/config/CalciteConnectionProperty.html#MODEL) | JSON/YAML 模型文件的url 或者 内联的（inline） 比如 `inline:{...}` for JSON and `inline:...` for YAML. |
| [parserFactory](https://calcite.apache.org/javadocAggregate/org/apache/calcite/config/CalciteConnectionProperty.html#PARSER_FACTORY) | 解析器工厂类. 实现了接口 [`interface SqlParserImplFactory`](https://calcite.apache.org/javadocAggregate/org/apache/calcite/sql/parser/SqlParserImplFactory.html) 的类名，并且有一个公有的默认构造器和`INSTANCE`常量 |
| [quoting](https://calcite.apache.org/javadocAggregate/org/apache/calcite/config/CalciteConnectionProperty.html#QUOTING) | 如何引用标识符。 可取值：DOUBLE_QUOTE, BACK_QUOTE, BRACKET.如果没有指明，会使用`lex`中的值 |
| [quotedCasing](https://calcite.apache.org/javadocAggregate/org/apache/calcite/config/CalciteConnectionProperty.html#QUOTED_CASING) | 如果标识符被引用了应该怎么被存储。 可取值：UNCHANGED, TO_UPPER, TO_LOWER.如果没有指明，会使用`lex`中的值 |
| [schema](https://calcite.apache.org/javadocAggregate/org/apache/calcite/config/CalciteConnectionProperty.html#SCHEMA) | 初始化schema的名称.                                          |
| [schemaFactory](https://calcite.apache.org/javadocAggregate/org/apache/calcite/config/CalciteConnectionProperty.html#SCHEMA_FACTORY) | Schema 工厂类 实现了接口 [`interface SchemaFactory`](https://calcite.apache.org/javadocAggregate/org/apache/calcite/schema/SchemaFactory.html) 的类名 ，并且有一个公有的默认构造器和`INSTANCE`常量 |
| [schemaType](https://calcite.apache.org/javadocAggregate/org/apache/calcite/config/CalciteConnectionProperty.html#SCHEMA_TYPE) | Schema 类型. 取值必须是： “MAP” (默认的), “JDBC”, or “CUSTOM” (如果指定了`schemaFactory`，则为隐式). 如果`model`被指明了，则忽略 |
| [spark](https://calcite.apache.org/javadocAggregate/org/apache/calcite/config/CalciteConnectionProperty.html#SPARK) | 指明如果无法下推到数据源时，是否使用spark作为数据处理的引擎，如果为false（默认的），Calcite会生成实现了Enumerable接口的代码 |
| [timeZone](https://calcite.apache.org/javadocAggregate/org/apache/calcite/config/CalciteConnectionProperty.html#TIME_ZONE) | 时区, 例如 “gmt-3”. 默认是JVM的时区.                         |
| [typeSystem](https://calcite.apache.org/javadocAggregate/org/apache/calcite/config/CalciteConnectionProperty.html#TYPE_SYSTEM) | 类型系统. 实现了接口 [`interface RelDataTypeSystem`](https://calcite.apache.org/javadocAggregate/org/apache/calcite/rel/type/RelDataTypeSystem.html) 的类名，并且有一个公有的默认构造器和`INSTANCE`常量 |
| [unquotedCasing](https://calcite.apache.org/javadocAggregate/org/apache/calcite/config/CalciteConnectionProperty.html#UNQUOTED_CASING) | 如果标识符没有被引用应该怎么被存储。 可取值：UNCHANGED, TO_UPPER, TO_LOWER.如果没有指明，会使用`lex`中的值 |
| [typeCoercion](https://calcite.apache.org/javadocAggregate/org/apache/calcite/config/CalciteConnectionProperty.html#TYPE_COERCION) | 在sql节点验证过程中，当类型不匹配时，是否进行隐式类型强制转换，默认值为true |

连接单一的内置schema类型的schema，不需要指明model，例如：

`jdbc:calcite:schemaType=JDBC; schema.jdbcUser=SCOTT; schema.jdbcPassword=TIGER; schema.jdbcUrl=jdbc:hsqldb:res:foodmart`

创建了一个到通过JDBC schema adapter到foodmart 数据库映射的shema的连接。

类似的，也可以连接到单一的用户自定义shema adapter的schema，例如：

`jdbc:calcite:schemaFactory=org.apache.calcite.adapter.cassandra.CassandraSchemaFactory; schema.host=localhost; schema.keyspace=twissandra`

建立了一个到Cassandra adapter的连接，等价于下面的model文件：

```json
{
  "version": "1.0",
  "defaultSchema": "foodmart",
  "schemas": [
    {
      type: 'custom',
      name: 'twissandra',
      factory: 'org.apache.calcite.adapter.cassandra.CassandraSchemaFactory',
      operand: {
        host: 'localhost',
        keyspace: 'twissandra'
      }
    }
  ]
}
```

注意`operand`中的每一个key，在连接字符串上都有一个`schema`前缀

#### Server

Calcite的核心模块（`calcite-core`）支持SQL查询（`SELECT`）以及DML操作（`INSERT`,`UPDATE`,`DELETE`,`MERGE`）但是不支持DDL操作，例如`CREATE SCHEMA` 或是 `CREATE TABLE` 。DDL使存储库的状态模型变得复杂，并使解析器的扩展变得更加困难，因此我们将DDL排除在核心之外。

服务器模块（`calcite-server`）给Calcite提供了DDL支持。它扩展了SQL解析器，使用了和子项目一样的组织结构，添加了一些DDL指令：

- `CREATE` and `DROP SCHEMA`
- `CREATE` and `DROP FOREIGN SCHEMA`
- `CREATE` and `DROP TABLE` (包括`CREATE TABLE ... AS SELECT`)
- `CREATE` and `DROP MATERIALIZED VIEW`
- `CREATE` and `DROP VIEW`
- `CREATE` and `DROP FUNCTION`
- `CREATE` and `DROP TYPE`

具体的指令及其描述在 [SQL reference](https://calcite.apache.org/docs/reference.html#ddl-extensions) 中可以找到。

为了使用这些功能，需要将`calcite-server.jar`include进classpath，然后再JDBC连接字符串中添加`parserFactory=org.apache.calcite.sql.parser.ddl.SqlDdlParserImpl#FACTORY`. 有一个使用 `sqlline `shell的例子：

```shell
$ ./sqlline
sqlline version 1.3.0
> !connect jdbc:calcite:parserFactory=org.apache.calcite.sql.parser.ddl.SqlDdlParserImpl#FACTORY sa ""
> CREATE TABLE t (i INTEGER, j VARCHAR(10));
No rows affected (0.293 seconds)
> INSERT INTO t VALUES (1, 'a'), (2, 'bc');
2 rows affected (0.873 seconds)
> CREATE VIEW v AS SELECT * FROM t WHERE i > 1;
No rows affected (0.072 seconds)
> SELECT count(*) FROM v;
+---------------------+
|       EXPR$0        |
+---------------------+
| 1                   |
+---------------------+
1 row selected (0.148 seconds)
> !quit
```

`calcite-server`模块是可选的，它的目标之一在于提供你可以通过使用SQL命令行的示例展示Calcite的功能（例如可视化的视图，foreign tables以及 genrated columns）的场合。所以这些可以展示的功能在`calcite-core` 的APIs中获得。

如果你是子项目的作者，你的语法扩展是不可能匹配`calcite-server`的，所以我们建议你通过扩展核心解析器的方式（ [extending the core parser](https://calcite.apache.org/docs/adapter.html#extending-the-parser)）添加你的SQL语法拓展；如果你想使用DDL命令，你可能需要将`calcite-server`复制粘贴到你的项目中。

到目前为止，存储库还没有被持久化。如果你执行了DDL指令，你在修改一个内存中的存储库，并且添加和删除从一个根Schema可以到达的对象，在一个SQL session（会话）中所有的指令都会看到这些对象。你可以用一个SQL指令的脚本在之后的session中创建相同的对象。

Calcite也可以作为数据可视化和数据整合的服务器：Calcite管理来自多个foreign schema的数据，但是对于客户端来说，就好像这个数据都是在同一个地方，Calcite选择数据处理应该在哪里进行， 并且根据效率决定数据是否创建数据的copies。`calcite-server`模块朝着这个目标迈出的一步：工业级的解决方案，需要结合仓库持久化、鉴权、安全等功能的完善，Calcite作为一个服务运行。

#### Extensibility

有许多APIs可以拓展Calcite的功能。

在这一节我们将简单地介绍这些API，讲明白什么是可能的。为了更加详细地了解这些APIs，你可能需要读其他的docs，比如接口的javadocs等等，深圳有可能你需要去看一些我们写的测试。

##### Functions and operators

有许多方法可以给Calcite添加functions和operators。我们首先介绍最简单的（也是最弱鸡的）。

*User-defined functions* 就是最简单的但是最弱鸡的，他们写起来最简单（你只需要写一个Java类，并将他们注册到你的schema中），但是在参数的数量和类型、解析重载函数或派生返回类型方面没有提供太多的灵活性。

如果你需要灵活性，你需要写 *user-defined operator*（see [`interface SqlOperator`](https://calcite.apache.org/javadocAggregate/org/apache/calcite/sql/SqlOperator.html)）

如果你的operator不符合标准的SQL语法：`f(arg1, arg2, ...)`，你需要扩展解析器

在测试里面有许多很好的例子：[`class UdfTest`](https://github.com/apache/calcite/blob/master/core/src/test/java/org/apache/calcite/test/UdfTest.java) 测试了用户自定义的functions以及用户自定义的aggregate functions。

##### Aggregate functions

*User-defined aggregate functions*和user-defined functions类似，但是每一个fucntion有一些相应的Java方法，每一个对应aggregate中的生命周期的一个stage（阶段）

- `init` 创建一个 accumulator（累加器）;
- `add` 添加一行的值到 accumulator;
- `merge` 结合两个accumulators 成为一个;
- `result` 完成accumulator 并转化为结果.

例如，应用在`SUM(int)` 的方法（伪码）如下：

```pseudocode
struct Accumulator {
  final int sum;
}
Accumulator init() {
  return new Accumulator(0);
}
Accumulator add(Accumulator a, int x) {
  return new Accumulator(a.sum + x);
}
Accumulator merge(Accumulator a, Accumulator a2) {
  return new Accumulator(a.sum + a2.sum);
}
int result(Accumulator a) {
  return new Accumulator(a.sum + x);
}
```

下面是计算一列中两行数据之和的调用序列：

```pseudocode
a = init()    # a = {0}
a = add(a, 4) # a = {4}
a = add(a, 7) # a = {11}
return result(a) # returns 11
```

##### Window functions

window function和aggregate function很类似，但是需要把`GROUP BY`子句换成`OVER`，每个aggregate function可被用作window function， 但是有一些关键差异。被window function看到的行可能是有序的，而且本身就依赖顺序的window functions（比如`RANK`）就不能被用作aggregate func.

另一个不同之处在于windows是不相交的(*non-disjoint*) ：特定的行可以出现在超过一个window，比如，9:37可能出现在9:00-10:00，也可能出现在9:15-9:45.

windows functions是被递增地计算的：当时钟从10:14到10:15，两行可能进入window，三行离开，window functions 有一个额外的生命周期的操作：

`remove` 从accumulator中移除一个值.

`SUM(int)`的伪码变为：

```pseudocode
Accumulator remove(Accumulator a, int x) {
  return new Accumulator(a.sum - x);
}
```

这里是一个计算移动的累加值的调动顺序，4行的值分别为4,7,2和3(emit:发出，释放)

```pseudocode
a = init()       # a = {0}
a = add(a, 4)    # a = {4}
emit result(a)   # emits 4
a = add(a, 7)    # a = {11}
emit result(a)   # emits 11
a = remove(a, 4) # a = {7}
a = add(a, 2)    # a = {9}
emit result(a)   # emits 9
a = remove(a, 7) # a = {2}
a = add(a, 3)    # a = {5}
emit result(a)   # emits 5
```

##### Grouped window functions

Grouped window是那些使用`GROUP BY`操作将一些数据聚集到一个集合里面的functions。内置的grouped window functions有`HOP`,`TUMBLE`和`SESSION`.你可以通过实现 [`interface SqlGroupedWindowFunction`](https://calcite.apache.org/javadocAggregate/org/apache/calcite/sql/fun/SqlGroupedWindowFunction.html) 来定义其他的functions。

##### Table functions and table macros

*User-defined table functions*和常规的标量（“scalar”）用户自定义函数（udf）具有相同的定义方式，但是在查询中需要一个`FROM`子句。下面的查询使用了一个叫`Ramp`的table function。

```SQL
SELECT * FROM TABLE(Ramp(3, 4))
```

*User-defined table* 宏（macro）具有和table fucntions一样的SQL语法，但是定义上会有一些差别。生成表达式而不是生成数据。（*Rather than generating data, they generate an relational expression*）。在准备查询期间调用表宏（table macro），然后可以优化它们生成的关系表达式。（Calcite 在views的实现上使用了table macros)

[`class TableFunctionTest`](https://github.com/apache/calcite/blob/master/core/src/test/java/org/apache/calcite/test/TableFunctionTest.java) 测试了table fuctions并且包含了一些有用的示例。

##### Extending the parser

假设您需要扩展Calcite的SQL语法，使其与将来对语法的修改兼容。如果copy一份`Parser.jj`文件到你的项目里面是非常愚蠢的，因为这个文件经常需要修改。

幸运的是，`Parser.jj`实际上是包含了可以被修改的变量的 [Apache FreeMarker](https://freemarker.apache.org/) 模板文件。`calciete-core` 中的解析器初始化模板，将变量置初值，通常是空的，但是你可以重写。如果你想要一个不同的解析器，你可以提供你自己的`config.fmpp`和`parserImpls.ftl` 文件，这样就可以生成一个扩展的解析器。

`calcite-server` 模块在[[CALCITE-707](https://issues.apache.org/jira/browse/CALCITE-707)]  被创建，添加了一些像`CREATE TABLE`这样的DDL语句，是一个你可以参照的例子。参照 [`class ExtensionSqlParserTest`](https://github.com/apache/calcite/blob/master/core/src/test/java/org/apache/calcite/sql/parser/parserextensiontesting/ExtensionSqlParserTest.java)。

##### Customizing SQL dialect accepted and generated

为了自定义什么样的SQL拓展解析器应该接受，实现接口[`interface SqlConformance`](https://calcite.apache.org/javadocAggregate/org/apache/calcite/sql/validate/SqlConformance.html) ，或者使用[`enum SqlConformanceEnum`](https://calcite.apache.org/javadocAggregate/org/apache/calcite/sql/validate/SqlConformanceEnum.html) 中的内置值。

为了控制SQL如何被外部的数据库（通常通过JDBC）生成，使用类 [`class SqlDialect`](https://calcite.apache.org/javadocAggregate/org/apache/calcite/sql/SqlDialect.html) 。方言描述了引擎的功能，比如是否支持`OFFSET`和`FETCH` 从句等。

##### Defining a custom schema

定义一个自定义的shema，你需要实现[`interface SchemaFactory`](https://calcite.apache.org/javadocAggregate/org/apache/calcite/schema/SchemaFactory.html) 。

在查询的准备阶段，Calcite会调用这个接口去查询你的schema包含了哪些ables和sub-schemas。当某个查询引用了你的schema中的一个table，Calcite会让你的schema创建一个[`interface Table`](https://calcite.apache.org/javadocAggregate/org/apache/calcite/schema/Table.html) 的实例。

这个table会被包装进一个`TableScan`， 然后进行查询优化过程。

##### Reflective schema

> A reflective schema ([`class ReflectiveSchema`](https://calcite.apache.org/javadocAggregate/org/apache/calcite/adapter/java/ReflectiveSchema.html)) is a way of wrapping a Java object so that it appears as a schema. Its collection-valued fields will appear as tables.

反射schema（[`class ReflectiveSchema`](https://calcite.apache.org/javadocAggregate/org/apache/calcite/adapter/java/ReflectiveSchema.html)）是一种包装Java对象的方法，使其能够以schema的方式呈现，它的集合值字段将显示为table。

它不是一个schema的工厂类，而是一个实际的schema，你必须创建对象，然后通过调用APIs将它包装进schema。

##### Defining a custom table

定义一个自定义的table，需要实现[`interface TableFactory`](https://calcite.apache.org/javadocAggregate/org/apache/calcite/schema/TableFactory.html) 。因为schema有一系列命名的tables组成，table factory生成的单个table在与schema绑定的时候需要制定一个特定的名字（一些额外的操作数是可选的）

##### Modifying data

如果你的表需要支持流查询（streaming queries），你对`interface Table` 的实现必须要实现[`interface StreamableTable`](https://calcite.apache.org/javadocAggregate/org/apache/calcite/schema/StreamableTable.html)。

可以在 [`class StreamTest `](https://github.com/apache/calcite/blob/master/core/src/test/java/org/apache/calcite/test/StreamTest.java) 中找到示例。

##### Streaming

如果你的表需要支持DML（INSERT,UPDATE,DETELE,MERGE），你对`interface Table` 的实现必须要实现[`interface ModifiableTable`](https://calcite.apache.org/javadocAggregate/org/apache/calcite/schema/ModifiableTable.html) 。

##### Pushing operations down to your table

如果你想要将处理过程（processing)下推导自定义表的源系统，考虑实现[`interface FilterableTable`](https://calcite.apache.org/javadocAggregate/org/apache/calcite/schema/FilterableTable.html) 或 [`interface ProjectableFilterableTable`](https://calcite.apache.org/javadocAggregate/org/apache/calcite/schema/ProjectableFilterableTable.html).

如果想实现更多的控制，你应该写一个 [planner rule](https://calcite.apache.org/docs/adapter.html#planner-rule) .这允许你下推表达式， 做一个cost-based的decision是否下推processing，并且下推复杂的算子，比如join，aggregate，sort等。

##### Type system

你可以自定义类型系统的一些方面，实现[`interface RelDataTypeSystem`](https://calcite.apache.org/javadocAggregate/org/apache/calcite/rel/type/RelDataTypeSystem.html) 。

##### Relational operators

所有的关系算子实现了接口 [`interface RelNode`](https://calcite.apache.org/javadocAggregate/org/apache/calcite/rel/RelNode.html) ，大部分继承了 [`class AbstractRelNode`](https://calcite.apache.org/javadocAggregate/org/apache/calcite/rel/AbstractRelNode.html). 核心算子 (被 [`SqlToRelConverter`](https://calcite.apache.org/javadocAggregate/org/apache/calcite/sql2rel/SqlToRelConverter.html)  使用的 以及 覆盖了传统的关系代数) 是 [`TableScan`](https://calcite.apache.org/javadocAggregate/org/apache/calcite/rel/core/TableScan.html), [`TableModify`](https://calcite.apache.org/javadocAggregate/org/apache/calcite/rel/core/TableModify.html), [`Values`](https://calcite.apache.org/javadocAggregate/org/apache/calcite/rel/core/Values.html), [`Project`](https://calcite.apache.org/javadocAggregate/org/apache/calcite/rel/core/Project.html), [`Filter`](https://calcite.apache.org/javadocAggregate/org/apache/calcite/rel/core/Filter.html), [`Aggregate`](https://calcite.apache.org/javadocAggregate/org/apache/calcite/rel/core/Aggregate.html), [`Join`](https://calcite.apache.org/javadocAggregate/org/apache/calcite/rel/core/Join.html), [`Sort`](https://calcite.apache.org/javadocAggregate/org/apache/calcite/rel/core/Sort.html), [`Union`](https://calcite.apache.org/javadocAggregate/org/apache/calcite/rel/core/Union.html), [`Intersect`](https://calcite.apache.org/javadocAggregate/org/apache/calcite/rel/core/Intersect.html), [`Minus`](https://calcite.apache.org/javadocAggregate/org/apache/calcite/rel/core/Minus.html), [`Window`](https://calcite.apache.org/javadocAggregate/org/apache/calcite/rel/core/Window.html) 以及[`Match`](https://calcite.apache.org/javadocAggregate/org/apache/calcite/rel/core/Match.html).

以上的这些每个都有一个纯粹的（“pure”）逻辑子类，[`LogicalProject`](https://calcite.apache.org/javadocAggregate/org/apache/calcite/rel/logical/LogicalProject.html) 等等。任何adapter将有这些算子的副本，使得它的引擎可以实现的更加高效。比如，Cassandra adapter 有 [`CassandraProject`](https://calcite.apache.org/javadocAggregate/org/apache/calcite/rel/cassandra/CassandraProject.html) 但是没有 `CassandraJoin`.

你可以定义自己的`RelNode` 的子类去添加一个新的算子或是一个现有的某个特定的引擎的算子的实现。

为了使一个算子更加的有用和有效，你往往需要[planner rules](https://calcite.apache.org/docs/adapter.html#planner-rule) 结合现有的算子（也需要提供元数据，看 [below](https://calcite.apache.org/docs/adapter.html#statistics-and-cost) ）。这是代数运算，结果是组合的:你写了一些新的规则，但它们结合起来可以处理指数级别的查询模式（patterns）。

如果可能的话，尽量使你的算子是现有算子的子类，那你就可以重用或是适用（adapt）这些算子的规则。更好的是，如果您的算子是一个逻辑操作，您可以根据现有的算子重写它(同样的，通过一个planner rules)，那么您应该这样做。您将能够重用这些算子的规则、元数据和实现，而不需要额外的工作。

##### Planner rule

一个 planner rule ([`class RelOptRule`](https://calcite.apache.org/javadocAggregate/org/apache/calcite/plan/RelOptRule.html)) 将一个关系表达式转换为另一个等效的关系表达式。

一个 planner engine 有许多注册的规则并触发它们将输入的查询转换成更加高效的查询。 因此，planner rules 是优化过程的核心，但令人惊讶的是，每个计划规则本身与成本无关。planner engine 负责通过按照一定的顺序触发规则来生成最有的计划，但是每个规则只关注其正确性。

Calcite 有两个内置的 planner engines: [`class VolcanoPlanner`](https://calcite.apache.org/javadocAggregate/org/apache/calcite/plan/volcano/VolcanoPlanner.html) 动态规划，有利于穷举搜索  而  [`class HepPlanner`](https://calcite.apache.org/javadocAggregate/org/apache/calcite/plan/hep/HepPlanner.html) 以更固定的顺序触发一系列规则。

##### Calling conventions

一个calling convention（调用约定） 特定的数据引擎使用的协议. 例如, Cassandra engine 有一个关系表达式的集合, `CassandraProject`, `CassandraFilter` 等等, 这些算子可以相互连接， 而不需要将数据从一种格式转换成另一种格式。

如果需要将数据从一种调用约定转换为另一种调用约定，那么Calcite使用一种称为转换器的关系表达式的特殊子类(参照[`class Converter`](https://calcite.apache.org/javadocAggregate/org/apache/calcite/rel/convert/Converter.html))。但是，当然，转换数据有运行时成本。

当计划使用多个引擎的查询时，Calcite将根据不同的调用约定将不同的关系表达式树的区域“染色”。planner通过触发规则将操作下推到数据源。如果引擎不支持特定的操作，则规则将不会触发。有时一个操作可能发生在多个地方，最终根据成本选择最佳计划。

一个调用约定是一个实现了[`interface Convention`](https://calcite.apache.org/javadocAggregate/org/apache/calcite/plan/Convention.html) 的类,或者一个附加的接口 (例如 [`interface CassandraRel`](https://calcite.apache.org/javadocAggregate/org/apache/calcite/adapter/cassandra/CassandraRel.html)), 或是一些由 [`class RelNode`](https://calcite.apache.org/javadocAggregate/org/apache/calcite/rel/RelNode.html)  的子类组成的集合，这些子类实现了核心关系算子的接口、 ([`Project`](https://calcite.apache.org/javadocAggregate/org/apache/calcite/rel/core/Project.html), [`Filter`](https://calcite.apache.org/javadocAggregate/org/apache/calcite/rel/core/Filter.html), [`Aggregate`](https://calcite.apache.org/javadocAggregate/org/apache/calcite/rel/core/Aggregate.html) 等).

##### Built-in SQL implementation

如果一个adapter没有实现所有的核心关系算子，那么Calcite怎么实现SQL？

答案是有一个特殊的内置调用约定：[`EnumerableConvention`](https://calcite.apache.org/javadocAggregate/org/apache/calcite/adapter/EnumerableConvention.html) 。枚举约定（enumerable convention）的关系表达式以内置的方式（“built-ins”）实现：Calcite 生成Java 代码，编译，并且将其在自己的JVM上执行。但是枚举约定比分布式引擎（例如，运行在面向列的数据文件上）效率要低，但是它可以实现所有的核心关系算子和所有的内置SQL函数和算子。如果一个数据源没有实现某个关系算子，那么枚举约定会是退路。

##### Statistics and cost

Calcite有一个元数据系统，允许你定义自己的代价函数以及关系算子的统计数据，统称为元数据（*metadata*）。每种元数据有一个（通常）有一个方法的接口。例如，选择（selectivity ）被定义为接口 [`interface RelMdSelectivity`](https://calcite.apache.org/javadocAggregate/org/apache/calcite/rel/metadata/RelMdSelectivity.html) 和对应的方法[`getSelectivity(RelNode rel, RexNode predicate)`](https://calcite.apache.org/javadocAggregate/org/apache/calcite/rel/metadata/RelMetadataQuery.html#getSelectivity-org.apache.calcite.rel.RelNode-org.apache.calcite.rex.RexNode-) 。

有许多种类型的内置元数据，包括：[collation](https://calcite.apache.org/javadocAggregate/org/apache/calcite/rel/metadata/RelMdCollation.html), [column origins](https://calcite.apache.org/javadocAggregate/org/apache/calcite/rel/metadata/RelMdColumnOrigins.html), [column uniqueness](https://calcite.apache.org/javadocAggregate/org/apache/calcite/rel/metadata/RelMdColumnUniqueness.html), [distinct row count](https://calcite.apache.org/javadocAggregate/org/apache/calcite/rel/metadata/RelMdDistinctRowCount.html), [distribution](https://calcite.apache.org/javadocAggregate/org/apache/calcite/rel/metadata/RelMdDistribution.html), [explain visibility](https://calcite.apache.org/javadocAggregate/org/apache/calcite/rel/metadata/RelMdExplainVisibility.html), [expression lineage](https://calcite.apache.org/javadocAggregate/org/apache/calcite/rel/metadata/RelMdExpressionLineage.html), [max row count](https://calcite.apache.org/javadocAggregate/org/apache/calcite/rel/metadata/RelMdMaxRowCount.html), [node types](https://calcite.apache.org/javadocAggregate/org/apache/calcite/rel/metadata/RelMdNodeTypes.html), [parallelism](https://calcite.apache.org/javadocAggregate/org/apache/calcite/rel/metadata/RelMdParallelism.html), [percentage original rows](https://calcite.apache.org/javadocAggregate/org/apache/calcite/rel/metadata/RelMdPercentageOriginalRows.html), [population size](https://calcite.apache.org/javadocAggregate/org/apache/calcite/rel/metadata/RelMdPopulationSize.html), [predicates](https://calcite.apache.org/javadocAggregate/org/apache/calcite/rel/metadata/RelMdPredicates.html), [row count](https://calcite.apache.org/javadocAggregate/org/apache/calcite/rel/metadata/RelMdRowCount.html), [selectivity](https://calcite.apache.org/javadocAggregate/org/apache/calcite/rel/metadata/RelMdSelectivity.html), [size](https://calcite.apache.org/javadocAggregate/org/apache/calcite/rel/metadata/RelMdSize.html), [table references](https://calcite.apache.org/javadocAggregate/org/apache/calcite/rel/metadata/RelMdTableReferences.html), and [unique keys](https://calcite.apache.org/javadocAggregate/org/apache/calcite/rel/metadata/RelMdUniqueKeys.html); 你也可以定义你自己的。

你可以提供一个*metadata provider* 来计算特定`RelNode`的子类的对应的元数据。metadata providers可以处理内置的和扩展的元数据类型，以及内置和扩展的`RelNode`类型。在查询准备阶段，Calcite 会结合所有的可以应用的元数据提供器（metadata providers），维持这些元数据构成的一个缓存，以便只计算给定的元数据片段(例如特定`Filter`操作符中条件`x > 10`的选择性)。

