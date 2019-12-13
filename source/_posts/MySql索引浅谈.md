---
title: MySQL索引浅谈
date: 2019-12-02 09:21:08
tags: [mysql]
categories: MySQL
---

`MySQL`数据库相信大家都应该很熟悉。在目前大多数互联网公司都会首选MySQL做为自己业务相关的持久化储存，而在数据库中索引又是非常重要的一块内容，基本上所有的业务查询都是需要走到索引，如果数据量比较大，进行全表扫描（没有走到索引）会非常消耗系统IO和CPU导致，性能很低。所以知道熟悉数据库的索引是很重要的。

#### 什么是索引

索引（Index），是一本书籍的重要组成部分，它把书中的重要名词罗列出来，并给出它们相应的页码，方便读者快速查找该名词的定义和含义。传统意义上的索引，举个例子我们去根据偏旁部首去查询字的时候，那个对应字会给出对应额页码，而不是我们一页一页去找。那个记录字对应页数的那几页就是索引。（方便我们快速的定位位置）

#### `MySQL`的索引

`MySQL`中的索引也叫键（key）是存储引擎用于快速查找记录的一种数据结构。`MySQL`中的存储引擎也是像书中索引一样，先找到索引对应的值，然后在根据匹配的索引记录找到对应的数据行。

#### 索引类型

在`MySQL`中常见两种索引类型，一种是Hash索引，一种是B-Tree索引。从名字中我们就能看出来两种索引的数据结构式不一样的，Hash索引存储结构的是一个Hash散列表；B-Tree索引使用的B+Tree的数据结构。

B-Tree索引：在没有指定索引类型时，默认的一般都是B-Tree索引，大多数存储引擎也支持这种数据结构。B-Tree索引使用的时B+Tree的数据结构（InnoDB使用的是原数据的格式存储，MyISAM使用的是物理地址）。B+Tree是一种**多路查询平衡树**，它存储数据也是有顺序的所以特别适合范围查找。而且B-Tree索引也是支持查找方式也很多；等值查找、匹配最左前缀查找、匹配范围查找、匹配列前缀查找、只访问索引查找（覆盖索引）。在B-Tree索引里面顺序是很重要的，后面我们会在符合索引里面也讲到这个。（B-Tree子叶存储数据顺序，左子叶<父节点<右子叶）

Hash索引：Hash索引是基于Hash表去实现，目前也只有Memory存储引擎支持。由于是Hash表的数据结构，Hash索引在精确值匹配搜索（等值查询）索引所有列的时候，索引才会生效，如果是范围查询，非等值要进行全表扫描。（把索引值计算Hash key 存在Hash表中，Hash表中没有顺序所以，范围查找的时候会全表扫描）Hash索引不存储字段值，只存储Hash值和行指针，一般在存储URL这种数据（只能精确查找）的时候会使用Hash索引。但是如果有大量重复值也会引起很大的Hash碰撞，导致索引失效（一般重复性高的字段也不要作为索引）

上面说的这两种类型索引是比较常见的索引，我们一般在生产上使用最多的也就是B-Tree索引，最多的也就是`InnoDB`的B-Tree。

#### `InnoDB`的索引类型

`InnoDB`是我现在用的最多也一种存储引擎，`InnoDB`的表是基于聚簇索引建立的。`InnoDB`会有主键索引和二级索引两种。`InnoDB`采用`MVCC`（多版本控制）支持高并发，并且实现了四个标准隔离级别（读未提交，读已提交，可重复读，串行化）。默认是级别是`REPEATABLE READ` （可重复读）。通过间隙锁防止幻读出现，而且还通过对索引中的间隙进行锁定，防止幻影行插入。

##### 聚簇索引和非聚簇索引

索引除了从存储的数据结构区分为Hash索引和B+Tree，也可以跟据存储数据的内容来区分成聚簇索引和非聚簇索引。

聚簇索引：聚簇索引并不是单独的存储索引类型，是一种数据存储方式。`InnoDB`的聚簇索引在同一个叶子页和节点（节点存储索引列，叶子页存放具体的数据行）保存了B-Tree的索引和数据行。术语是数据行和相邻的键值（key 索引）紧凑存储在一起。简单来说就是索引和数据行在同一颗B+Tree上（数据行和索引存储在同一个叶子页）。因为无法同时把数据行同时存放到两个不同的地方，所以一个表只能有一个聚簇索引（主键）；不过覆盖索引可以模拟多个聚簇索引情况。

一般`InnoDB`存储引擎中会选择唯一非空索引来做，而且最好是这个主键是有顺序的，B+Tree页是有顺序的这样页方便存储，同时页可以提高查询的性能。

聚簇索引的优点：

1、可以把相关的数据保存到一起，只要查到主键索引可以把所有数据就带出来，减少不必要的IO。

2、数据访问更快，聚簇索引讲索引和数据保存同一个Tree，因此聚簇索引会比非聚簇查询更快

3、使用覆盖索引扫描可以直接使用叶节点的主键值，不用回表。（后面我们会说回表和覆盖索引）

聚簇索引缺点：

1、二级索引（非聚簇索引）要查找两次，而不是一次

2、二级索引（非聚簇索引）比想象更大因为在二级索引的叶子节点包含所有主键列

3、更新聚簇索引列的代价会很高因为，`InnoDB`会将每一个更新的行移动到新的位置（包括数据行）

4、插入速度严重依赖插入顺序，如果按照主键顺序会很快，负责要重新排序性能会很低。

5、当数据行比较稀疏（主键顺序间隔很大），由于页分裂的表多，会导致全表扫描会变慢。

6、当插入新行，或者更新行的时候，导致原来的叶子节点满掉了。存储引擎会将一个存储叶子页分裂成两个页来容纳数据。（刚刚的操作叫页分裂）页分裂会导致存储空间变大。

非聚簇索引：非聚簇索引的定义和聚簇索引就正好相反，即索引值和数据行分开存储，而不是紧凑的存在一起。这中就像`MyIASM`存储数据那样，叶子页存放的不是数据，而是数据锁存储的物理地址。这样的话，每次查找都要去查询一次物理地址把对应数据行拿出来，而不是一次查询就可以带出所有的值，多了一次IO。（`InnoDB`的二级索引叶子页存的不是物理地址，而是主键）

回表：在二级索引（非主键索引，非聚簇所以 ）查询时，不能一次把所有的数据都带出来，还要在去查询一次（查询主键）获取数据，这个过程叫回表。

##### 覆盖索引

上面提到回表，一般在数据量不大的收还可以。当数据量很大的时候走二级索引还是有点慢，因为它有两次查询，所以，我们能不能走主键索引而不是走二级索引。

覆盖索引：如果一个索引包含（或者覆盖）要查询的所有字段的值，我们称之为覆盖索引。

大家通常都会在where的查询条件来创建合适的索引，而优秀的索引设计应该考虑到整个查询。而不是只是where条件部分。`MySQL`通过使用索引直接获取列数据（一般索引都会被缓存）而不是根据主键在查询表一次获取索、所要的数据（回表）。这样就可以极大的提高性能

##### 复合索引和最左侧匹配原则

除了上面说的索引之外还有，索引可以根据个数来分单个索引和符合索引，单个索引是由一个列（一个字段）组成的索引。复合索引是由多个列组成的一个索引。在查找是时候在where语句中用AND相连接

举个例子，我们对一个User表，创建三个索引姓名，年龄，性别。我们我们知道一般像性别这种是不会作为索引，但是它可以和姓名和年龄组成一个联合索引，这样姓名 AND 年龄 AND 性别这样他们区别度就足够大了。

```sql
KEY index_name_age_sex on user(name,age,sex);
```

联合索引 index_name_age_sex实际建立了(name)、(name,age)、(name,age,sex)三个索引。而且在查询的时候实现最左匹配原则。

```sql
select * from user where name ='aa' and age= 13 and sex = 1  --- 会走name age sex 三个联合索引
select * from user where name = 'nn' and age =14 and adress = 'home' --- 会走 name age 两个联合索引
select * from user where name ='hh' and sex = 1 ---会走到 name的那个索引

select * from user where age = 11 and sex = 1 --- 不会走到索引

select * from user where age =11 and name ='ll' and sex =0 --这种是可以走到name age sex 三个联合索引，mysql语句在执行的时候会重新排序，所以前面的顺序不是那么重要对于索引，而是能匹配的上（可能面试会问的问题）

```

![](https://raw.githubusercontent.com/ClaudeWilliam/pic-blog/master/img/55.png)

最左匹配原则：在`mysql`建立联合索引时会遵循最左前缀匹配的原则，即最左优先，在检索数据时从联合索引的最左边开始匹配。

在简单的查询中，也要把能过滤的多键写在前面，`mysql`执行匹配条件是从左匹配，然后逐级的过滤数据。

```sql
---如果上面三个条件没有索引，走的是全表扫描
select * from user where sex =1 and age =10 and name = 'kk'
select * from user where name ='kk'and age = 10 and sex =1
---下面的sql就会比上面的高效
```

##### 唯一索引

索引列的数据不可以重复，保证唯一性。这个和主键索引类似，这给我就不作为重点说了。一般在保证数据唯一性的情况下使用，一般最好不用为NULL，在`mysql`中NULL不算是唯一值，也就是NULL可以在唯一索引中存在多个NULL。

##### 索引下推

索引条件下推（index condition pushdown）简称ICP，这个是`MySQL`5.7版本后出现的功能。而且新版本的`MySQL`都是默认开启这个功能.

```sql
SET optimizer_switch = 'index_condition_pushdown=off'; --关闭
SET optimizer_switch = 'index_condition_pushdown=on';  --开启
```

`Index Condition Pushdown` (`ICP`)是`MySQL`用索引去表里取数据的一种优化。如果禁用`ICP`，引擎层会穿过索引在基表中寻找数据行，然后返回给`MySQL Server`层，再去为这些数据行进行WHERE后的条件的过滤。 `ICP`启用，如果部分WHERE条件能使用索引中的字段，`MySQL Server` 会把这部分下推到引擎层。存储引擎通过使用索引条目，然后推索引条件进行评估，使用这个索引把满足的行从表中读取出。`ICP`能减少引擎层访问基表的次数和`MySQL Server`访问存储引擎的次数。

![](https://raw.githubusercontent.com/ClaudeWilliam/pic-blog/master/img/icp01.png)

在没有使用`ICP`我们可以看到，存储引擎只会拿到跟索引相关的数据，其他的数据的where条件过滤都在`MySQL Server`层进行。`MySQL Server`会多次根据where条件去查询存储引擎。这样效率比较低。

![](https://raw.githubusercontent.com/ClaudeWilliam/pic-blog/master/img/icp02.png)

在使用了`ICP`中，存储引擎会把where条件判断下推到存储引擎上，在存储引擎上进行where条件判断，过滤出符合条件的数据后再返回给Server层。而由于在引擎层就能够过滤掉大量的数据，这样无疑能够减少了对base table和`mysql server`的访问次数。 从而提升了性能。

举个栗子

```sql
select * from user where user_id =14 and name_id like '%2%' and country like '%a%' and name like '%a%' --这个sql在使用ICP后性能会有很大的提高
```

 在没有`ICP`前，由于优化器只能只能使用前缀索引来过滤满足条件的查询，那么`mysql`只能够利用索引的第一个字段user_id，来扫描user表满足user_id=14条件的记录，而后面的name_id和country由于使用了模糊查询，而不能在索引中继续过滤满足条件的记录，这样就导致了Server层对user表的扫描多次。

有了`ICP`，`mysql`在读取user表前，继续检查满足name_id和country条件的记录,这个行为在存储引擎层完成。直接把过滤好的返回给Server层，就减少了Server层的操作和对存储引擎的访问次数。 把之前在SERVER层的下推到引擎层去处理，提高了效率。

简单一句话，索引下推就是把where查询条件推到存储引擎去过滤数据。

- 支持`InnoDB`和`MyISAM`表。
- ICP只能用于二级索引，不能用于主索引。
- 并非全部where条件都可以用ICP筛选。如果where条件的字段不在索引列中,还是要读取整表的记录到server端做where过滤。
- ICP的加速效果取决于在存储引擎内通过ICP筛选掉的数据的比例。
- 5.6 版本的不支持分表的ICP 功能，5.7 版本的开始支持。
- 当`sql` 使用覆盖索引时，不支持`ICP `优化方法。
- 当`sql`需要全表访问时,`ICP`的优化策略可用于`range`, `ref`,` eq_ref`, ` ref_or_null` 类型的访问数据方法 。

#### 判断查询是否走到索引

判断一个`sql`是否能走到索引就是看这个`sql`语句的执行计划。一般就是`explain` +`sql`语句去查看和分析.

![](https://raw.githubusercontent.com/ClaudeWilliam/pic-blog/master/img/explain.png)

`id`:`SQL`执行的顺序的标识,`SQL`从大到小的执行

- id相同时，执行顺序由上至下

- 如果是子查询，id的序号会递增，id值越大优先级越高，越先被执行

- id如果相同，可以认为是一组，从上往下顺序执行；在所有组中，id值越大，优先级越高，越先执行

`select_type`:表示查询中每个select子句的类型

- `SIMPLE`(简单SELECT,不使用UNION或子查询等)

- `PRIMARY`(查询中若包含任何复杂的子部分,最外层的select被标记为PRIMARY)

- `UNION`(UNION中的第二个或后面的SELECT语句)

- `DEPENDENT UNION`(UNION中的第二个或后面的SELECT语句，取决于外面的查询)

- `UNION RESULT`(UNION的结果)

- `SUBQUERY`(子查询中的第一个SELECT)
-  `DEPENDENT SUBQUERY`(子查询中的第一个SELECT，取决于外面的查询)
- `DERIVED`(派生表的SELECT, FROM子句的子查询)
- `UNCACHEABLE SUBQUERY`(一个子查询的结果不能被缓存，必须重新评估外链接的第一行)

`table`:查询表名

`possible_keys`:可能走到的索引，该索引将被列出，但不一定被查询使用。只有一个最好，这样Server层不用去判断，具体那个索引。

`key`:如果key有值说明可以走到索引，否则走不到索引。

`key_len`：索引长度。一般索引长度越小效率越高。

`rows`:是对应要扫描多少行数据，这个越小说明效率越高

`type`:叫访问类型，常用的类型有： `ALL`, `index`,`  range`,` ref`,` eq_re`f,` cons`t, `system`` NULL`

- `ALL`：Full Table Scan， `MySQL`将遍历全表以找到匹配的行

- `index`: Full Index Scan，index与ALL区别为index类型只遍历索引树

- `range`:只检索给定范围的行，使用一个索引来选择行

- `ref`: 表示上述表的连接匹配条件，即哪些列或常量被用于查找索引列上的值

- `eq_ref`: 类似`ref`，区别就在使用的索引是唯一索引，对于每个索引键值，表中只有一条记录匹配，简单来说，就是多表连接中使用`primary key`或者 `unique key`作为关联条件

- `const`、`system`: 当`MySQL`对查询某部分进行优化，并转换为一个常量时，使用这些类型访问。如将主键置于where列表中，`MySQL`就能将该查询转换为一个常量,`system`是`const`类型的特例，当查询的表只有一行的情况下，使用system

- `ref_or_null`: `MySQL`在优化过程中分解语句，执行时甚至不用访问表或索引，例如从一个索引列里选取最小值可以通过单独索引查找完成

`system > const > eq_ref > ref > fulltext > ref_or_null > index_merge > unique_subquery > index_subquery > range > index > ALL`

一般来说，得保证查询至少达到range级别，最好能达到ref，否则就可能会出现性能问题。

`ref`：表示上述表的连接匹配条件，即哪些列或常量被用于查找索引列上的值。

`Extra`:该列包含`MySQL`解决查询的详细信息

- `Distinct`:一旦`MYSQL`找到了与行相联合匹配的行，就不再搜索了

- `Using where`:列数据是从仅仅使用了索引中的信息而没有读取实际的行动的表返回的，这发生在对表的全部的请求列都是同一个索引的部分的时候，表示`mysql`服务器将在存储引擎检索行后再进行过滤

- `Using temporary`：表示`MySQL`需要使用临时表来存储结果集，常见于排序和分组查询。

  看到这个的时候，查询需要优化了。这 里，`MYSQL`需要创建一个临时表来存储结果，这通常发生在对不同的列集进行ORDER BY上，而不是GROUP BY上

- `Using filesort`：`MySQL`中无法利用索引完成的排序操作称为“文件排序”

  查询需要优化。`MYSQL`需要进行额外的步骤来发现如何对返回的行排序。它根据连接类型以及存储排序键值和匹配条件的全部行的行指针来 排序全部行

- `Using join buffer`：改值强调了在获取连接条件时没有使用索引，并且需要连接缓冲区来存储中间结果。如果出现了这个值，那应该注意，根据查询的具体情况可能需要添加索引来改进能。

- `Impossible where`：这个值强调了where语句会导致没有符合条件的行。(需要优化)

- `Select tables optimized away`：这个值意味着仅通过使用索引，优化器可能仅从聚合函数结果中返回一行（需要优化）

当type 显示为 “index” 时，并且Extra显示为“Using Index”， 表明使用了覆盖索引。

#### 建立索引注意是事项

1、索引不会包含有NULL的列

只要列中包含有NULL值，都将不会被包含在索引中，复合索引中只要有一列含有NULL值，那么这一列对于此符合索引就是无效的。

2、用短索引

对串列进行索引，如果可以就应该指定一个前缀长度。例如，如果有一个char（255）的列，如果在前10个或20个字符内，多数值是唯一的，那么就不要对整个列进行索引。短索引不仅可以提高查询速度而且可以节省磁盘空间和I/O操作。

3、索引要建立在值比较唯一的字段上。（波动性较大的字段）

4、对于那些定义为text、image和bit数据类型的列不应该增加索引。因为这些列的数据量要么相当大，要么取值很少（和第2条类似）

5、引要建立在经常进行select操作的字段上。

 如果这些列很少用到，那么有无索引并不能明显改变查询速度。相反，由于增加了索引，反而降低了系统的维护速度和增大了空间需求。

6、经常与其他表进行连接的表，在连接字段上应该建立索引；

7、不要过多创建索引,除了增加额外的磁盘空间外,对于`DML`操作的速度影响很大,因为其每增删改一次就得从新建立索引

8、可以使用复合索引代替多个单列索引，提高查询效率，降低存储空间。

#### 导致索引失效几种情况

1、使用不等于操作符(<>, !=)

下面这种情况，即使在列user_no有一个索引，查询语句仍然执行一次全表扫描

```sql
select * from user where user_no <> 1000
select * from user where user_no != 1000

---通过把用 and 语法替代不等号进行查询，就可以使用索引，以避免全表扫描：上面的语句改成下面这样的，就可以使用索引了。 这个根据实际查询数据来判断，如果全盘扫描速度比索引速度要快则不走索引 。
select * from user shere user_no < 1000 and user_no > 1000
```

2、不匹配字段类型

在数据库声明的是int类型的user_no但是我查询时候，使用字符串类型

```sql
select * from user where user_no = '1000' ---索引失效
select * from user where user_no = 1000  ---索引生效
```

3、查询的时候使用左右模糊查询

```sql
---user_mobile是索引
select * from user where user_mobile like '%134%' ---索引不生效
select * from user where user_mobile lile '134%' ---右模糊索引是生效的
```

4、在查询条件中使用函数

```sql
select * from user where trunc(birth) = '01-MAY-82'; ---索引失效

select * from staff where birthdate < (to_date('01-MAY-82') + 0.9999); ---索引生效
```

5、is null 和is not null使索引失效

使用 is null 或is nuo null也会限制索引的使用，因为数据库并没有定义null值。如果被索引的列中有很多null，就不会使用这个索引

在被声明索引的字段在数据库设置的时候设置成not null 或者指定默认值 default 0或者''

6、索引字段使用or进行查询（在MyiASM存储引擎下是可以走索引的）

```sql
---user_name是索引 innoDB存储引擎
select * from user where user_name = '11' or user_name ='yy' ---索引失效
select * from user where user_name in('ll','yy') ---索引生效
```

7 、in在嵌套查询的时候，子查询的集合数据比较大时候索引失效

```sql
---user_index表数据比较多的情况下
select * from user where id in (select id from user_index); ---索引失效
 
select * from user where exists (select id from user_index); ---索引生效

--- IN适合于外表大而内表小的情况；
--- EXISTS适合于外表小而内表大的情况。
```

in是在内存里遍历比较，而exists需要查询数据库。所以数据量大的情况下使用exist

8、由于索引字段数据差异性性较小，导致索引没有生效

```sql
---sex字段只有0和1 这种不适合创建索引的字段
select * from user sex =1
```

但是这种请情况有的时候也会走到索引，就是数据存储中99%的数据sex =0 只有1%的数据等于1

```sql
select * from user sex =0 ; ---索引生效 1%数据 sex=0
```

字段本身波动性不强，但是数据的分布范围差异很大，就可以走到索引

最后最后把这篇文章送给一个，a fat man without hair。boss shen