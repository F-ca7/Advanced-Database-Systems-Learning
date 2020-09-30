## Benchmark

1. TPC-C:

   OLTP的基准测试，也就是关注事务处理性能这一块，其性能由吞吐率衡量，单位是 tpm (transactions per minute)。

   模拟了一个比较复杂并具有代表意义的OLTP应用环境:假设有一个大型商品批发商，它拥有若干个分布在不同区域的商品库；每个仓库负责为10个销售点供货；每个销售点为3000个客户提供服务；每个客户平均一个订单有10项产品;所有订单中约1%的产品在其直接所属的仓库中没有存货，需要由其他区域的仓库来供货。同时，每个仓库都要维护公司销售的十万种商品的库存记录。

   | 事务类型     | 解释                             | 事务混合比 |
   | ------------ | -------------------------------- | ---------- |
   | New-Order    | 客户输入一笔新的订货交易         | 最多45%    |
   | Payment      | 更新客户账户余额以反映其支付状况 | 最少43%    |
   | Delivery     | 发货(模拟批处理交易)             | 最少4%     |
   | Order-Status | 查询客户最近交易的状态           | 最少4%     |
   | Stock-Level  | 查询仓库库存状况                 | 最少4%     |

   五种事务类型分别如下：

   - NewOrder: 是TPCC测试中的核心事务，特点是读写混合、高频发生并需要保证响应时间满足用户在线实时下单。并且为了模拟真实应用中，会有1%的NewOrder事务发生回退，代表订单的取消。

   - Payment: Payment事务代表客户对订单付款行为，并且需要更新客户账户余额，以及将交易记录更新到地区和仓库的销售统计信息中，特点是读写混合、高频发生且同样需要保证响应时间满足在线交易。

   - Delivery: 代表批量配送订单，对响应时间的要求较为宽松。

   - OrderStatus: 即模拟客户去查询自己最近一次订单的交易状态，为只读查询。

   - StockLevel: 查看系统的库存状态，该事务频率较低，且对响应时间要求不高。

2. [TPC-H](http://www.tpc.org/tpc_documents_current_versions/pdf/tpc-h_v2.17.3.pdf):

   OLAP的基准测试，也就是关注分析处理性能这一块，包括 22 个查询(Q1~Q22)，其主要评价指标是各个查询的响应时间,即从提交查询到结果返回所需时间。也可用 一个小时内能够完成的复杂交易数量 来衡量。

   其schema如下

   ![](https://cchw-1257198376.cos.ap-chengdu.myqcloud.com/test/clipboard_20200920085426.png)

   几个查询的示例

   ```sql
   Q1:
   # 查询lineItems的一个定价总结报告。在单个表lineitem上查询某个时间段内，对已经付款的、已经运送的等各类商品进行统计，包括业务量的计费、发货、折扣、税、平均价格等信息
   select 
       l_returnflag, //返回标志
       l_linestatus, 
       sum(l_quantity) as sum_qty, //总的数量
       sum(l_extendedprice) as sum_base_price, //聚集函数操作
       sum(l_extendedprice * (1 - l_discount)) as sum_disc_price, 
       sum(l_extendedprice * (1 - l_discount) * (1 + l_tax)) as sum_charge, 
       avg(l_quantity) as avg_qty, 
       avg(l_extendedprice) as avg_price, 
       avg(l_discount) as avg_disc, 
       count(*) as count_order //每个分组所包含的行数
   from 
       lineitem
   where 
       l_shipdate <= date'1998-12-01' - interval '90' day //时间段是随机生成的
   group by //分组操作
       l_returnflag, 
       l_linestatus
   order by //排序操作
       l_returnflag, 
       l_linestatus;
   ```

   ```sql
   Q2：
   获得最小代价的供货商。得到给定的区域内，对于指定的零件（某一类型和大小的零件），哪个供应者能以最低的价格供应它，就可以选择哪个供应者来订货
   select
       s_acctbal, s_name, n_name, p_partkey, p_mfgr, s_address, s_phone, s_comment /*查询供应者的帐户余额、名字、国家、零件的号码、生产者、供应者的地址、电话号码、备注信息 */
   from
       part, supplier, partsupp, nation, region //五表连接
   where
       p_partkey = ps_partkey
       and s_suppkey = ps_suppkey
       and p_size = [SIZE] //指定大小，在区间[1, 50]内随机选择
       and p_type like '%[TYPE]' //指定类型，在TPC-H标准指定的范围内随机选择
       and s_nationkey = n_nationkey
       and n_regionkey = r_regionkey
       and r_name = '[REGION]' //指定地区，在TPC-H标准指定的范围内随机选择
       and ps_supplycost = ( //子查询
           select
               min(ps_supplycost) //聚集函数
           from
               partsupp, supplier, nation, region //与父查询的表有重叠
           where
               p_partkey = ps_partkey
               and s_suppkey = ps_suppkey
               and s_nationkey = n_nationkey
               and n_regionkey = r_regionkey
               and r_name = '[REGION]'
       )
   order by //排序
       s_acctbal desc,
       n_name,
       s_name,
       p_partkey;
   ```

   

3. 