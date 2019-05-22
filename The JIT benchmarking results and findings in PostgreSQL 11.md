## The JIT benchmarking results and findings in PostgreSQL 11

Just-in-time compilation of SQL statements is one of the new features in PostgreSQL 11. Just-in-Time compilation (JIT) is the process of turning some form of interpreted program evaluation into a native program and doing so at runtime. In the JIT testing, PostgreSQL 11 is about 29.31% faster at executing the TPC-H Q1 query than PostgreSQL 10. You could get details from https://www.citusdata.com/blog/2018/09/11/postgresql-11-just-in-time/.

As far as I know, the JIT technique is implemented using llvm.

#### What is the LLVM?
>The LLVM Project is a collection of modular and reusable compiler and toolchain technologies. Despite its name, LLVM has little to do with traditional virtual machines. The name "LLVM" itself is not an acronym; it is the full name of the project.

#### Is the JIT necessary?
In today's enterprise, the data has been growing at an unprecedented rate, with the sharp rise in data the requirement for performing business intelligence and analytical queries is also increasing. Most enterprises are also relying upon to store there very large historical data in NoSQL or distributed storage like Hadoop for performing complex analytical queries. Having said that the OLAP capabilities of the database are also getting there run for the money, the OLAP workloads are increasing and hence the need to enhance database OLAP functionality to handle the growing OLAP workloads. OLAP functionality is primarily affected by data throughput and CPU computing power. Now with the rise of SSD, Column Storage, and Distributed Databases, I/O is not the main bottleneck of the database. Modern databases add a lot of logical and virtual function calls in response to various scenarios. This reduces the efficiency of OLAP capabilities for the database. In the PostgreSQL 11, they added the Just-in-Time compilation which could reduce the redundant logical operations and virtual function calls.

#### JIT Accelerated Operations in PostgreSQL 11
>Currently PostgreSQL's JIT implementation has support for accelerating expression evaluation and tuple deforming. Several other operations could be accelerated in the future.

>Expression evaluation is used to evaluate WHERE clauses, target lists, aggregates and projections. It can be accelerated by generating code specific to each case.

>Tuple deforming is the process of transforming an on-disk tuple (see Section 68.6.1) into its in-memory representation. It can be accelerated by creating a function specific to the table layout and the number of columns to be extracted.

The first time I came into contact with this technology was using the Vitesse database. In recent years, some new database products with this technology have been discovered.

Vitesse DB:
* Proprietary database based on PostgreSQL 9.5.x
* JIT compiling expressions as well as execution plan
* Speedup up to 8x on TPC-H Q1


Butterstein D., Grust T., Precision Performance Surgery for PostgreSQL –
LLVM-based Expression Compilation, Just in Time. VLDB 2016.
* JIT compiling expressions for Filter and Aggregation
* Speedup up to 37% on TPC-H


New expression interpreter (by Andres Freund, PostgreSQL 11):
* Changes tree walker based interpreter to more effective one
* Also adds LLVM JIT for expressions
* Speedup up to 30% on TPC-H

I find that the JIT is disabled by default in PostgreSQL 11. And compared with the other two database products, PostgreSQL 11 has a lower performance improvement. In my opinion, this is a major improvement, but it seems to tell me not. I want to get the answer. So in this blog, I will comparative benchmarking with JIT with PostgreSQL 11 and PostgreSQL 10, and report the benchmarking results and findings.

#### My Hardware Configuration：
* CPU : i5 8400
* Memory : 16G DDR4
* Disk : Inter SSD 1T
* OS : CentOS Linux release 7.6.1810 (Core)

#### Install the PostgreSQL 11 from the source with llvm
PostgreSQL has built-in support to perform JIT compilation using LLVM when PostgreSQL is built with --with-llvm. In the first, you need to install llvm5.0. Then you can install the PostgreSQL 11 and PostgreSQL 10.
```
./configure --prefix=/home/postgres/pg11JIT --with-openssl --with-libxml --with-libxslt --with-llvm
make -j6; make install
cd contrib
make -j6; make install

./configure --prefix=/home/postgres/pgdb10 --with-openssl --with-libxml --with-libxslt
make -j6; make install
cd contrib
make -j6; make install
```
#### The configuration variable about JIT
>The configuration variable jit determines whether JIT compilation is enabled or disabled. If it is enabled, the configuration variables jit_above_cost, jit_inline_above_cost, and jit_optimize_above_cost determine whether JITcompilation is performed for a query, and how much effort is spent doing so.

>jit_provider determines which JIT implementation is used. It is rarely required to be changed.

#### Enabling The JIT in PostgreSQL 11
If you install the PostgreSQL 11 suscessfully, you will find a new folder named bit_code. There is a lot of file with .bc extension. These are pre-generated bytecodes for LLVM for facilitating features like in-lining.

I changed some configuration variable in PostgreSQL10 and PostgreSQL11:
```
hared_buffers = 4GB
temp_buffers = 64MB
work_mem = 32MB
maintenance_work_mem = 1GB
wal_level = minimal
max_wal_senders = 0
max_wal_size = 40GB
random_page_cost = 1.1
jit = on
```
Let's try the JIT(For a simple demonstration, I will take these parameters as 0;):
```
[postgres@neoc bin]$ ./psql
psql (11.2)
Type "help" for help.

postgres=# set jit=on;
SET
postgres=# set jit_above_cost=0;
SET
postgres=# set jit_inline_above_cost=0;
SET
postgres=# set jit_optimize_above_cost=0;
SET
postgres=# explain (analyze,verbose,costs,buffers,summary) select count(*) from pg_class;
                                                                       QUERY PLAN

-------------------------------------------------------------------------------------------------------------------------------------------------
-------
 Aggregate  (cost=14.73..14.74 rows=1 width=8) (actual time=180.390..180.391 rows=1 loops=1)
   Output: count(*)
   Buffers: shared hit=22 read=1 dirtied=1
   ->  Index Only Scan using pg_class_oid_index on pg_catalog.pg_class  (cost=0.27..13.76 rows=386 width=0) (actual time=0.393..0.618 rows=386 lo
ops=1)
         Output: oid
         Heap Fetches: 183
         Buffers: shared hit=22 read=1 dirtied=1
 Planning Time: 2.060 ms
 JIT:
   Functions: 2
   Options: Inlining true, Optimization true, Expressions true, Deforming true
   Timing: Generation 1.852 ms, Inlining 67.384 ms, Optimization 55.645 ms, Emission 56.331 ms, Total 181.213 ms
 Execution Time: 330.002 ms
(13 rows)
```

I used the tool pg_tpch to get the benchmarking results. For detailed usage, please refer to the URL:

We could get the results like this(unit is second, JIT disabled in PG11 and JIT enabled in PG11):
![PG112](/assets/PG112_atz0727xa.png)
The total is the sum of all 22 SQL execution times.
To be more intuitive, we can look at the histogram:
![pg11h](/assets/pg11h.png)
We will find that the execution time is reduced by turning on the JIT, about 6.1% down.
With the JIT turned on, performance has dropped. It may be that my test case and hardware do not meet the test requirements.
So I checked the explain. I found a interesting thing that the JIT disabled used less time than JIT enabled in the Nested Loop.
JIT enabled in PG11:
```
----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
 Limit (actual time=72490.850..72624.050 rows=1 loops=1)
   ->  Finalize GroupAggregate (actual time=71891.664..71891.664 rows=1 loops=1)
         ->  Gather Merge (actual time=71891.649..72024.844 rows=4 loops=1)
               ->  Sort (actual time=71799.894..71799.896 rows=53 loops=3)
                     ->  Partial HashAggregate (actual time=71799.269..71799.684 rows=175 loops=3)
                           ->  Hash Join (actual time=69631.747..71189.348 rows=1087902 loops=3)
                                 ->  Parallel Hash Join  (actual time=69310.166..70696.067 rows=1087902 loops=3)
                                       ->  Parallel Seq Scan on public.orders (actual time=0.504..9327.128 rows=5000000 loops=3)
                                       ->  Parallel Hash (actual time=57497.965..57497.965 rows=1087902 loops=3)
                                             ->  Nested Loop (actual time=45.005..56802.904 rows=1087902 loops=3)
                                                   ->  Parallel Hash Join  (actual time=44.034..5326.659 rows=145140 loops=3)
                                                         ->  Nested Loop (actual time=0.471..5146.321 rows=145140 loops=3)
                                                               ->  Parallel Seq Scan on public.part (actual time=0.030..353.243 rows=36285 loops=3)
                                                               ->  Index Scan using idx_partsupp_partkey on public.partsupp (actual time=0.122..0.130 rows=4 loops=108855)
                                                         ->  Parallel Hash (actual time=43.378..43.378 rows=33333 loops=3)
                                                               ->  Parallel Seq Scan on public.supplier (actual time=0.016..96.976 rows=100000 loops=1)
                                                   ->  Index Scan using idx_lineitem_part_supp on public.lineitem (actual time=0.086..0.351 rows=7 loops=435420)
                                 ->  Hash (actual time=321.382..321.382 rows=25 loops=3)
                                       ->  Seq Scan on public.nation (actual time=321.368..321.371 rows=25 loops=3)
 Planning Time: 30.108 ms
 JIT:
   Functions: 182
   Options: Inlining true, Optimization true, Expressions true, Deforming true
   Timing: Generation 59.807 ms, Inlining 154.474 ms, Optimization 866.697 ms, Emission 539.179 ms, Total 1620.158 ms
 Execution Time: 72662.415 ms
(142 rows)
```
JIT disabled In PG11:

```
----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
 Limit (actual time=53927.711..54069.684 rows=1 loops=1)
   ->  Finalize GroupAggregate (actual time=53927.710..53927.710 rows=1 loops=1)
         ->  Gather Merge (actual time=53927.698..54069.665 rows=4 loops=1)
               ->  Sort (actual time=53885.863..53885.865 rows=54 loops=3)
                     ->  Partial HashAggregate (actual time=53884.461..53884.900 rows=175 loops=3)
                           ->  Hash Join (actual time=51626.569..53258.211 rows=1087902 loops=3)
                                 ->  Parallel Hash Join (actual time=51624.992..53052.825 rows=1087902 loops=3)
                                       ->  Parallel Seq Scan on public.orders (actual time=0.195..9286.259 rows=5000000 loops=3)
                                       ->  Parallel Hash (actual time=40238.570..40238.570 rows=1087902 loops=3)
                                             ->  Nested Loop (actual time=68.942..39704.018 rows=1087902 loops=3)
                                                   ->  Parallel Hash Join (actual time=68.193..5794.431 rows=145140 loops=3)
                                                         ->  Nested Loop (actual time=1.703..5596.634 rows=145140 loops=3)
                                                               ->  Parallel Seq Scan on public.part (actual time=0.209..289.795 rows=36285 loops=3)
                                                               ->  Index Scan using idx_partsupp_partkey on public.partsupp (actual time=0.136..0.144 rows=4 loops=108855)
                                                         ->  Parallel Hash (actual time=66.231..66.231 rows=33333 loops=3)
                                                               ->  Parallel Seq Scan on public.supplier (actual time=0.022..57.748 rows=33333 loops=3)
                                                   ->  Index Scan using idx_lineitem_part_supp on public.lineitem (actual time=0.069..0.231 rows=7 loops=435420)
                                 ->  Hash (actual time=0.038..0.038 rows=25 loops=3)
                                       ->  Seq Scan on public.nation (actual time=0.020..0.023 rows=25 loops=3)
 Planning Time: 16.777 ms
 Execution Time: 54071.434 ms
(144 rows)
```
So I tried to turn the enable_nestloop off. I got the results:
JIT enabled in PG11:
```
----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
 Limit  (cost=2417565.92..2417566.19 rows=1 width=66) (actual time=26286.402..28662.321 rows=1 loops=1)
   ->  Finalize GroupAggregate  (cost=2417565.92..2433857.53 rows=60150 width=66) (actual time=25747.698..25747.698 rows=1 loops=1)
         ->  Gather Merge  (cost=2417565.92..2431601.90 rows=120300 width=66) (actual time=25747.682..28123.594 rows=4 loops=1)
               ->  Sort  (cost=2416565.90..2416716.27 rows=60150 width=66) (actual time=25728.040..25728.045 rows=117 loops=3)
                     ->  Partial HashAggregate  (cost=2410738.48..2411791.11 rows=60150 width=66) (actual time=25727.454..25727.844 rows=175 loops=3)
                           ->  Hash Join  (cost=1936728.06..2388650.49 rows=1262171 width=57) (actual time=8919.348..25117.729 rows=1087902 loops=3)
                                 ->  Parallel Hash Join  (cost=1936726.50..2378463.21 rows=1262171 width=35) (actual time=8555.334..24580.928 rows=1087902 loops=3)
                                       ->  Parallel Seq Scan on public.orders  (cost=0.00..338723.33 rows=6250133 width=8) (actual time=0.034..567.581 rows=5000000 loops=3)
                                       ->  Parallel Hash  (cost=1911088.36..1911088.36 rows=1262171 width=39) (actual time=7466.736..7466.736 rows=1087902 loops=3)
                                             ->  Parallel Hash Join  (cost=1616063.36..1911088.36 rows=1262171 width=39) (actual time=6438.537..7258.071 rows=1087902 loops=3)
                                                   ->  Parallel Hash Join  (cost=1612441.83..1904153.48 rows=1262171 width=47) (actual time=6431.432..7101.985 rows=1087902 loops=3)
                                                         ->  Parallel Seq Scan on public.partsupp  (cost=0.00..216566.52 rows=3332652 width=22) (actual time=0.085..337.680 rows=2666667 loops=3)
                                                         ->  Parallel Hash  (cost=1582415.26..1582415.26 rows=1262171 width=45) (actual time=5735.438..5735.438 rows=1087902 loops=3)
                                                               ->  Parallel Hash Join  (cost=51923.86..1582415.26 rows=1262171 width=45) (actual time=87.559..5486.450 rows=1087902 loops=3)
                                                                     ->  Parallel Seq Scan on public.lineitem  (cost=0.00..1464889.92 rows=24990992 width=41) (actual time=0.019..3087.888 rows=19995351 loops=3)
                                                                     ->  Parallel Hash  (cost=51397.76..51397.76 rows=42088 width=4) (actual time=87.245..87.246 rows=36285 loops=3)
                                                                           ->  Parallel Seq Scan on public.part  (cost=0.00..51397.76 rows=42088 width=4) (actual time=0.016..243.671 rows=108855 loops=1)
                                                   ->  Parallel Hash  (cost=2886.24..2886.24 rows=58824 width=12) (actual time=6.928..6.928 rows=33333 loops=3)
                                                         ->  Parallel Seq Scan on public.supplier  (cost=0.00..2886.24 rows=58824 width=12) (actual time=0.012..8.941 rows=100000 loops=1)
                                 ->  Hash  (cost=1.25..1.25 rows=25 width=30) (actual time=363.979..363.979 rows=25 loops=3)
                                       ->  Seq Scan on public.nation  (cost=0.00..1.25 rows=25 width=30) (actual time=363.964..363.967 rows=25 loops=3)
 Planning Time: 2.046 ms
 JIT:
   Functions: 218
   Options: Inlining true, Optimization true, Expressions true, Deforming true
   Timing: Generation 16.778 ms, Inlining 76.056 ms, Optimization 950.728 ms, Emission 602.449 ms, Total 1646.010 ms
 Execution Time: 28669.215 ms
(152 rows)
```

JIT disabled in PG11:
```
 Limit  (cost=2417565.92..2417566.19 rows=1 width=66) (actual time=10543.751..10785.349 rows=1 loops=1)
   ->  Finalize GroupAggregate  (cost=2417565.92..2433857.53 rows=60150 width=66) (actual time=10543.751..10543.751 rows=1 loops=1)
         ->  Gather Merge  (cost=2417565.92..2431601.90 rows=120300 width=66) (actual time=10543.741..10785.333 rows=4 loops=1)
               ->  Sort  (cost=2416565.90..2416716.27 rows=60150 width=66) (actual time=10540.393..10540.396 rows=62 loops=3)
                     ->  Partial HashAggregate  (cost=2410738.48..2411791.11 rows=60150 width=66) (actual time=10539.846..10540.206 rows=175 loops=3)
                           ->  Hash Join  (cost=1936728.06..2388650.49 rows=1262171 width=57) (actual time=8494.973..9919.414 rows=1087902 loops=3)
                                 ->  Parallel Hash Join  (cost=1936726.50..2378463.21 rows=1262171 width=35) (actual time=8494.901..9720.725 rows=1087902 loops=3)
                                       ->  Parallel Seq Scan on public.orders  (cost=0.00..338723.33 rows=6250133 width=8) (actual time=0.021..634.898 rows=5000000 loops=3)
                                       ->  Parallel Hash  (cost=1911088.36..1911088.36 rows=1262171 width=39) (actual time=7348.100..7348.100 rows=1087902 loops=3)
                                             ->  Parallel Hash Join  (cost=1616063.36..1911088.36 rows=1262171 width=39) (actual time=6251.609..7135.988 rows=1087902 loops=3)
                                                   ->  Parallel Hash Join  (cost=1612441.83..1904153.48 rows=1262171 width=47) (actual time=6240.936..6948.291 rows=1087902 loops=3)
                                                         ->  Parallel Seq Scan on public.partsupp  (cost=0.00..216566.52 rows=3332652 width=22) (actual time=0.021..361.590 rows=2666667 loops=3)
                                                         ->  Parallel Hash  (cost=1582415.26..1582415.26 rows=1262171 width=45) (actual time=5524.176..5524.176 rows=1087902 loops=3)
                                                               ->  Parallel Hash Join  (cost=51923.86..1582415.26 rows=1262171 width=45) (actual time=120.604..5269.257 rows=1087902 loops=3)
                                                                     ->  Parallel Seq Scan on public.lineitem  (cost=0.00..1464889.92 rows=24990992 width=41) (actual time=0.023..2755.599 rows=19995351 loops=3)
                                                                     ->  Parallel Hash  (cost=51397.76..51397.76 rows=42088 width=4) (actual time=120.332..120.332 rows=36285 loops=3)
                                                                           ->  Parallel Seq Scan on public.part  (cost=0.00..51397.76 rows=42088 width=4) (actual time=0.013..114.142 rows=36285 loops=3)
                                                   ->  Parallel Hash  (cost=2886.24..2886.24 rows=58824 width=12) (actual time=10.445..10.445 rows=33333 loops=3)
                                                         ->  Parallel Seq Scan on public.supplier  (cost=0.00..2886.24 rows=58824 width=12) (actual time=0.010..5.438 rows=33333 loops=3)
                                 ->  Hash  (cost=1.25..1.25 rows=25 width=30) (actual time=0.026..0.026 rows=25 loops=3)
                                       ->  Seq Scan on public.nation  (cost=0.00..1.25 rows=25 width=30) (actual time=0.017..0.019 rows=25 loops=3)
 Planning Time: 2.665 ms
 Execution Time: 10785.906 ms
(152 rows)
```
If I turned the enable_nestloop off, time will be significantly reduced. I think the query optimizer has some unreasonable design. The query optimizer chose nestloop instead of others.

If you want to know more details about explains, please refer to https://github.com/wangguoke/blog/tree/master/results%20about%20tpch.

We could get the results like this(unit is second, JIT disabled in PG11 and PG10):
![1110noJIT](/assets/1110noJIT.png)

To be more intuitive, we can look at the histogram:
![1110noJITh](/assets/1110noJITh.png)
We could get that the performance of the PG11 is 41% higher than the PG10. This shows that PG11 has made a lot of improvements in performance. Please refer to the official documentation for detailed information.

We could get the results like this(unit is second, JIT enabled in PG11 and PG10):
![pg1110JIT](/assets/pg1110JIT.png)

To be more intuitive, we can look at the histogram:
![pg1110JITh](/assets/pg1110JITh.png)
We could get that the performance of the PG11 is 37.1% higher than the PG10.

#### overall summary of JIT performance:
This JIT depends on the situation. In the 22 SQL cases, performance has improved, and some have declined.
I think there are some issues in the query optimizer. In a meeting, someone talked to me about Oracle. They told me about the Adaptive Execution Plans technology of Oracle. Although Oracle is now losing on the cloud, his database technology is still very advanced. Its query optimizer can convert the nestloop to hashjoin or others. The underlying optimization technology of llvm is superior to humans. We can improve our query optimizer based on the information provided by LLVM. But this is a large project.

#### I have 3 suggestions for Improvement:
1. Add new parameters to find the right case for query optimizer.
2. We can add some special passes for PostgreSQL.
3. We can improve our query optimizer based on the information provided by LLVM.

### Reference
* https://www.postgresql.org/docs/11/runtime-config-developer.html
* https://www.postgresql.org/docs/9.6/parallel-query.html
* 《JIT-Compiling SQL Queries in PostgreSQL Using LLVM》
* 《The Architecture of Open Source Applications: LLVM》
