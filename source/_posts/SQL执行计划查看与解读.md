title: SQL执行计划查看与解读
date: '2019-05-10 15:05:32'
tags: [explain, sql调优]
categories: [工作笔记]
---
&emsp;&emsp;公司之前组织了一场关于《据库调优实践》的培训，恰好自己从事的项目也是统计类的，涉及了很多SQL语句。其中有关SQL执行计划的讲解，对SQL的分析和优化有很大的帮助。所以我也是结合网上资料，根据自己的理解整理了一下。因为公司的数据库是基于PostgreSQL的，所以整理的方向也是基于PostgreSQL。

<!-- more -->

## 执行计划查看命令

### explain 语法

>explain[(option[...])]  
>explain [analyze][verbose]statement  
>可选的option选项有：  
>- analyze[boolean]：得到statement的真实运行时间。默认是false  
>- verbose[boolean]：得到statement语句的执行计划和执行计划中的每个节点的详细信息。默认为false  
>- costs[boolean]：得到计划中每个接地哪的cost，rows，width的估算值，默认为true  
>- buffers[boolean]：analyze出现时可选。缓存的使用情况  
>共享缓存(shared blocks)的hit，read，dirtied，written数值  
>本地缓存(local blocks)的hit，read，dirtied,written数值  
>临时快(temp blocks)的read,written数值  
>- timing[boolean]：analyze出现时可选。显示每个节点的启动时间和总时间花费。默认true  
>- format{text|xml|json|yaml}：指定执行计划的输出格式  
>text：默认值。以行为单位，显示每个结点的计划信息，以缩进格式表示子节点的计划信息。buffers参数时的文本格式，只输出非零值  
>xml：xml格式  
>json：json格式  
>yaml：以yaml格式显示执行计划  

### 常用组合

- 一般查询  
explain analyze select ... ;
- 查询缓存及详细信息  
explain (analyze,verbose,buffers) select ... ;
- 针对更新插入删除的执行计划查询  
begin;  
explain analyze insert/update/delete ... ;  
rollback;  

> 需要注意的是，explain analyze 会分析SQL语句的执行计划，并真正执行SQL语句。而默认情况下，PostgreSQL数据库的事务是自动开启、提交的。那么，对于DML语句来说，如果要使用explain analyze来查看其执行计划的话，就得格外注意了。如`explain analyze delete from p_table ;`查看一条删除操作SQL的执行计划，其结果就是，真正的把表里的数据删除了，且提交了。对于所有的DML操作，如果只是想通过explain analyze来查看其执行计划的话，需要在执行之前，显示开启事务，然后查看执行计划，最后回滚事务。

## 执行计划解读

### 关键字
- cost:重要的指标。cost=0.00..16.11有两个部分，启动时间=0.00 和总时间=16.11。单位是毫秒。这个指标也只是预测值。启动时间也有解释为找到符合条件的第一行所花的时间。
- rows:返回的行数，如果执行vacuum和analyze那么返回的结果更加接近实际值
- width:查询结果所有字段的总宽度，并非关键指标。
- actual time：实际花费的时间。
- loops：循环的次数
- buffers：缓冲命中数
- output： 输出的字段名
- planning time: 生成执行计划时间
- execution time：执行执行计划时间

### 阅读顺序
- 嵌套层次最深的，最先执行
- 同样嵌套深度的，从上到下，先予执行
- 每一步的cost包括上一步

### 节点
- Seq Scan: 表扫描
- Index Scan：索引扫描(读取索引块，然后读取数据文件)
- Index Only Scan：索引只读扫描(只读取索引文件，根据映射文件获取数据)
- Nested [(type)] Loop：嵌套循环连接。type可能是Inner，left，right，full，semi，anti。inner的可以显示省略
- Merge[(type)] Join 归并连接。type同上
- Hash[(type)] Join：哈希连接。type同上 

## 解读案例 

&emsp;&emsp;项目中实际用到的业务SQL，以及执行计划分析结果如下：

```sql
EXPLAIN SELECT aj.n_ajbs AS ajbh,
    aj.d_sarq AS larq,
    aj.d_jarq AS jarq,
    aj.c_baah AS ah,
    aj.n_jafs as jafs,
    aj.n_saly AS ajly,
    fy.fbdm AS jbfy,
    aj.n_cbbmbs AS cbsptbs,
    aj.c_yyhsqsx AS zxsx,
    jg.n_lx AS jglx,
    spzz.n_spzxzzcylx AS spzxzzcylx,
    sl.n_sjslts AS sjslts,
    (((sl.n_fdqxn * 365) + (sl.n_fdqxy * 30)) + sl.n_fdqxr) AS fdqx,
    ((sl.n_sjslts - (((sl.n_fdqxn * 365) + (sl.n_fdqxy * 30)) + sl.n_fdqxr)) -
     ((sxyc.n_pzyzqxy *
     30) + sxyc.n_pzyzqxr)) AS csxts,
    upper("substring"((aj.c_stm)::text, 5, 40)) AS stm
   FROM (((((((db_zxyy.t_zxyy aj
     LEFT JOIN db_fy.t_jg jg ON ((jg.n_jgbs = aj.n_cbbmbs)))
     LEFT JOIN ( SELECT ss.c_stm_zxyy,
            count(1) AS n_slfs
           FROM db_zxyy.t_zxyyslfsxx ss
          WHERE (ss.n_slfs = ANY (ARRAY[1, 4]))
          GROUP BY ss.c_stm_zxyy) slfs ON ((slfs.c_stm_zxyy = aj.c_stm)))
     LEFT JOIN ( SELECT sa.c_stm_zxyy,
            count(1) AS n_gksl
           FROM db_zxyy.t_zxyyslfsxx sa
          WHERE (sa.n_gksl = 1)
          GROUP BY sa.c_stm_zxyy) gksl ON ((gksl.c_stm_zxyy = aj.c_stm)))
     LEFT JOIN ( SELECT t_zxyyspzxzzcygc.c_stm_zxyy,
            count(1) AS n_spzxzzcylx
           FROM db_zxyy.t_zxyyspzxzzcygc
          WHERE (t_zxyyspzxzzcygc.n_spzxzzcylx = 4)
          GROUP BY t_zxyyspzxzzcygc.c_stm_zxyy) spzz ON ((spzz.c_stm_zxyy = aj.c_stm)))
     LEFT JOIN ( SELECT sxyc_1.c_stm_zxyy,
            sum(sxyc_1.n_pzyzqxy) AS n_pzyzqxy,
            sum(sxyc_1.n_pzyzqxr) AS n_pzyzqxr
           FROM db_zxyy.t_zxyyqxycxx sxyc_1
          GROUP BY sxyc_1.c_stm_zxyy) sxyc ON ((aj.c_stm = sxyc.c_stm_zxyy)))
     LEFT JOIN db_zxyy.t_zxyyslzxqxgl sl ON ((aj.c_stm = sl.c_stm)))
     LEFT JOIN db_sftj.t_fy_zh fy ON ((aj.n_jbfy = fy.fyid)))
  WHERE (((aj.d_sarq < '2016-01-01 00:00:00'::timestamp without time zone) OR ((aj.n_saly IS NOT 
  NULL) AND ((aj.c_baah)::text ~~ '%执异%'::text))) AND (aj.c_baah IS NOT NULL) AND ((aj.d_jarq 
  IS NULL) OR (aj.d_jarq > '2016-12-31 00:00:00'::timestamp without time zone)) AND (aj.n_saly = 
  ANY (ARRAY[1, 2, 3, 4, 5])) AND (aj.c_yyhsqsx = ANY (ARRAY['1'::text, '2'::text, '3'::text, 
  '4'::text, '5'::text, '6'::text, '7'::text, '255'::text])));
```
```sql
Nested Loop Left Join  (cost=1548.57..1695.16 rows=1 width=183)
  ->  Nested Loop Left Join  (cost=1548.29..1686.81 rows=1 width=167)
        Join Filter: (aj.c_stm = sxyc_1.c_stm_zxyy)
        ->  Nested Loop Left Join  (cost=1533.32..1662.68 rows=1 width=151)
              Join Filter: (t_zxyyspzxzzcygc.c_stm_zxyy = aj.c_stm)
              ->  Hash Right Join  (cost=877.59..1006.92 rows=1 width=143)
                    Hash Cond: (jg.n_jgbs = aj.n_cbbmbs)
                    ->  Seq Scan on t_jg jg  (cost=0.00..114.23 rows=4023 width=8)
                    ->  Hash  (cost=877.58..877.58 rows=1 width=139)
                          ->  Hash Right Join  (cost=785.57..877.58 rows=1 width=139)
                                Hash Cond: (fy.fyid = aj.n_jbfy)
                                ->  Seq Scan on t_fy_zh fy  (cost=0.00..77.00 rows=4000 width=7)
                                ->  Hash  (cost=785.56..785.56 rows=1 width=138)
                                      ->  Seq Scan on t_zxyy aj  (cost=0.00..785.56 rows=1 width=138)
                                            Filter: ((c_baah IS NOT NULL) AND ((d_jarq IS NULL) 
                                            OR (d_jarq > '2016-12-31 00:00:00'::timestamp without 
                                            time zone)) AND ((d_sarq < '2016-01-01 
                                            00:00:00'::timestamp without time zone) OR ((n_saly 
                                            IS NOT NULL) AND ((c_baah)::text ~~ '%执异%'::text))) 
                                            AND (n_saly = ANY ('{1,2,3,4,5}'::integer[])) AND 
                                            (c_yyhsqsx = ANY ('{1,2,3,4,5,6,7,255}'::text[])))
              ->  HashAggregate  (cost=655.73..655.74 rows=1 width=37)
                    Group Key: t_zxyyspzxzzcygc.c_stm_zxyy
                    ->  Seq Scan on t_zxyyspzxzzcygc  (cost=0.00..655.73 rows=1 width=37)
                          Filter: (n_spzxzzcylx = 4)
        ->  HashAggregate  (cost=14.96..17.79 rows=282 width=45)
              Group Key: sxyc_1.c_stm_zxyy
              ->  Seq Scan on t_zxyyqxycxx sxyc_1  (cost=0.00..11.98 rows=398 width=45)
  ->  Index Scan using i_zxyyslzxqxgl_stm on t_zxyyslzxqxgl sl  (cost=0.29..8.30 rows=1 width=53)
        Index Cond: (aj.c_stm = c_stm)
```
&emsp;&emsp;(下面都是个人理解，如有误请指正)
&emsp;&emsp;1、根据上述的阅读顺序，嵌套层次最深最先执行，可以得出第14行`Seq Scan on t_zxyy aj`应该最先执行，该节点扫描了 t_zxyy 表，并对 Filter 下的条件（也就是SQL中的where条件）做了过滤。
&emsp;&emsp;2、继续往下看，下一个层次节点为，12行的`Seq Scan on t_fy_zh fy`以及13行的`Hash`，这两个嵌套深度相同所以依次执行。其中12行对 t_fy_zh 表进行了扫描，而13行的意思是将上一步（也就是第14行）中扫描结果作为Hash节点的输入建立哈希表。
&emsp;&emsp;3、接下来的节点是第10行的`Hash Right Join`，这是一个连接操作，`Hash Cond`是连接的条件，结合上一步，可理解为：扫描 t_fy_zh 表并和哈希表中的每一条数据进行匹配连接。
&emsp;&emsp;4、继续，有三个节点在同一层次，分别是8行的`Seq Scan on t_jg jg`、9行的`Hash`以及24h行的`Seq Scan on t_zxyyspzxzzcygc`,依次执行就是，8行对 t_jg 进行扫描，9行对上一步（第10行）的结果作为Hash节点的输入建立哈希表，24行是对 t_zxyyspzxzzcygc 进行扫描，Filter是过滤条件。
&emsp;&emsp;5、接下来是6行的`Hash Right Join`、22行的`HashAggregate`和28行的`Seq Scan on t_zxyyqxycxx sxyc_1`。6行和28行的理解不在赘述。22行的意思是**聚合函数**的实现，`Group Key`则是对该字段进行聚合，可以理解为：对24行扫描后的结果根据摸个字段进行聚合。
&emsp;&emsp;6、第4行和第26行，第4行也是连接操作，Join Filter 为连接的条件，第6行是聚合函数。
&emsp;&emsp;7、最后是第2行和第29行，29行`Index Scan` 为索引扫描，表是该表连接走索引；

> 总的执行结果结合原SQL大概是这样，先对t_zxyy执行where条件，输出结束存入Hash表，然后与t_fy_zh关联，结果存入Hash表，然后是关联t_jg，与此同时对 t_zxyyspzxzzcygc 进行聚合操作，然后两着关联（记为结果1），在做关联的操作的同时，对 t_zxyyqxycxx 进行聚合操作，然后与结果1再做关联，最后是与 t_zxyyslzxqxgl 索引关联；

> 通过这个结果我们可以参考每个节点的计划时间，根据时间的长短来进行SQL优化。此外，通过该实例可以看到，在原SQL中有两张子表关联 t_zxyyslfsxx 实际并未用到，所以可以删去无用的连接。

## 总结

&emsp;&emsp;上诉案例解读也是自行理解后的结果，没有人帮我验证对或不对，也没有实践优化过SQL语句，真是惭愧😥😥。不过有强大的DBA在，SQL优化什么的不在话下。
&emsp;&emsp;这里推荐一篇博客园的博文写的相当详细：[PostgreSql的Explain命令详解](https://www.cnblogs.com/smiler/p/10250110.html)





