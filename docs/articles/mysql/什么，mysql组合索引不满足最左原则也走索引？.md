### 起因
某天，有位同事正在研究mysql组合索引，我们都知道mysql组合索引满足最左原则，假设建立了(a,b,c)组合索引，当单独使用b、c是不走索引的，可同事测试后发现不满足也可以走索引，有图有真相，纳尼，
这波跟我的mysql企业级理解不一致阿，必须建个表测试一番

#### 新建测试表

```sql
CREATE TABLE test_index (
	id int auto_increment NOT NULL,
	a varchar(10) NULL,
	b varchar(10) NULL,
	c varchar(10) NULL,
	CONSTRAINT test_pk PRIMARY KEY (id)
)
ENGINE=InnoDB
DEFAULT CHARSET=utf8
COLLATE=utf8_general_ci;
CREATE INDEX test_a_IDX USING BTREE ON test_index (a,b,c);

insert into test_index (a, b, c)
values 
(1,1,1),
(2,2,2),
(3,3,3),
(4,4,4),
(5,5,5);
```

#### 测试sql

```sql
explain select * from test_index ti where b = '2'
```

哦豁，看执行计划是使用了索引，不会吧不会吧，怎么回事？

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7dde9ce1022d4887af841be4000c7a7c~tplv-k3u1fbpfcp-watermark.image?)

正当我百思不得其解时，再看了一下执行计划，发现了一丝线索

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/20397c4eb0354fdea41992f0248e202b~tplv-k3u1fbpfcp-watermark.image?)

原来是走了覆盖索引，但是查询的时候还是进行了全表扫描，我们可以看到rows=5以及Extra里的Using where，只不过select * 查询的字段恰好能从组合索引（覆盖索引）中取到，优化器会使用该索引，避免去回表查询

### 总结

组合索引优点

1. 减少空间开销

   分析业务sql的常用的字段条件，尽量建立组合索引，减少单个索引的建立，索引过多往往会降低写操作的性能，和增加磁盘空间的开销

2. 覆盖索引

   当优化器分析到select的字段能够在覆盖索引中查询到，便会直接从覆盖索引中获取数据，不做回表操作，减少了很多随机的IO操作，覆盖索引是主要的提升性能的优化手段之一

3. 效率高

   多个条件的查询，假如只建立一个个单列索引，查询的效率并不高，会筛选几次数据才能得出最终结果，但如果使用组合索引且满足最左原则的情况下，通过索引可以一次筛选出查询数据，效率无疑是大大提高


> 注意，最左原则

> 注意，最左原则

> 注意，最左原则