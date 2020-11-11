## Quantifying TPC-H Choke Points and Their Optimizations

​	主要对TPC-H测试的瓶颈点 和 优化点 进行了系统性的总结。

1. **Motivation**：

   TPC-H 测试本来是 用来比较端到端的数据库系统，也有很多学术研究中使用它来测试一些算法实现的性能。但在*Fair Benchmarking Considered Diﬃcult*中指出了，几乎很少有人注意公平的基准测试，这样很容易不实地表现性能的信息，使得一个系统或一个算法看起来比另一个好（可能是无意的 或 故意的）。

   就比如本文中的测试显示，在TPC-H [Q6](#Q6)（重scan的任务）上，Hyrise系统比MonetDB要快了**近7倍**，从而想说明Hyrise在scan上的性能更好。但事实上Hyrise是通过一个优化来避免了82%输入数据的扫描，如果去掉该优化，Hyrise比起MonetDB的性能提升只能达到1.8倍。

   所以对TPC-H测试进行分析，不仅可以对优化器的开发提供指导，也可以更好地解释不同系统间的基准测试结果差异。

2. **瓶颈类别**：（文章主要考虑逻辑层面，不考虑具体实现的优化）

   - **计划层面**

     对中间结果的基数大小造成影响。比如 join顺序、谓词下推与排序、子查询展开。

   - **逻辑算子层面**

     在单个算子上对计划造成影响，不会改变输出的基数大小，但逻辑上效率能有提升。比如移除对其他group by条件有函数依赖的group by列。

   - **具体实现层面**

     也就是算子的物理实现效率。比如在join时使用bloomfilter，或者用Boyer-moore来进行LIKE表达式的匹配，SIMD，压缩执行等。

3. 计划层面瓶颈：

   - **Join顺序**

     除了Q1和Q6都是多张表上的查询，且多数的的Join谓词不是 显式的`a JOIN b ON a.id=b.id`，而是隐式的 `FROM a,b WHERE a.id=b.id`。所以首先是要找到对应的JOIN谓词，否则大部分会超时。

     第二步是决定表的JOIN顺序。论文分别对比了 不重新排序（即按照SQL规定的顺序）、基于动态规划的DPccp、自底向上的贪心算法，发现 [Q2](#Q2), [Q5](#Q5), [Q11](#Q11)都有很大提升，且只有[Q7](#Q7)上 动态规划和贪心算法给出了不同的结果（贪心算法将没有过滤的customers表和orders表先join了，破坏了原本最佳的顺序；动态规划 现在把customers表和nation表join来减小中间结果大小，这样做就和原本SQL中的顺序一致）。

     这里给出的建议是：**在早期的实现阶段，可以先考虑别的优化点，不用先考虑复杂的join排序算法**。（因为对于如TPC-DS的复杂查询，简单的贪心算法和min/max/count/distinct count这些统计数据就基本足够了）

   - **谓词下推与重排序**

     对于谓词的重排序，可以通过列的直方图等统计信息，来找到最严格的筛选条件，并优先执行这个谓词条件的筛选；同时还要考虑到 这个**列的类型**、string列的长度 以及 **压缩方法**等。

     实验表明，TPCH 大部分都能通过谓词下推和重新排序获得 大幅度的提升（仅[Q15](#Q15)、[Q18](#Q18)、[Q22](#Q22)获得小幅度提升）。所以建议是：优先实现谓词的优化。

   - **Between**

     在[Q19](#Q19)中，l_quantity使用了数值的 >=与<=谓词；[Q4](#Q4), [5](#Q5), [6](#Q6), [10](#Q10), [12](#Q12), [14](#Q14), [15](#Q15)则使用了日期的>=与<谓词来筛选范围，而between关键词则是闭区间。所以数据库应具备能力把它们视作一个组合谓词，一起下推、一起执行。

   - **Join依赖谓词的重复**

     [Q7](#Q7)和[Q19](#Q19)中，比如`(n1.n_name = '[NATION1]' and n2.n_name = '[NATION2]') or (n1.n_name = '[NATION2]' and n2.n_name = '[NATION1]')`，这个谓词条件是无法被下推到JOIN下面的，但是可以通过提取 `n_name = '[NATION1]' OR n_name = '[NATION2]'`这个条件 并加到JOIN两端的输入下，可以减小输入的大小。<u>注意：不能算是谓词下推，因为JOIN完后还要再筛选一次，避免两边同时是NATION1或NATION2的情况。</u>

     实验表明，对Q7有大幅度提升，Q19有小幅度提升。

   - **物理访问的局部性**

     无理访问在 **l_shipdate**属性上进行过滤，*orders*表在有5个查询在 **o_orderdate**属性上的过滤。所以如果能基于这两个列建聚簇索引可以减少很大的扫描基数。

     实验的baseline是tpch生成的原始表；(1)中对*lineitem*表和*orders*表进行了完全的shuffle；(2)中对两张表分别按上述属性聚簇排列，但DBMS无法感知到，不能提前对扫描剪枝；(3)中DBMS可以感知到两张表的聚簇排列。结果图表如下所示：

     ![](https://cchw-1257198376.cos.ap-chengdu.myqcloud.com/test/clipboard_20201111_105329.png)

     注意，如[Q18](#Q18)中，对**l_orderkey**的group by操作会对物理局部性产生破坏，所以结果更差了。并且即使不能感知到聚簇的排列，也能比baseline提升性能，是因为 <u>CPU cache局部性更好 且 分支预测错误更少</u>。

   - **相关列**

     比如 *lineitem*表中**l_shipdate**（运送日期） 和 **l_receiptdate**（收货日期）总是相差在30天内，因为有语义上的关联。当然，<u>TPC-H的规范中声明 禁止显式地告知这些信息，但允许DBMS自身去发现这些关联</u>。这样的话，系统可以将一列的剪枝信息 传递给相关联的另一列。

     实验结果表明，允许利用列的相关联性，对[Q10](#Q10)和[Q12](#Q12)有小幅度提升。

     另外，在现实场景中，一般订单的创建也是随时间顺序递增，所以如果主键也是递增的话，这个物理局部性也能被很好地利用。

     *Pushing data-induced predicates through joins in big-data clusters (2019)*中指出可以在join中利用列的关联性，减少访问的数据量。

   - **Flattening Subqueries**

     [Q2](#Q2), [4](#Q4), [17](#Q17), [20](#Q20), [21](#Q21), [22](#Q22) 这六个用到了子查询。

     > **常见情况**
     >
     > - A in (subquery)
     >
     > - A not in (subquery) 
     >
     > - A = (subquery) 
     >
     > - NOT exists (subquery) 
     >
     > - exists (subquery)
     >
     > **Subquery Flattening & Unnesting**
     >
     > - Subquery **unnesting** is always done for correlated subqueries with at most one table in the FROM clause, which are used in ANY, ALL and EXISTS.
     > - An uncorrelated subquery, or a subquery with more than one table in the FROM clause, is **flattened** if it can be decided that the subquery returns at most one row.
     >
     > **关联与非关联**
     >
     > - **关联子查询**：对于外部查询返回的每一行数据，内部查询都要执行一次。另外，关联子查询的信息流是双向的，外部查询的每行数据传递一个值给子查询，然后子查询为每一行数据执行一次并返回它的记录，之后外部查询根据返回的记录做出决策。
     >
     >   即<u>先执行外层查询，再执行内层查询</u>。
     >
     > - **关联子查询**：独立于外部查询的子查询，子查询执行完毕后将值传递给外部查询
     >
     >   即<u>先执行内层查询，再执行外层查询</u>。

     对于其他工作中提出的子查询优化法，如*Enhanced subquery optimiza-tions in Oracle(2009)* 中提出的 不同子查询类型合并 可以在Q21中应用；*Integration*
     *of VectorWise with Ingres(2011)* 可以在ALL 语句中使用，但在TPC-H中没有使用场景。

     实验表明，Q2, 17, 20 可以通过 flatten相关联标量子查询 来提升效率，Q4, 21, 22 通过 相关联的EXISTS子查询 来提升效率。

   - **Semi Join简化**

     > semi-join是指semi-join子查询。 当一张表在另一张表找到匹配的记录之后，半连接只返回第一张表中的记录。不管在右表中找到几条匹配的记录，左表也只会返回一条记录，且返回结果不包含右表。半连接通常使用 **IN** 或 **EXISTS** 作为连接条件（本身没有标准 SQL 语句来表示 SEMI JOIN）

     [Q17](#Q17)中，可以在聚合函数之前添加semi join，来提前去掉 不会在最后JOIN中被匹配的行。如下图所示，最后一步JOIN的输入大小减小了1k倍：

     ![](https://cchw-1257198376.cos.ap-chengdu.myqcloud.com/test/clipboard_20201111_120319.png)

     这种计划层面的优化 是可以和 执行引擎的优化 一起独立使用的(orthogonal)。

   - **子计划reuse**

     [Q2](#Q2), [11](#Q11), [15](#Q15), [17](#Q17), [21](#Q21) 中，比如Q15，在使用两个地方使用了**revenue**视图，，而创建视图的计算就占了80%的开销。而Q21在*lineitem*表上有两个几乎一样的子查询，如果展开的话，可以在join时 共用一个外层的*lineitem*实例。

   - **结果reuse**

     直接cache整个结果。这里是为了整个研究的完整性 才加入的理论分析。

4. 逻辑算子层面瓶颈：

   - **Dependent Group-By Keys**
   
     如果能通过主键、外键的关系来发现列之间的函数依赖，可以减少基于hash和sort的agg算子开销。可以适用于[Q3](#Q3), [10](#Q10), [18](#Q18)。
   
   - **大的IN语句**
   
     [Q12](#Q12), [16](#Q16), [19](#Q19), [22](#Q22)中使用了IN常量的语句，在不使用即时编译的DBMS中，通常由三种方式来处理IN表达式：
   
     1. 使用解释型的计算。可以处理任意复杂嵌套的表达式，虽然通用性和扩展性很好，但是需要两层循环，还会带来虚方法调用的额外代价
     2. 把IN的值拆成析取谓词，最后合并结果
     3. 使用semi join，先把IN的值做哈希，然后再用输入值去probe。但是有建表的代价
   
     下图展示了 随IN的列表长度增加，不同方法执行时间的变化：
   
     ![](https://cchw-1257198376.cos.ap-chengdu.myqcloud.com/test/clipboard_20201111_031059.png)



-----------

**TPC-H22个查询参考**：

## Q1. 

```sql
select
l_returnflag,
l_linestatus,
sum(l_quantity) as sum_qty,
sum(l_extendedprice) as sum_base_price,
sum(l_extendedprice*(1-l_discount)) as sum_disc_price,
sum(l_extendedprice*(1-l_discount)*(1+l_tax)) as sum_charge,
avg(l_quantity) as avg_qty,
avg(l_extendedprice) as avg_price,
          avg(l_discount) as avg_disc,
count(*) as count_order
from
lineitem
where
l_shipdate <= date '1998-12-01' - interval '[DELTA]' day (3)
group by
l_returnflag,
l_linestatus
order by
l_returnflag,
l_linestatus;
```

## Q2. 

```sql
SELECT
    s_acctbal,
    s_name,
    n_name,
    p_partkey,
    p_mfgr,
    s_address,
    s_phone,
    s_comment
FROM
    part,
    supplier,
    partsupp,
    nation,
    region
WHERE
    p_partkey = ps_partkey
AND s_suppkey = ps_suppkey
AND p_size = [ SIZE ]
AND p_type LIKE '%[TYPE]'
AND s_nationkey = n_nationkey
AND n_regionkey = r_regionkey
AND r_name = '[REGION]'
AND ps_supplycost = (
    SELECT
        MIN (ps_supplycost)
    FROM
        partsupp,
        supplier,
        nation,
        region
    WHERE
        p_partkey = ps_partkey
    AND s_suppkey = ps_suppkey
    AND s_nationkey = n_nationkey
    AND n_regionkey = r_regionkey
    AND r_name = '[REGION]'
)
ORDER BY
    s_acctbal DESC,
    n_name,
    s_name,
    p_partkey;
```

## Q3. 

```sql
 SELECT
    l_orderkey,
    SUM (
        l_extendedprice * (1 - l_discount)
    ) AS revenue,
    o_orderdate,
    o_shippriority
FROM
    customer,
    orders,
    lineitem
WHERE
    c_mktsegment = '[SEGMENT]'
AND c_custkey = o_custkey
AND l_orderkey = o_orderkey
AND o_orderdate < DATE '[DATE]'
AND l_shipdate > DATE '[DATE]'
GROUP BY
    l_orderkey,
    o_orderdate,
    o_shippriority
ORDER BY
    revenue DESC,
    o_orderdate;
```

## Q4. 

```sql
SELECT
    o_orderpriority,
    COUNT (*) AS order_count
FROM
    orders
WHERE
    o_orderdate >= DATE '[DATE]'
AND o_orderdate < DATE '[DATE]' + INTERVAL '3' MONTH
AND EXISTS (
    SELECT
        *
    FROM
        lineitem
    WHERE
        l_orderkey = o_orderkey
    AND l_commitdate < l_receiptdate
)
GROUP BY
    o_orderpriority
ORDER BY
    o_orderpriority;
```

## Q5. 

```sql
select
    n_name,
    sum(l_extendedprice * (1 - l_discount)) as revenue
from
    customer,
    orders,
    lineitem,
    supplier,
    nation,
    region
where
    c_custkey = o_custkey
    and l_orderkey = o_orderkey
    and l_suppkey = s_suppkey
    and c_nationkey = s_nationkey
    and s_nationkey = n_nationkey
    and n_regionkey = r_regionkey
    and r_name = ':1'
    and o_orderdate >= date ':2'
    and o_orderdate < date ':2' + interval '1' year
group by
    n_name
order by
    revenue desc;
```

## Q6. 

```sql
select
sum(l_extendedprice*l_discount) as revenue
from 
lineitem
where 
l_shipdate >= date '[DATE]'
and l_shipdate < date '[DATE]' + interval '1' year
and l_discount between [DISCOUNT] - 0.01 and [DISCOUNT] + 0.01
and l_quantity < [QUANTITY];
```

## Q7. 

```sql
select
supp_nation, 
cust_nation, 
l_year, sum(volume) as revenue
from (
select 
n1.n_name as supp_nation, 
n2.n_name as cust_nation, 
extract(year from l_shipdate) as l_year,
l_extendedprice * (1 - l_discount) as volume
from 
supplier, 
lineitem, 
orders, 
customer, 
nation n1, 
nation n2
where 
s_suppkey = l_suppkey
and o_orderkey = l_orderkey
and c_custkey = o_custkey
and s_nationkey = n1.n_nationkey
and c_nationkey = n2.n_nationkey
and (
(n1.n_name = '[NATION1]' and n2.n_name = '[NATION2]')
or (n1.n_name = '[NATION2]' and n2.n_name = '[NATION1]')
)
and l_shipdate between date '1995-01-01' and date '1996-12-31'
) as shipping
group by 
supp_nation, 
cust_nation, 
l_year
order by 
supp_nation, 
cust_nation, 
l_year;
```

## Q8. 

```sql
select
o_year, 
sum(case 
when nation = '[NATION]' 
then volume
else 0
end) / sum(volume) as mkt_share
from (
select 
extract(year from o_orderdate) as o_year,
l_extendedprice * (1-l_discount) as volume, 
n2.n_name as nation
from 
part, 
supplier, 
lineitem, 
orders, 
customer, 
nation n1, 
nation n2, 
region
where 
p_partkey = l_partkey
and s_suppkey = l_suppkey
and l_orderkey = o_orderkey
and o_custkey = c_custkey
and c_nationkey = n1.n_nationkey
and n1.n_regionkey = r_regionkey
and r_name = '[REGION]'
and s_nationkey = n2.n_nationkey
and o_orderdate between date '1995-01-01' and date '1996-12-31'
and p_type = '[TYPE]' 
) as all_nations
group by 
o_year
order by 
o_year;
```

## Q9. 

```sql
select 
nation, 
o_year, 
sum(amount) as sum_profit
from (
select 
n_name as nation, 
extract(year from o_orderdate) as o_year,
l_extendedprice * (1 - l_discount) - ps_supplycost * l_quantity as amount
from 
part, 
supplier, 
lineitem, 
partsupp, 
orders, 
nation
where 
s_suppkey = l_suppkey
and ps_suppkey = l_suppkey
and ps_partkey = l_partkey
and p_partkey = l_partkey
and o_orderkey = l_orderkey
and s_nationkey = n_nationkey
and p_name like '%[COLOR]%'
) as profit
group by 
nation, 
o_year
order by 
nation, 
o_year desc;
```

## Q10. 

```sql
 select
c_custkey, 
c_name, 
sum(l_extendedprice * (1 - l_discount)) as revenue,
c_acctbal, 
n_name, 
c_address, 
c_phone, 
c_comment
from 
customer, 
orders, 
lineitem, 
nation
where 
c_custkey = o_custkey
and l_orderkey = o_orderkey
and o_orderdate >= date '[DATE]'
and o_orderdate < date '[DATE]' + interval '3' month
and l_returnflag = 'R'
and c_nationkey = n_nationkey
group by 
c_custkey, 
c_name, 
c_acctbal, 
c_phone, 
n_name, 
c_address, 
c_comment
order by 
revenue desc;
```

## Q11. 

```sql
select
ps_partkey, 
sum(ps_supplycost * ps_availqty) as value
from 
partsupp, 
supplier, 
nation
where 
ps_suppkey = s_suppkey
and s_nationkey = n_nationkey
and n_name = '[NATION]'
group by 
ps_partkey having 
sum(ps_supplycost * ps_availqty) > (
select 
sum(ps_supplycost * ps_availqty) * [FRACTION]
from 
partsupp, 
supplier, 
nation
where 
ps_suppkey = s_suppkey
and s_nationkey = n_nationkey
and n_name = '[NATION]'
)
order by
value desc;
```

## Q12. 

```sql
select
l_shipmode, 
sum(case 
when o_orderpriority ='1-URGENT'
or o_orderpriority ='2-HIGH'
then 1
else 0
end) as high_line_count,
sum(case 
when o_orderpriority <> '1-URGENT'
and o_orderpriority <> '2-HIGH'
then 1
else 0
end) as low_line_count
from 
orders, 
lineitem
where 
o_orderkey = l_orderkey
and l_shipmode in ('[SHIPMODE1]', '[SHIPMODE2]')
and l_commitdate < l_receiptdate
and l_shipdate < l_commitdate
and l_receiptdate >= date '[DATE]'
and l_receiptdate < date '[DATE]' + interval '1' year
group by 
l_shipmode
order by 
l_shipmode;
```

## Q13.

```sql
select 
c_count, count(*) as custdist 
from (
select 
c_custkey,
count(o_orderkey) 
from 
customer left outer join orders on 
c_custkey = o_custkey
and o_comment not like ‘%[WORD1]%[WORD2]%’
group by 
c_custkey
)as c_orders (c_custkey, c_count)
group by 
c_count
order by 
custdist desc, 
c_count desc;
```

## Q14. 

```sql
select
100.00 * sum(case 
when p_type like 'PROMO%'
then l_extendedprice*(1-l_discount)
else 0
end) / sum(l_extendedprice * (1 - l_discount)) as promo_revenue
from 
lineitem, 
part
where 
l_partkey = p_partkey
and l_shipdate >= date '[DATE]'
and l_shipdate < date '[DATE]' + interval '1' month;
```

## Q15. 

```sql
create view revenue[STREAM_ID] (supplier_no, total_revenue) as
select 
l_suppkey, 
sum(l_extendedprice * (1 - l_discount))
from 
lineitem
where 
l_shipdate >= date '[DATE]'
and l_shipdate < date '[DATE]' + interval '3' month
group by 
l_suppkey;
select
s_suppkey, 
s_name, 
s_address, 
s_phone, 
total_revenue
from 
supplier, 
revenue[STREAM_ID]
where 
s_suppkey = supplier_no
and total_revenue = (
select 
max(total_revenue)
from 
revenue[STREAM_ID]
)
order by 
s_suppkey;
drop view revenue[STREAM_ID];
```

## Q16. 

```sql
select
p_brand, 
p_type, 
p_size, 
count(distinct ps_suppkey) as supplier_cnt
from 
partsupp, 
part
where 
p_partkey = ps_partkey
and p_brand <> '[BRAND]'
and p_type not like '[TYPE]%'
and p_size in ([SIZE1], [SIZE2], [SIZE3], [SIZE4], [SIZE5], [SIZE6], [SIZE7], [SIZE8])
and ps_suppkey not in (
select 
s_suppkey
from 
supplier
where 
s_comment like '%Customer%Complaints%'
)
group by 
p_brand, 
p_type, 
p_size
order by 
supplier_cnt desc, 
p_brand, 
p_type, 
p_size;
```

## Q17. 

```sql
select
sum(l_extendedprice) / 7.0 as avg_yearly
from 
lineitem, 
part
where 
p_partkey = l_partkey
and p_brand = '[BRAND]'
and p_container = '[CONTAINER]'
and l_quantity < (
select
0.2 * avg(l_quantity)
from 
lineitem
where 
l_partkey = p_partkey
);
```

## Q18. 

```sql
select 
c_name,
c_custkey, 
o_orderkey,
o_orderdate,
o_totalprice,
sum(l_quantity)
from 
customer,
orders,
lineitem
where 
o_orderkey in (
select
l_orderkey
from
lineitem
group by 
l_orderkey having 
sum(l_quantity) > [QUANTITY]
)
and c_custkey = o_custkey
and o_orderkey = l_orderkey
group by 
c_name, 
c_custkey, 
o_orderkey, 
o_orderdate, 
o_totalprice
order by 
o_totalprice desc,
o_orderdate;
```

## Q19. 

```sql
select
sum(l_extendedprice * (1 - l_discount) ) as revenue
from 
lineitem, 
part
where 
(
p_partkey = l_partkey
and p_brand = ‘[BRAND1]’
and p_container in ( ‘SM CASE’, ‘SM BOX’, ‘SM PACK’, ‘SM PKG’) 
and l_quantity >= [QUANTITY1] and l_quantity <= [QUANTITY1] + 10 
and p_size between 1 and 5 
and l_shipmode in (‘AIR’, ‘AIR REG’)
and l_shipinstruct = ‘DELIVER IN PERSON’ 
)
or 
(
p_partkey = l_partkey
and p_brand = ‘[BRAND2]’
and p_container in (‘MED BAG’, ‘MED BOX’, ‘MED PKG’, ‘MED PACK’)
and l_quantity >= [QUANTITY2] and l_quantity <= [QUANTITY2] + 10
and p_size between 1 and 10
and l_shipmode in (‘AIR’, ‘AIR REG’)
and l_shipinstruct = ‘DELIVER IN PERSON’
)
or 
(
p_partkey = l_partkey
and p_brand = ‘[BRAND3]’
and p_container in ( ‘LG CASE’, ‘LG BOX’, ‘LG PACK’, ‘LG PKG’)
and l_quantity >= [QUANTITY3] and l_quantity <= [QUANTITY3] + 10
and p_size between 1 and 15
and l_shipmode in (‘AIR’, ‘AIR REG’)
and l_shipinstruct = ‘DELIVER IN PERSON’
);
```

## Q20. 

```sql
select 
s_name, 
s_address
from 
supplier, nation
where 
s_suppkey in (
select 
ps_suppkey
from 
partsupp
where 
ps_partkey in (
select 
p_partkey
from 
part
where 
p_name like '[COLOR]%'
)
and ps_availqty > (
select 
0.5 * sum(l_quantity)
from 
lineitem
where 
l_partkey = ps_partkey
and l_suppkey = ps_suppkey
and l_shipdate >= date('[DATE]’)
and l_shipdate < date('[DATE]’) + interval ‘1’ year 
)
)
and s_nationkey = n_nationkey
and n_name = '[NATION]'
order by 
s_name;
```

## Q21. 

```sql
select 
s_name, 
count(*) as numwait
from 
supplier, 
lineitem l1, 
orders, 
nation
where 
s_suppkey = l1.l_suppkey
and o_orderkey = l1.l_orderkey
and o_orderstatus = 'F'
and l1.l_receiptdate > l1.l_commitdate
and exists ( 
select 
*
from 
lineitem l2
where 
l2.l_orderkey = l1.l_orderkey
and l2.l_suppkey <> l1.l_suppkey
)
and not exists ( 
select 
*
from 
lineitem l3
where 
l3.l_orderkey = l1.l_orderkey
and l3.l_suppkey <> l1.l_suppkey
and l3.l_receiptdate > l3.l_commitdate
)
and s_nationkey = n_nationkey
and n_name = '[NATION]'
group by 
s_name
order by 
numwait desc, 
s_name;
```

## Q22. 

```sql
select 
cntrycode, 
count(*) as numcust, 
sum(c_acctbal) as totacctbal
from (
select 
substring(c_phone from 1 for 2) as cntrycode, 
c_acctbal
from 
customer
where 
substring(c_phone from 1 for 2) in 
('[I1]','[I2]','[I3]','[I4]','[I5]','[I6]','[I7]')
and c_acctbal > (
select 
avg(c_acctbal)
from 
customer
where 
c_acctbal > 0.00
and substring (c_phone from 1 for 2) in
('[I1]','[I2]','[I3]','[I4]','[I5]','[I6]','[I7]')
)
and not exists (
select 
* 
from 
orders
where 
o_custkey = c_custkey
)
) as custsale
group by 
cntrycode 
order by 
cntrycode;
```

