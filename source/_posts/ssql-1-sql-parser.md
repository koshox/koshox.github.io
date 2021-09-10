---
title: SSQL(Simle-SQL)框架 1.SQL语法解析和自定义SPI
date: 2021-09-08 22:21:17
tags:
- SQL解析
- SPI
- 编译原理
categories:
- 数据库
---

SSQL(Simle-SQL)是SQL-Like语法的一个数据查询、操作框架，使用Java实现，主要目标是屏蔽不同数据源的差异，能够使用同一套SQL语法操作不同的数据源。这篇文章的内容是ssql-core模块的实现，主要是SSQL语法解析和自定义SPI模块。

完整代码放在我的Github上：https://github.com/koshox/ssql

<!-- More -->

## SQL语法解析

SQL的语法解析和C、Java这些语言的解析没有本质区别，但是相对比较简单。都会经过词法分析、语法分析、语义分析、中间代码生成、优化、目标代码生成这几个阶段。

SQL语法解析一般有这么几个方式：

1. 直接使用已有的库，比如Druid，但是不好定制和扩展。
2. 手写递归下降解析器、解析器组合子。
3. 使用ANTLR，JavaCC这样的工具来帮助我们生成语法解析代码。

出于简单和可扩展的两个角度，我们使用[ANTLR](https://www.antlr.org)来做词法和语法分析，Hibernate、Spark、ShardingSphere等项目也都使用了ANTLR。只需要编写EBNF的语法文件就可以了，ANTLR会基于语法文件生成LL(k)的语法解析器。

### 词法分析

词法分析就是做分词，把输入转换为Token流，比如select * from t_user，会被转换为select，*，from，t_user。

ANTLR的lexer很简单，基本就是简单的正则表达式语法。为了控制篇幅用......省略一些重复的内容，如果感兴趣可以看[github上的完整实现](https://github.com/koshox/ssql)。

```java
lexer grammar SsqlLexer;

// 生成代码包路径
@header {package com.kosho.ssql.core.dsl.parser;}

// 跳过空格和注释
SPACE: [ \t\r\n]+ -> channel(HIDDEN);
COMMENT: '/*' .*? '*/' -> channel(HIDDEN);
LINE_COMMENT: (
                  ('-- ' | '#') ~[\r\n]* ('\r'? '\n' | EOF)
                  | '--' ('\r'? '\n' | EOF)
              ) -> channel(HIDDEN);

// 关键字
SELECT: S E L E C T;
FROM: F R O M;
WHERE: W H E R E;
ORDER: O R D E R;
GROUP: G R O U P;
HAVING: H A V I N G;

BY: B Y;
AS: A S;
AND: A N D;
OR: O R;
......

// 操作符
STAR: '*';
DIV: '/';
MOD: '%';
PLUS: '+';
EQ: '=';
GT: '>';
LT: '<';
.....

// 字符串
SQUOTE_STRING: '\'' ('\\'. | '\'\'' | ~('\'' | '\\'))* '\'';
DQUOTE_STRING: '"' ('\\'. | '""' | ~('"' | '\\'))* '"';
RQUOTE_STRING: '`' ('\\'. | '``' | ~('`' | '\\'))* '`';

// 数字
INTEGER: DEG_DIGIT+;
DECIMAL: ('0' | [1-9] DEG_DIGIT*) (DOT DEG_DIGIT+)? ([eE] [+\-]? DEG_DIGIT+)?;

// 标识符
ID: [a-zA-Z0-9_]*? [a-zA-Z_]+? [a-zA-Z0-9_]*;

// FRAGMENT
fragment A: [aA];
fragment B: [bB];
fragment C: [cC];
......

fragment DEG_DIGIT: [0-9];
```

然后使用ANTLR的jar或者idea插件就可以生成词法解析器Java文件。

### 语法分析

SSQL目前主要是支持查询语句，主要语法树如下：

![SSQL AST](http://media.kosho.tech/blog/ssql/ssql-ast.png)

语法分析器可以引用上面的Lexer，并针对select，from等子句依次编写语法，ANTLR会对每一个分支都生成对应的语法树和监听/遍历方法。使用#可以进行标记，以生成更精确的方法。

```java
parser grammar SsqlParser;

@header {package com.kosho.ssql.core.dsl.parser;}

options {
    tokenVocab=SsqlLexer;
}

// ROOT节点
ssql: selectStatement SEMI? EOF;

// SELECT语句
selectStatement
    : selectClause fromClause whereClause? groupByClause? orderByClause? limitClause?

// SELECT
selectStatement
    : selectClause fromClause whereClause? groupByClause? orderByClause? limitClause? #simpleSelectStatement
    | LBKT selectStatement RBKT                                                       #bracketedSelectStatement;

selectClause: SELECT DISTINCT? selectColumns;

selectColumns
    : STAR                               #selectAll
    | selectColumn (COMMA selectColumn)* #selectSpecialColumns
    ;

// FROM
fromClause: FROM tableRefs;
tableRefs: tableRef (COMMA tableRef)?;
tableRef: table=identifierValue (AS? alias=identifierValue)?;

// WHERE
whereClause: WHERE expression;

expression
    : left=expression operator=AND right=expression #logicExpr
    | left=expression operator=OR right=expression  #logicExpr
    | LBKT expression RBKT                          #bracketedExpr
    | predicateExpr                                 #predicatedExpr
    ;

predicateExpr
    : fieldAtom IS NOT? NULL                 #isNullPredicate
    | fieldAtom comparisonOperator valueAtom #binaryComparisonPredicate
    | fieldAtom IN (listValue | varValue)    #inPredicate
    ;

// GROUP BY
groupByClause: GROUP BY valueAtom (COMMA valueAtom)* havingClause?;

......

identifierValue: ID | DQUOTE_STRING;
constant: NULL | stringLiteral | numberLiteral | booleanLiteral;
stringLiteral: SQUOTE_STRING;
numberLiteral: MINUS? literal=INTEGER | MINUS? literal=DECIMAL;
booleanLiteral: TRUE | FALSE;
comparisonOperator: EQ | GT | GTE | LT | LTE | LT GT | NOT_EQ | IS | IS NOT;
```

到这一步生成的语法解析器已经能够把SSQL语句解析成语法树了，可以使用idea的ANTLR插件调试语法规则。

### 中间代码生成

上一步ANTLR生成的语法树不够直观，想生成更加简洁的中间表示，即SSQL对象。后续的其他数据源语法转换、分库分表，也会更加方便。

ANTLR提供了Listener和Visitor两种方式遍历生成的语法树，Listener更简单，Visitor更适合复杂一定的树形结构，这里使用Visitor模式进行遍历和中间代码生成，直接继承上一步生成的SsqlParserBaseVisitor类即可。

```java
public class SsqlAstVisitor extends SsqlParserBaseVisitor<Object> {
    public static final SsqlAstVisitor INSTANCE = new SsqlAstVisitor();

    // 遍历root节点
    @Override
    public Object visitSsql(SsqlParser.SsqlContext ctx) {
        return visit(ctx.selectStatement());
    }

    // 遍历select语句
    @Override
    public Object visitSimpleSelectStatement(SsqlParser.SimpleSelectStatementContext ctx) {
        Ssql ssql = new Ssql();
        if (ctx.selectClause() != null) {
            ssql.setSelect((Select) visitSelectClause(ctx.selectClause()));
        }

        if (ctx.fromClause() != null) {
            ssql.setFrom((From) visitFromClause(ctx.fromClause()));
        }

        if (ctx.whereClause() != null) {
            ssql.setWhere((Where) visitWhereClause(ctx.whereClause()));
        }

        if (ctx.groupByClause() != null) {
            ssql.setGroupBy((GroupBy) visitGroupByClause(ctx.groupByClause()));
        }

        if (ctx.orderByClause() != null) {
            ssql.setOrderBy((OrderBy) visitOrderByClause(ctx.orderByClause()));
        }

        if (ctx.limitClause() != null) {
            ssql.setLimit((Limit) visitLimitClause(ctx.limitClause()));
        }

        return ssql;
    }

    // 遍历select子句
    @Override
    public Object visitSelectClause(SsqlParser.SelectClauseContext ctx) {
        Select select = new Select();
        if (ctx.DISTINCT() != null) {
            select.setDistinct(true);
        }

        Object selects = visit(ctx.selectColumns());
        if (selects instanceof String) {
            select.setStar(true);
            return select;
        }

        select.setSelectItems((List<SelectItem>) selects);
        return select;
    }

    @Override
    public Object visitSelectAll(SsqlParser.SelectAllContext ctx) {
        return "*";
    }

    @Override
    public Object visitSelectSpecialColumns(SsqlParser.SelectSpecialColumnsContext ctx) {
        List<SelectItem> selectItems = new ArrayList<>();
        for (SsqlParser.SelectColumnContext column : ctx.selectColumn()) {
            SelectItem selectItem = (SelectItem) visitSelectColumn(column);
            selectItems.add(selectItem);
        }

        return selectItems;
    }

    @Override
    public Object visitSelectColumn(SsqlParser.SelectColumnContext ctx) {
        SelectItem selectItem = new SelectItem();
        selectItem.setField((Value) visit(ctx.column));
        if (ctx.alias != null) {
            String alias = visit(ctx.alias).toString();
            selectItem.setAlias(alias);
        }

        return selectItem;
    }

    // 遍历from子句
    @Override
    public Object visitFromClause(SsqlParser.FromClauseContext ctx) {
        From from = new From();
        from.setTables((List<Table>) visitTableRefs(ctx.tableRefs()));
        return from;
    }

    @Override
    public Object visitTableRefs(SsqlParser.TableRefsContext ctx) {
        ArrayList<Table> tables = new ArrayList<>();
        for (SsqlParser.TableRefContext tableRefContext : ctx.tableRef()) {
            tables.add((Table) visitTableRef(tableRefContext));
        }

        return tables;
    }

    @Override
    public Object visitTableRef(SsqlParser.TableRefContext ctx) {
        Table table = new Table();
        table.setName(visit(ctx.table).toString());

        if (ctx.alias != null) {
            table.setAlias(visit(ctx.alias).toString());
        }

        return table;
    }

	......
}
```

随后我们给Ssql加上编译方法

```java
public static Ssql compile(String ssqlStr) {
    // 词法解析器
    SsqlLexer lexer = new SsqlLexer(CharStreams.fromString(ssqlStr));

    // 语法解析器
    SsqlParser parser = new SsqlParser(new CommonTokenStream(lexer));

    // 异常监听器
    parser.addErrorListener(SsqlErrorListener.INSTANCE);

    // 语法树生成
    SsqlParser.SsqlContext context = parser.ssql();

    // 语法树遍历
    return (Ssql) SsqlAstVisitor.INSTANCE.visit(context);
}
```

至此，已经支持将表达式编译为Ssql对象了

```java
// 表达式编译为SSQL语法树
Ssql ssql = Ssql.compile("select * from t_user order by name");// select * from t_user order by name

// 参数化编译
Map<String, Object> params = new HashMap<>();
params.put("name", "kosho");
params.put("age", 25);
Ssql ssql = Ssql.compile("select * from t_user where name = $name and age = $age"); // select * from t_user where name = 'kosho' and age = 25
```

## 自定义SPI实现

Java的SPI(Service Provider Interface)机制大家应该都不陌生，它是Java提供的一种服务扩展机制。像JDBC，SLF4J，Dubbo，Seata都用到了SPI的机制来加载组件。

Java原生的SPI有一些缺点：

1. 默认实例化所有的SPI实现，无法按需加载，导致资源浪费
2. 没有优先级，按名称加载这些功能，不够易用

所以很多框架都实现了自己的SPI机制，Dubbo的SPI就做的功能很强大（复杂），还基于javassist做了支持URL自适应的能力。

SSQL也使用SPI扩展组件（后续分表算法，语法树改写都会用到），并开发了一套自定义SPI实现。大概100多行代码就能实现一个支持优先级、名称、单/多例的一个SPI模块。

首先我们先定义SPI注解，作为辅助扩展。

```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
@Documented
public @interface Spi {
    // SPI名称
    String value() default "";

    // 是否单例
    boolean isSingleton() default true;

    // 是否默认
    boolean isDefault() default false;

    // 优先级
    int order() default 0;

    // 最高优先级
    int ORDER_HIGHEST = Integer.MIN_VALUE;

    // 最低优先级
    int ORDER_LOWEST = Integer.MAX_VALUE;
}
```

然后编写SPI加载器，主要就是加载META-INF/services下的SPI配置

```java
public class SpiLoader<S> {
    private static final String SPI_CFG_PREFIX = "META-INF/services/";

    private static final Map<String, SpiLoader> SPI_LOADERS = new ConcurrentHashMap<>();

    private final List<Class<? extends S>> classList = Collections.synchronizedList(new ArrayList<>());

    private final ConcurrentHashMap<String, Class<? extends S>> spiName2Class = new ConcurrentHashMap<>();

    private final ConcurrentHashMap<String, S> className2Singleton = new ConcurrentHashMap<>();

    private final AtomicBoolean loaded = new AtomicBoolean(false);

    private Class<? extends S> defaultClass = null;

    private Class<S> service;

    private SpiLoader(Class<S> service) {
        this.service = service;
    }

    /**
     * 创建SPI加载器
     *
     * @param service SPI接口
     * @param <T>     SPI泛型
     * @return SPI加载器
     */
    public static <T> SpiLoader<T> of(Class<T> service) {
        Validate.notNull(service, "Spi class cannot be null");
        Validate.isTrue(service.isInterface(), "Spi class[" + service.getName() + "] must be interface");

        return SPI_LOADERS.computeIfAbsent(service.getName(), key -> new SpiLoader(service));
    }

    /**
     * 按照名称加载SPI实现
     *
     * @param name SPI名称
     * @return SPI impl
     */
    public S load(String name) {
        Validate.notEmpty(name, "SPI name cannot be empty");

        load();

        Class<? extends S> clazz = spiName2Class.get(name);
        if (clazz == null) {
            throw new SsqlException("no Provider class's aliasName is " + name);
        }

        return createInstance(clazz);
    }

    /**
     * 加载默认SPI实现
     *
     * @return default SPI impl
     */
    public S loadDefault() {
        load();

        if (defaultClass == null) {
            return null;
        }

        return createInstance(defaultClass);
    }

    /**
     * 加载优先级最高的SPI实现
     *
     * @return SPI impl
     */
    public S loadHighestPriority() {
        load();

        if (classList.isEmpty()) {
            return null;
        }

        return createInstance(classList.get(0));
    }

    /**
     * 加载所有SPI实现
     *
     * @return SPI impl list
     */
    public List<S> loadAll() {
        load();

        return createInstances(classList);
    }

    private void load() {
        if (!loaded.compareAndSet(false, true)) {
            return;
        }

        String fullFileName = SPI_CFG_PREFIX + service.getName();
        ClassLoader classLoader = Thread.currentThread().getContextClassLoader();
        if (classLoader == null) {
            classLoader = ClassLoader.getSystemClassLoader();
        }

        Enumeration<URL> urls;
        try {
            urls = classLoader.getResources(fullFileName);
        } catch (IOException e) {
            throw new SsqlException("Error locating SPI configuration file, filename=" + fullFileName + ", classloader=" + classLoader, e);
        }

        if (urls == null || !urls.hasMoreElements()) {
            return;
        }

        while (urls.hasMoreElements()) {
            URL url = urls.nextElement();
            try (InputStream in = url.openStream();
                 BufferedReader br = new BufferedReader(new InputStreamReader(in, StandardCharsets.UTF_8))) {
                String line;
                while ((line = br.readLine()) != null) {
                    if (StringUtils.isBlank(line)) {
                        continue;
                    }

                    line = line.trim();
                    int commentIndex = line.indexOf("#");
                    if (commentIndex == 0) {
                        // 跳过注释
                        continue;
                    }

                    if (commentIndex > 0) {
                        line = line.substring(0, commentIndex);
                        line = line.trim();
                    }

                    load(classLoader, line, fullFileName);
                }
            } catch (IOException e) {
                throw new SsqlException("error reading SPI configuration file", e);
            }
        }

        classList.sort((class1, class2) -> {
            Spi spi1 = class1.getAnnotation(Spi.class);
            int order1 = spi1 == null ? 0 : spi1.order();

            Spi spi2 = class2.getAnnotation(Spi.class);
            int order2 = spi2 == null ? 0 : spi2.order();

            return Integer.compare(order1, order2);
        });
    }

    private void load(ClassLoader classLoader, String implClazzName, String fullFileName) {
        Class<S> clazz;
        try {
            clazz = (Class<S>) Class.forName(implClazzName, false, classLoader);
        } catch (ClassNotFoundException e) {
            throw new SsqlException("class " + implClazzName + " not found", e);
        }

        if (!service.isAssignableFrom(clazz)) {
            throw new SsqlException("class " + clazz.getName() + "is not implement of " + service.getName() + ", SPI configuration file=" + fullFileName);
        }

        classList.add(clazz);
        Spi spi = clazz.getAnnotation(Spi.class);
        String spiName = (spi == null || StringUtils.isEmpty(spi.value()))
                ? clazz.getName()
                : spi.value();
        if (spiName2Class.containsKey(spiName)) {
            Class<? extends S> existClass = spiName2Class.get(spiName);
            throw new SsqlException("Found repeat SPI name for " + clazz.getName() + " and "
                    + existClass.getName() + ",SPI configuration file=" + fullFileName);
        }
        spiName2Class.put(spiName, clazz);

        if (spi != null && spi.isDefault()) {
            if (defaultClass != null) {
                throw new SsqlException("Found more than one default Provider, SPI configuration file=" + fullFileName);
            }
            defaultClass = clazz;
        }
    }

    private List<S> createInstances(List<Class<? extends S>> clazzList) {
        if (CollectionUtils.isEmpty(clazzList)) {
            return Collections.emptyList();
        }

        return clazzList.stream().map(this::createInstance).collect(Collectors.toList());
    }

    private S createInstance(Class<? extends S> clazz) {
        Spi spi = clazz.getAnnotation(Spi.class);
        boolean singleton = spi == null || spi.isSingleton();

        if (!singleton) {
            return createInstance0(clazz);
        }

        return className2Singleton.computeIfAbsent(clazz.getName(), key -> createInstance0(clazz));
    }

    private S createInstance0(Class<? extends S> clazz) {
        try {
            return service.cast(clazz.newInstance());
        } catch (Exception e) {
            throw new SsqlException(clazz.getName() + " could not be instantiated");
        }
    }
}

```

核心代码100行不到，我们就实现了一个功能强大的SPI，简单测试一下：

```java
public interface Foo {}

@Spi(order = Spi.ORDER_HIGHEST)
public class FooImpl1 implements Foo {}

@Spi(isDefault = true, order = -10)
public class FooImpl2 implements Foo {}

@Spi(value = "foo3", isSingleton = false)
public class FooImpl3 implements Foo {}

class SpiLoaderTest {
    @Test
    void testLoadInstance() {
        Foo foo = SpiLoader.of(Foo.class).load("foo3");
        Assert.assertTrue(foo instanceof FooImpl3);
    }

    @Test
    void testLoadDefaultInstance() {
        Foo foo = SpiLoader.of(Foo.class).loadDefault();
        Assert.assertTrue(foo instanceof FooImpl2);
    }

    @Test
    void testLoadInstanceByPriority() {
        Foo foo = SpiLoader.of(Foo.class).loadHighestPriority();
        Assert.assertTrue(foo instanceof FooImpl1);
    }

    @Test
    void testLoadAllInstances() {
        List<Foo> foos = SpiLoader.of(Foo.class).loadAll();
        Assert.assertEquals(3, foos.size());
    }
}
```



