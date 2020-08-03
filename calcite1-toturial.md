## Calcite学习笔记1-Tutorial

### CSV 示例

#### calcite的重要概念

- 用户自定义`SchemaFactory `和`Schema`接口；
- 使用model JSON文件声明schemas；
- 使用model JSON文件声明views;
- 用户自定义`Table`接口；
- 决定table的记录类型（record type);
- `Table`接口的简单实现，使用`ScannableTable`接口，直接遍历所有行；
- 更加高级的实现：`FilterableTable`，可以通过简单的prefictates过滤一些行；
- 再高级一些的Table实现：`TranslatableTable`，可以用过plans去翻译成关系操作符（**relational operators**）

#### 下载和编译

需要java （8,9,10) 以及Git

```shell
$ git clone https://github.com/apache/calcite.git
$ cd calcite/example/csv
$ ./sqlline
```

#### 简单的查询

通过sqlline连接到calcite

```shell
$ ./sqlline
sqlline> !connect jdbc:calcite:model=src/test/resources/model.json admin admin
```

在这里指定了model.json，用户名admin，密码admin

如果是windows，使用sqlline.bat（但是这里我没有成功）

使用一个metadate query（元数据查询）

```shell
sqlline> !tables
+------------+--------------+-------------+---------------+----------+------+
| TABLE_CAT  | TABLE_SCHEM  | TABLE_NAME  |  TABLE_TYPE   | REMARKS  | TYPE |
+------------+--------------+-------------+---------------+----------+------+
| null       | SALES        | DEPTS       | TABLE         | null     | null |
| null       | SALES        | EMPS        | TABLE         | null     | null |
| null       | SALES        | HOBBIES     | TABLE         | null     | null |
| null       | metadata     | COLUMNS     | SYSTEM_TABLE  | null     | null |
| null       | metadata     | TABLES      | SYSTEM_TABLE  | null     | null |
+------------+--------------+-------------+---------------+----------+------+
```

除了`!tables`还可以使用`!columns`和`!describe`

在src/test/resources/sales下有数据文件，两个csv，一个csv.gz压缩文件

```shell
$ ls
DEPTS.csv  EMPS.csv.gz  SDEPTS.csv
```

使用select

```sql
sqlline> SELECT * FROM emps;
+--------+--------+---------+---------+----------------+--------+-------+---+
| EMPNO  |  NAME  | DEPTNO  | GENDER  |      CITY      | EMPID  |  AGE  | S |
+--------+--------+---------+---------+----------------+--------+-------+---+
| 100    | Fred   | 10      |         |                | 30     | 25    | t |
| 110    | Eric   | 20      | M       | San Francisco  | 3      | 80    | n |
| 110    | John   | 40      | M       | Vancouver      | 2      | null  | f |
| 120    | Wilma  | 20      | F       |                | 1      | 5     | n |
| 130    | Alice  | 40      | F       | Vancouver      | 2      | null  | f |
+--------+--------+---------+---------+----------------+--------+-------+---+
```

使用JOIN和GROUP BY：

```sql
sqlline> SELECT d.name, COUNT(*)
. . . .> FROM emps AS e JOIN depts AS d ON e.deptno = d.deptno
. . . .> GROUP BY d.name;
+------------+---------+
|    NAME    | EXPR$1  |
+------------+---------+
| Sales      | 1       |
| Marketing  | 2       |
+------------+---------+
```

还可以直接使用算子（operator），这是一个简单的方式来测试表达式和SQL内嵌的functions

```sql
sqlline> VALUES CHAR_LENGTH('Hello, ' || 'world!');
+---------+
| EXPR$0  |
+---------+
| 13      |
+---------+
```

#### schema 探究

事实上，Calcite并不知道CSV文件的存在，作为没有存储层的数据库，Calcite不知道任何数据格式。

流程是：

- 首先通过model文件定义一个用于产生schema的schema factory
- schema factory 创建一个 schema
- schema 创建一系列表格，schema 知道怎么去从CSV文件中获取数据
- Calcite解析query，并规划好（plan）之后用在这些表格上，当query被执行的时候，Calcite就从tables拿到了数据

在JDBC连接中，使用JSON文件作为model，

```json
 * A JSON model of a simple Calcite schema.
{
  "version": "1.0",
  "defaultSchema": "SALES",
  "schemas": [
    {
      "name": "SALES",
      "type": "custom",
      "factory": "org.apache.calcite.adapter.csv.CsvSchemaFactory",
      "operand": {
        "directory": "sales"
      }
    }
  ]
}
```

这个model定义了一个名叫SALES的schema，指定了工厂类`org.apache.calcite.adapter.csv.CsvSchemaFactory`来创建schema，这个工厂类实现了`schemaFactory`接口，它的`creta`方法用来创建schema实例，并且从model文件中传递`directory`参数。

```java
public Schema create(SchemaPlus parentSchema, String name,
    Map<String, Object> operand) {
  String directory = (String) operand.get("directory");
  String flavorName = (String) operand.get("flavor");
  CsvTable.Flavor flavor;
  if (flavorName == null) {
    flavor = CsvTable.Flavor.SCANNABLE;
  } else {
    flavor = CsvTable.Flavor.valueOf(flavorName.toUpperCase());
  }
  return new CsvSchema(
      new File(directory),
      flavor);
}
```

至此，创建了一个名为“SALES”的`org.apache.calcite.adapter.csv.CsvSchema`实例，这个实例实现了`schema`接口

> A schema’s job is to produce a list of tables. (It can also list sub-schemas and table-functions, but these are advanced features and calcite-example-csv does not support them.) The tables implement Calcite’s [Table](https://calcite.apache.org/javadocAggregate/org/apache/calcite/schema/Table.html) interface. `CsvSchema` produces tables that are instances of [CsvTable](https://github.com/apache/calcite/blob/master/example/csv/src/main/java/org/apache/calcite/adapter/csv/CsvTable.java) and its sub-classes.

schema的作用是创建一系列的tables（当然也可以是子schema，以及table-functions，但是目前的csv例子是不包含这个的）。这些tables实现了Calcite的`Table`接口，`Csvschema`创建了一些`CsvTable`的实例以及它的子类的实例。

`Csvschema`继承了`Abstractschema`基类，重写了`getTableMap()`方法：

```java
protected Map<String, Table> getTableMap() {
  // Look for files in the directory ending in ".csv", ".csv.gz", ".json",
  // ".json.gz".
  // 寻找以".csv"、".csv.gz"、".json"、".json.gz"后缀的的文件
  File[] files = directoryFile.listFiles(
      new FilenameFilter() {
        public boolean accept(File dir, String name) {
          final String nameSansGz = trim(name, ".gz");
          return nameSansGz.endsWith(".csv")
              || nameSansGz.endsWith(".json");
        }
      });
  if (files == null) {
    System.out.println("directory " + directoryFile + " not found");
    files = new File[0];
  }
  // Build a map from table name to table; each file becomes a table.
  // 创建从table name到table的映射，每一个文件成为一个table
  final ImmutableMap.Builder<String, Table> builder = ImmutableMap.builder();
  for (File file : files) {
    String tableName = trim(file.getName(), ".gz");
    final String tableNameSansJson = trimOrNull(tableName, ".json");
    if (tableNameSansJson != null) {
      JsonTable table = new JsonTable(file);
      builder.put(tableNameSansJson, table);
      continue;
    }
    tableName = trim(tableName, ".csv");
    final Table table = createTable(file);
    builder.put(tableName, table);
  }
  return builder.build();
}

/** Creates different sub-type of table based on the "flavor" attribute. */
// 根据flavor属性（调料的意思）来创建table的子类型
private Table createTable(File file) {
  switch (flavor) {
  case TRANSLATABLE:
    return new CsvTranslatableTable(file, null);
  case SCANNABLE:
    return new CsvScannableTable(file, null);
  case FILTERABLE:
    return new CsvFilterableTable(file, null);
  default:
    throw new AssertionError("Unknown flavor " + flavor);
  }
}
```

通过EMPS.csv和DEPTS.csv创建了EMPS和DEPTS的表格。

#### Tables and views in schemas

不需要在model文件中定义任何tables，schema主动生成了tables

除了自动创建的tables以为，可以通过schema的`tables`属性自定义table

> A view looks like a table when you are writing a query, but it doesn’t store data. It derives its result by executing a query. The view is expanded while the query is being planned, so the query planner can often perform optimizations like removing expressions from the SELECT clause that are not used in the final result.

当在查询数据的时候，一个view就像是一个table，但是view不存储任何数据。view的数据来源于执行的query。当query被plan的时候，view被扩展，所以通常情况下，一个query planner通过将没有实际执行的select的字句去除来优化性能。

下面是如何通过schema来定义view

```json
{
  version: '1.0',
  defaultSchema: 'SALES',
  schemas: [
    {
      name: 'SALES',
      type: 'custom',
      factory: 'org.apache.calcite.adapter.csv.CsvSchemaFactory',
      operand: {
        directory: 'sales'
      },
      tables: [
        {
          name: 'FEMALE_EMPS',
          type: 'view',
          sql: 'SELECT * FROM emps WHERE gender = \'F\''
        }
      ]
    }
  ]
}
```

`type : 'view'`说明了`FEMALE_EMPS`是一个view，不同于常规的table和自定义的table。注意'F'的反斜杠，这在JSON是常规操作。

JSON对长String的支持不友好，Calcite可以用lines list来替代单行String。

```json
{
  name: 'FEMALE_EMPS',
  type: 'view',
  sql: [
    'SELECT * FROM emps',
    'WHERE gender = \'F\''
  ]
}
```

在定义了view之后，可以像查询table一样使用view

```sql
sqlline> SELECT e.name, d.name FROM female_emps AS e JOIN depts AS d on e.deptno = d.deptno;
+--------+------------+
|  NAME  |    NAME    |
+--------+------------+
| Wilma  | Marketing  |
+--------+------------+
```

#### Custom tables

自定义的tables，通过用户自定义的代码来创建，不需要在custom schema中

下面是`model-with-custom-table.json`:

```json
{
  version: '1.0',
  defaultSchema: 'CUSTOM_TABLE',
  schemas: [
    {
      name: 'CUSTOM_TABLE',
      tables: [
        {
          name: 'EMPS',
          type: 'custom',
          factory: 'org.apache.calcite.adapter.csv.CsvTableFactory',
          operand: {
            file: 'sales/EMPS.csv.gz',
            flavor: "scannable"
          }
        }
      ]
    }
  ]
}
```

可以使用常规手段查询

```sql
sqlline> !connect jdbc:calcite:model=src/test/resources/model-with-custom-table.json admin admin
sqlline> SELECT empno, name FROM custom_table.emps;
+--------+--------+
| EMPNO  |  NAME  |
+--------+--------+
| 100    | Fred   |
| 110    | Eric   |
| 110    | John   |
| 120    | Wilma  |
| 130    | Alice  |
+--------+--------+
```

model中指明了table的工厂类`org.apache.calcite.adapter.csv.CsvTableFactory`，这个工厂类实现了`TableFactory`接口，`create`方法创建了`CsvScannableTable`实例，在model文件中传递了`file`参数

```java
public CsvTable create(SchemaPlus schema, String name,
    Map<String, Object> map, RelDataType rowType) {
  String fileName = (String) map.get("file");
  final File file = new File(fileName);
  final RelProtoDataType protoRowType =
      rowType != null ? RelDataTypeImpl.proto(rowType) : null;
  return new CsvScannableTable(file, protoRowType);
}
```

自定义table是自定义schema的另一种选择。两种方式最终都创建了`Table`接口的实现，但是自定了table，没有去实现元数据挖掘（**metadata discovery**），也就是说，`CsvTableFactory`创建了`CsvScannableTable`,`Csvschema`也创建了相同的`Table`实现，但是自定义table不需要在文件系统中去扫描.csv文件（这就是所谓的metadata discovery)。

自定义table需要作者做更多的事，比如为不同的table指定不同的文件等，但是也给了作者更多的控制权限，比如为不同的table定义不同的参数等。

#### Comments in models

可以使用`/* ... */`和`//`来注释，比如

```json
{
  version: '1.0',
  /* Multi-line
     comment. */
  defaultSchema: 'CUSTOM_TABLE',
  // Single-line comment.
  schemas: [
    ..
  ]
}
```

并不是标准的JSON，但是是无害的扩展。

#### Optimizing queries using planner rules

到目前为止我们进行的操作看起来都没有什么问题，这是因为数据量实在是太少了。如果一个自动了的表格里面有一百行、一百万列，你就不会希望系统会检索每一条数据了。需要使用Calcite去和Adapter“协商”出一个处理数据的更有效的方式。

> This negotiation is a simple form of query optimization. Calcite supports query optimization by adding *planner rules*. Planner rules operate by looking for patterns in the query parse tree (for instance a project on top of a certain kind of table), and replacing the matched nodes in the tree by a new set of nodes which implement the optimization.

这个”协商“就是query optimization（查询优化）的一种方式。Calcite 支持通过添加计划规则（**planner rules **）来实现优化。Planner rules 寻找在查询解析树（可以理解为基于table的一个project）当中的模式，并且将与模式匹配的节点利用一系列新的节点集来替代，实现优化。

Planner rules是可以拓展的，比如schemas和tables。所以，如果你通过sql查询数据，最好先定义tables和shemas，然后定义一些规则（rules）去实现高效查询。

看个例子，相同的查询应用在**相似**的schema上：

```shell
sqlline> !connect jdbc:calcite:model=src/test/resources/model.json admin admin
sqlline> explain plan for select name from emps;
+-----------------------------------------------------+
| PLAN                                                |
+-----------------------------------------------------+
| EnumerableCalcRel(expr#0..9=[{inputs}], NAME=[$t1]) |
|   EnumerableTableScan(table=[[SALES, EMPS]])        |
+-----------------------------------------------------+
sqlline> !connect jdbc:calcite:model=src/test/resources/smart.json admin admin
sqlline> explain plan for select name from emps;
+-----------------------------------------------------+
| PLAN                                                |
+-----------------------------------------------------+
| EnumerableCalcRel(expr#0..9=[{inputs}], NAME=[$t1]) |
|   CsvTableScan(table=[[SALES, EMPS]])               |
+-----------------------------------------------------+
```

两个schema的差异在于`smart.json`的里面多一条：

```yaml
flavor: "translatable"
```

这个属性让`CsvSchema`在被创建的时候考虑了`flavor = TRANLATABLE`，所以呢，`createTable`方法就会穿件`CsvTranslatableTable`而不是`CsvScannableTable`.

`CsvTranslatableTable`实现了`TranslatableTable.toRel()`方法创建了`CsvTableScan`，Table Scans是查询算子树（query operator tree）上的叶子节点。 最常见的实现是`EnumerableTableScan`，但是这里我们创建了一个更有特色的子类来实现规则的触发（**file rules**）

下面是全部的规则：

```java
public class CsvProjectTableScanRule extends RelOptRule {
  public static final CsvProjectTableScanRule INSTANCE =
      new CsvProjectTableScanRule();

  private CsvProjectTableScanRule() {
    super(
        operand(Project.class,
            operand(CsvTableScan.class, none())),
        "CsvProjectTableScanRule");
  }

  @Override
  public void onMatch(RelOptRuleCall call) {
    final Project project = call.rel(0);
    final CsvTableScan scan = call.rel(1);
    int[] fields = getProjectFields(project.getProjects());
    if (fields == null) {
      // 投影包含比字段引用更加复杂的表达式
      // Project contains expressions more complex than just field references.
      return;
    }
    call.transformTo(
        new CsvTableScan(
            scan.getCluster(),
            scan.getTable(),
            scan.csvTable,
            fields));
  }

  private int[] getProjectFields(List<RexNode> exps) {
    final int[] fields = new int[exps.size()];
    for (int i = 0; i < exps.size(); i++) {
      final RexNode exp = exps.get(i);
      if (exp instanceof RexInputRef) {
        fields[i] = ((RexInputRef) exp).getIndex();
      } else {
        return null; // not a simple projection
      }
    }
    return fields;
  }
}
```

这个构造方法声明了会触发规则的关系表达式的模式（pattern）。

`onMatch`方法创建了新的关系表达式并且调用了`RelOptRuleCall.transfromTo()`去表明规则已经成功触发。

#### The query optimization process

Calcite 的query planner 是很厉害的，他的厉害之处在于减少了你的工作，以及planner rules制定者的工作。

首先，Calcite并不会按照可预测的顺序触发规则。就好像下象棋一样，不同的走法有很多的组合。如果rule A 和入了rule B 同时匹配给定的query operator tree（查询算子树）,那么Caicite会将两条规则都触发。

第二，Calcite利用代价（cost）来选择计划（plans)，但是代价模型并不会组织某条规则被触发（哪怕看起来确实代价更大）。

很多优化器有线性的优化模式。当在A和B规则当中做选择的时候，这样的优化器必须立刻做出抉择。他们可能会这样选择：将A和B对着同时应用到整棵树上，然后基于代价模型选择代价更小的结果。

Calcite并不会采取这样的折中策略。这样会很容易结合不同的规则集。如果你希望将用于识别物化视图（materialized views）的规则与用于从CSV和JDBC源系统读取的规则组合在一起，那么只需将所有规则集交给Calcite，并告诉它进行处理。

Calcite确实是使用了代价模型。这个代价模型决定了最终会被使用了计划（plan），并且可能会修剪掉一些search tree以防止search space（搜索空间）的爆炸，但是Calcite的代价模型，绝不会强制你在A和B之间做选择。这很重要，因为这避免了陷入局部最优的陷阱。

当然，这个代价模型是pluggable（可插拔的），代价模型所基于的table和query算子的statistics（数据统计）也是pluggable。

#### JDBC adapter

前面将的都是CSV的例子

JDBC adapter将JDBC 数据源的schema映射到Calcite schema上。

例如，这个schema从一个MySQL“foormart”数据库读数据：

```json
{
  version: '1.0',
  defaultSchema: 'FOODMART',
  schemas: [
    {
      name: 'FOODMART',
      type: 'custom',
      factory: 'org.apache.calcite.adapter.jdbc.JdbcSchema$Factory',
      operand: {
        jdbcDriver: 'com.mysql.jdbc.Driver',
        jdbcUrl: 'jdbc:mysql://localhost/foodmart',
        jdbcUser: 'foodmart',
        jdbcPassword: 'foodmart'
      }
    }
  ]
}
```

> **Current limitations**: The JDBC adapter currently only pushes down table scan operations; all other processing (filtering, joins, aggregations and so forth) occurs within Calcite. Our goal is to push down as much processing as possible to the source system, translating syntax, data types and built-in functions as we go. If a Calcite query is based on tables from a single JDBC database, in principle the whole query should go to that database. If tables are from multiple JDBC sources, or a mixture of JDBC and non-JDBC, Calcite will use the most efficient distributed query approach that it can.

目前的限制（limitations）：

JDBC适配器只下推scan，其他的processing（filtering, joins, aggregations and so forth）在Calcite内部进行。***我们的目标是尽可能下推多的算子到数据源，通过翻译语法、数据类型以及内置函数***。如果查询只是基于单个来自JDBC的table，那么所有的查询直接被下推到该数据源，没有什么可说的。但是如果数据来自于不同的JDBC数据源，或是JDBC和非JDBC混合的数据源，那么Calcite将会使用它所能做到的最有效的分配查询手段。

#### The cloning JDBC adapter

克隆JDBC(cloning JDBC)适配器创建了混合的数据库。数据来自于JDBC，但是在每一张表被访问时，这张表的数据就会被加载进内存。Calcite会根据内存里面的tables来评估查询，这实际上是对数据库的缓存。

例如，下面的model将读取MySQL“footmart”的tables。

```json
{
  version: '1.0',
  defaultSchema: 'FOODMART_CLONE',
  schemas: [
    {
      name: 'FOODMART_CLONE',
      type: 'custom',
      factory: 'org.apache.calcite.adapter.clone.CloneSchema$Factory',
      operand: {
        jdbcDriver: 'com.mysql.jdbc.Driver',
        jdbcUrl: 'jdbc:mysql://localhost/foodmart',
        jdbcUser: 'foodmart',
        jdbcPassword: 'foodmart'
      }
    }
  ]
}
```

还有一种方式来创建一种克隆的schema，从已有的schema上克隆。可以指定`source`属性来说引用一个之前定义过的schema model，像这样：

```json
{
  version: '1.0',
  defaultSchema: 'FOODMART_CLONE',
  schemas: [
    {
      name: 'FOODMART',
      type: 'custom',
      factory: 'org.apache.calcite.adapter.jdbc.JdbcSchema$Factory',
      operand: {
        jdbcDriver: 'com.mysql.jdbc.Driver',
        jdbcUrl: 'jdbc:mysql://localhost/foodmart',
        jdbcUser: 'foodmart',
        jdbcPassword: 'foodmart'
      }
    },
    {
      name: 'FOODMART_CLONE',
      type: 'custom',
      factory: 'org.apache.calcite.adapter.clone.CloneSchema$Factory',
      operand: {
        source: 'FOODMART'
      }
    }
  ]
}
```

你可以用这种方式clone任何一种shemal，而不只是JDBC schema。

克隆适配器不是我们最终、最好的部分。我们计划开发一种更加精妙的缓存策略，以及对内存里面的table更加高效和完整的实现，但是现在而言，克隆适配器展示了可能性以及我们最初的尝试。