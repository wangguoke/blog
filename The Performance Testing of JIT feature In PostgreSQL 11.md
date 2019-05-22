## The Performance Testing of JIT feature In PostgreSQL 11

In today's enterprise, the data has been growing at an unprecedented rate, with the sharp rise in data the requirement for performing business intelligence and analytical queries is also increasing. Most enterprises are also relying upon to store there very large historical data in NoSQL or distributed storage like Hadoop for performing complex analytical queries. Having said that the OLAP capabilities of the database are also getting there run for the money, the OLAP workloads are increasing and hence the need to enhance database OLAP functionality to handle the growing OLAP workloads. OLAP functionality is primarily affected by data throughput and CPU computing power. Now with the rise of SSD, Column Storage, and Distributed Databases, I/O is not the main bottleneck of the database. Modern databases add a lot of logical and virtual function calls in response to various scenarios. A lot of redundant operations have been added to the original operation to ensure that enough scenes are handled. This reduces the efficiency of OLAP capabilities for the database. In the PostgreSQL 11, they added the Just-in-Time compilation which could reduce the redundant logical operations and virtual function calls.

JIT is considered as a major feature in PG 11 and is expected to give a significant rise in performance for specific workloads. It is important to note that JIT doesn't benefit every workload, it is known to give a significant performance to specific workloads especially the OLAP workloads.

It is, therefore, I ventured to perform testing and benchmarking of JIT with PG 11, my testing focusses on the following areas :

1. Perform testing of JIT feature and identify the performance difference with JIT one and off with PG 11.
2. Another purpose is to compare the performance of PG 11 with/without JIT enabled and compare it with the previous 10.* version.

### What is Just-in-Time compilation?
>Just-in-Time compilation (JIT) is the process of turning some form of interpreted program evaluation into a native program, and doing so at runtime.
>For example, instead of using a facility that can evaluate arbitrary SQL expressions to evaluate an SQL predicate like WHERE a.col = 3, it is possible to generate a function than can be natively executed by the CPU that just handles that expression, yielding a speedup.
>That this is done at query execution time, possibly even only in cases where the relevant task is done a number of times, makes it JIT, rather than ahead-of-time (AOT). Given the way JIT compilation is used in PostgreSQL, the lines between interpretation, AOT and JIT are somewhat blurry.
>Note that the interpreted program turned into a native program does not necessarily have to be a program in the classical sense. E.g. it is highly beneficial to JIT compile tuple deforming into a native function just handling a specific type of table, despite tuple deforming not commonly being understood as a "program".

### How to JIT?
>PostgreSQL, by default, uses LLVM to perform JIT. LLVM was chosen because it is developed by several large corporations and therefore unlikely to be discontinued, because it has a license compatible with PostgreSQL, and because its IR can be generated from C using the Clang compiler.

### What is the LLVM?
>The LLVM Project is a collection of modular and reusable compiler and toolchain technologies. Despite its name, LLVM has little to do with traditional virtual machines. The name "LLVM" itself is not an acronym; it is the full name of the project.
>LLVM began as a research project at the University of Illinois, with the goal of providing a modern, SSA-based compilation strategy capable of supporting both static and dynamic compilation of arbitrary programming languages. Since then, LLVM has grown to be an umbrella project consisting of a number of subprojects, many of which are being used in production by a wide variety of commercial and open source projects as well as being widely used in academic research. Code in the LLVM project is licensed under the "Apache 2.0 License with LLVM exceptions".

#### LLVM's implementation of the three-phase design
The following picture despites the three-phase design of llvm.
![Image [2]](/assets/Image%20[2].png)
In an LLVM-based compiler, a front end is responsible for parsing, validating and diagnosing errors in the input code, then translating the parsed code into LLVM IR (usually, but not always, by building an AST and then converting the AST to LLVM IR). This IR is optionally fed through a series of analysis and optimization passes which improve the code, then is sent into a code generator to produce native machine code.
#### The time about LLVM
![Image [3]](/assets/Image%20[3].png)
As mentioned earlier, LLVM IR can be efficiently (de)serialized to/from a binary format known as LLVM bitcode. Since LLVM IR is self-contained, and serialization
is a lossless process, we can do part of a compilation, save our progress to disk, then continue work at some point in the future. This feature provides a number of
interesting capabilities including support for link-time and install-time optimization, both of which delay code generation from "compile time".

#### The Pass about LLVM
![PassLinkage](/assets/PassLinkage.png)
>The library-based design of the LLVM optimizer allows our implementer to pick and choose both the order in which passes execute, and which ones make sense for the image processing domain: if everything is defined as a single big function, it doesn't make sense to waste time on inlining. If there are few pointers, alias analysis and memory optimization aren't worth bothering about. However, despite our best efforts, LLVM doesn't magically solve all optimization problems! Since the pass subsystem is modularized and the PassManager itself doesn't know anything about the internals of the passes, the implementer is free to implement their own language-specific passes to cover for deficiencies in the LLVM optimizer or to explicit language-specific optimization opportunities.

You can write a special pass for your program.

### How to open the JIT in PostgreSQL 11?
#### Build with --with-llvm
PostgreSQL has built-in support to perform JIT compilation using LLVM when PostgreSQL is built with --with-llvm.
```
./configure --prefix=/home/postgres/pg11JIT --with-openssl --with-libxml --with-libxslt --with-llvm
make -j6; make install
cd contrib
make -j6; make install
```

#### Turn on the JIT
>The configuration variable jit determines whether JIT compilation is enabled or disabled. If it is enabled, the configuration variables jit_above_cost, jit_inline_above_cost, and jit_optimize_above_cost determine whether JITcompilation is performed for a query, and how much effort is spent doing so.
```
set jit=on;
set jit_above_cost=0;
set jit_inline_above_cost=0;
set jit_optimize_above_cost=0;
```
jit_provider determines which JIT implementation is used. It is rarely required to be changed.

### How does the JIT work?
#### Reduce the redundant logical and the virtual function calls
We could use 'select 3<4;' as an example:
![3compare4](/assets/3compare4_9a1pmtntz.png)

#### How to verify this process?
There is a jit_dump_bitcode:
>Writes the generated LLVM IR out to the file system, inside data_directory. This is only useful for working on the internals of the JIT implementation. The default setting is off. This parameter can only be changed by a superuser.

You will get the bitcode about your query plan:
```
-rw-------. 1 postgres postgres 18732 May 14 18:03 30949.0.bc
-rw-------. 1 postgres postgres 15592 May 14 18:03 30949.0.optimized.bc
-rw-------. 1 postgres postgres 14332 May 14 18:03 30949.1.bc
-rw-------. 1 postgres postgres  9808 May 14 18:03 30949.1.optimized.bc
-rw-------. 1 postgres postgres 15472 May 14 18:04 30949.2.bc
-rw-------. 1 postgres postgres 11616 May 14 18:04 30949.2.optimized.bc
-rw-------. 1 postgres postgres 18732 May 14 18:04 30949.3.bc
-rw-------. 1 postgres postgres 15592 May 14 18:04 30949.3.optimized.bc
-rw-------. 1 postgres postgres 15516 May 14 18:04 30949.4.bc
-rw-------. 1 postgres postgres 11616 May 14 18:04 30949.4.optimized.bc
-rw-------. 1 postgres postgres 14332 May 14 18:04 30949.5.bc
-rw-------. 1 postgres postgres  9808 May 14 18:04 30949.5.optimized.bc
-rw-------. 1 postgres postgres 21100 May 14 18:05 30949.6.bc
-rw-------. 1 postgres postgres 16800 May 14 18:05 30949.6.optimized.bc
-rw-------. 1 postgres postgres 17180 May 14 18:05 30949.7.bc
-rw-------. 1 postgres postgres 13688 May 14 18:05 30949.7.optimized.bc
```
The file name consists of the MyProcPid, llvm_generation, and extension. We could use the tool llvm-dis to turn a .bc file into a .ll file.
```
[root@neoc data]# llvm-dis 30949.0.bc
cat 30949.0.ll
; ModuleID = '30949.0.bc'
source_filename = "pg"
target datalayout = "e-m:e-i64:64-f80:128-n8:16:32:64-S128"
target triple = "x86_64-unknown-linux-gnu"

%struct.ExprState = type { %struct.Node, i8, i8, i64, %struct.TupleTableSlot*, %struct.ExprEvalStep*, i64 (%struct.ExprState*, %struct.ExprContext*, i8*)*, %struct.Expr*, i8*, i32, i32, %struct.PlanState*, %struct.ParamListInfoData*, i64*, i8*, i64*, i8* }
%struct.Node = type { i32 }
%struct.TupleTableSlot = type { i32, i8, i8, i8, i8, %struct.HeapTupleData*, %struct.tupleDesc*, %struct.MemoryContextData*, i32, i32, i64*, i8*, %struct.MinimalTupleData*, %struct.HeapTupleData, i32, i8 }
%struct.tupleDesc = type { i32, i32, i32, i8, i32, %struct.tupleConstr*, [0 x %struct.FormData_pg_attribute] }
%struct.tupleConstr = type { %struct.attrDefault*, %struct.constrCheck*, %struct.attrMissing*, i16, i16, i8 }
%struct.attrDefault = type { i16, i8* }
%struct.constrCheck = type { i8*, i8*, i8, i8 }
.
.
.
It's too lang.
```

#### How the llvm improve the utilization of cpu cache?
You could see it in the following picture:
We could use 'select column from table where condition group by column order by  column;' as an example:
![improve cpu cache](/assets/improve%20cpu%20cache_rfwpyjg8u.png)
Without the JIT, data are obtained one by one(like the left).
With the JIT, we could get a lot of data to process together(like the right).
This may be to improve the performance with enhancing the optimization options or changing the order of optimization.

### Do the performance testing
#### Hardware Configuration：
* CPU : i5 8400
* Memory : 16G DDR4
* Disk : Inter SSD 1T
* OS : CentOS Linux release 7.6.1810 (Core)
#### Installation PostgreSQL 11 and 10：
1. Install the PostgreSQL 11 by source code:
```
git clone -b REL_11_STABLE git://git.postgresql.org/git/postgresql.git pg11
```
2. Compiled the database:

```
./configure --prefix=/home/postgres/pg11JIT --with-openssl --with-libxml --with-libxslt --with-llvm
make -j6; make install
cd contrib
make -j6; make install
```
3. Setting environment variables:
```
export PATH=/home/postgres/pg11JIT/bin/:$PATH
```
 4. installation the database:
```
su - postgres
cd /home/postgres/
mkdir data
initdb -D data/
```
5. Start service:
```
pg_ctl -D ../data
psql
psql (11.2)
Type "help" for help.

postgres=#
```
6. the Configuration of postgresql.conf I changed:
```
shared_buffers = 4GB
temp_buffers = 64MB
work_mem = 32MB
maintenance_work_mem = 1GB
wal_level = minimal
max_wal_senders = 0
max_wal_size = 40GB
random_page_cost = 1.1
```
7. Install the PostgreSQL 10 and the Configuration of postgresql.conf I changed:
```
git clone -b REL_11_STABLE git://git.postgresql.org/git/postgresql.git pg10
cd pg10
./configure --prefix=/home/postgres/pgdb10 --with-openssl --with-libxml --with-libxslt --with-llvm
make -j6; make install
cd contrib
make -j6; make install

shared_buffers = 4GB
temp_buffers = 64MB
work_mem = 32MB
maintenance_work_mem = 1GB
wal_level = minimal
max_wal_senders = 0
max_wal_size = 40GB
random_page_cost = 1.1
```

v  supplier.csv

[postgres@shawnc dbgen]$ ln -s /home/postgres/tpch/tpch/dbgen/dss/data/ /tmp/dss-data
[postgres@shawnc dbgen]$ ll /tmp/
total 0
lrwxrwxrwx. 1 postgres postgres 40 Apr 26 23:07 dss-data -> /home/postgres/tpch/tpch/dbgen/dss/data/
```

#### Get the pg_tpch
This repository of pg_tpch contains a simple implementation that runs a TPC-H-like benchmark with a PostgreSQL database. It builds on the official TPC-H benchmark available at http://tpc.org/tpch/default.asp (uses just the dbgen a qgen parts).

1. Get the repository:
```
[postgres@shawnc tpch]$ git clone https://github.com/tvondra/pg_tpch.git
Cloning into 'pg_tpch'...
remote: Enumerating objects: 47, done.
remote: Total 47 (delta 0), reused 0 (delta 0), pack-reused 47
Unpacking objects: 100% (47/47), done.
```
2. Adding some tools for pg_tpch:
```
[postgres@shawnc tpch]$ cd pg_tpch/
[postgres@shawnc pg_tpch]$ cp ../tpch/dbgen/dbgen ./
[postgres@shawnc pg_tpch]$ cp ../tpch/dbgen/qgen ./
[postgres@shawnc pg_tpch]$ cp ../tpch/dbgen/dists.dss ./
[postgres@shawnc pg_tpch]$ 
```
3. Generating directory dss/queries:
```
[postgres@shawnc pg_tpch]$ mkdir dss/queries
```
4. modified queries in the 'dss/templates' directory :
```
for q in `seq 1 22`
do
    DSS_QUERY=dss/templates ./qgen $q >> dss/queries/$q.sql
    sed 's/^select/explain select/' dss/queries/$q.sql > dss/queries/$q.explain.sql
    cat dss/queries/$q.sql >> dss/queries/$q.explain.sql;
done
```

#### Running the benchmark

The actual benchmark is implemented in the 'tpch.sh' script. It expects an already prepared database and four parameters - directory where to place the results, database and user name. So to run it, do this:
```
[postgres@shawnc pg_tpch]$
[postgres@shawnc pg_tpch]$ ./tpch.sh ./results postgres postgres
./tpch.sh: line 166: kill: (578) - No such process
./tpch.sh: line 189: 32072 Terminated              vmstat $DELAY >> $RESULTS/vmstat.log
```
and wait until the benchmark.


*note: If there is a problem, please check the files in the results/error directory. I have an error, there is not a time command in my OS, so I installation it.*
```
yum install time -y
```
#### Processing the results
All the results are written into the output directory (first parameter). To get useful results (timing of each query, various statistics), you can use script process.php. It expects two parameters - input dir (with data collected by the tpch.sh script) and output file (in CSV format). For example like this:
```
php process.php ./results output.csv
```
#### Test Plan
As we all know, PG 10 and 11 have the function of parallel query.
##### What is Parallel Query?
>PostgreSQL can devise query plans which can leverage multiple CPUs in order to answer queries faster. This feature is known as parallel query.

In order to avoid the impact of parallel query, I turn off this feature for testing in the first.
My configuration parameters are:
1. Turn on JIT in PostgreSQL 11:
```
set jit=on;
```
2. Turn on JIT in PostgreSQL 11:
```
set jit=off;
```
3. PostgreSQL 10:
```
as default
```
We could get the results like this(unit is second, JIT disabled in PG11 and JIT enabled in PG11):
![PG112](/assets/PG112_atz0727xa.png)
The total is the sum of all 22 SQL execution times.
To be more intuitive, we can look at the histogram:
![pg11h](/assets/pg11h.png)
We will find that the execution time is reduced by turning on the JIT, about 6.1% down.
With the JIT turned on, performance has dropped.
I had checked the explain. I found that the JIT disabled used less time than JIT enabled in the Nested Loop.
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
If you want to know more details, please refer to https://github.com/wangguoke/blog/tree/master/results%20about%20tpch .

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

### overall summary of JIT performance:
This JIT depends on the situation. In the 22 SQL cases, performance has improved, and some have declined.
I think there are two issues in the JIT.
1. The default values about the parameters for JIT are not right.
2. The query optimizer has some unreasonable design.

### Suggestions for Improvement
1. Improve the cost value or add new parameters to find the right case.
2. We can add some special passes for PostgreSQL, like the Nested Loop.
3. We can improve our query optimizer based on the information provided by LLVM.

### Reference
* https://www.postgresql.org/docs/11/runtime-config-developer.html
* https://www.postgresql.org/docs/9.6/parallel-query.html
* 《JIT-Compiling SQL Queries in PostgreSQL Using LLVM》
* 《The Architecture of Open Source Applications: LLVM》
