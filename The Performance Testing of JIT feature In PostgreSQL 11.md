#The Performance Testing of JIT feature In PostgreSQL 11

As the data increases, the OLAP capabilities of the database will also be taken seriously.  OLAP functionality is primarily affected by data throughput and CPU computing power.  Now with the rise of SSD, Column Storage and Distributed Databases, I/O is not the main bottleneck of the database. Modern databases add a lot of logical and virtual function calls in response to various scenarios.  A lot of redundant operations have been added to the original operation to ensure that enough scenes are handled. But this reduces the efficiency of OLAP capabilities for database. In the PostgreSQL 11, they added the Just-in-Time compilation which could reduce the redundant logical operations and virtual function calls.
1. Because those, I will do performance testing of JIT feature and identify the performance difference with JIT one and off.
2. Another purpose is to compare the peformance of PG 11 with/without JIT enabled and compare it with previous 10.* version.

###What is Just-in-Time compilation?
>Just-in-Time compilation (JIT) is the process of turning some form of interpreted program evaluation into a native program, and doing so at runtime.
>For example, instead of using a facility that can evaluate arbitrary SQL expressions to evaluate an SQL predicate like WHERE a.col = 3, it is possible to generate a function than can be natively executed by the CPU that just handles that expression, yielding a speedup.
>That this is done at query execution time, possibly even only in cases where the relevant task is done a number of times, makes it JIT, rather than ahead-of-time (AOT). Given the way JIT compilation is used in PostgreSQL, the lines between interpretation, AOT and JIT are somewhat blurry.
>Note that the interpreted program turned into a native program does not necessarily have to be a program in the classical sense. E.g. it is highly beneficial to JIT compile tuple deforming into a native function just handling a specific type of table, despite tuple deforming not commonly being understood as a "program".

###How to JIT?
>PostgreSQL, by default, uses LLVM to perform JIT. LLVM was chosen because it is developed by several large corporations and therefore unlikely to be discontinued, because it has a license compatible with PostgreSQL, and because its IR can be generated from C using the Clang compiler.

###What is the LLVM?
>The LLVM Project is a collection of modular and reusable compiler and toolchain technologies. Despite its name, LLVM has little to do with traditional virtual machines. The name "LLVM" itself is not an acronym; it is the full name of the project.
>LLVM began as a research project at the University of Illinois, with the goal of providing a modern, SSA-based compilation strategy capable of supporting both static and dynamic compilation of arbitrary programming languages. Since then, LLVM has grown to be an umbrella project consisting of a number of subprojects, many of which are being used in production by a wide variety of commercial and open source projects as well as being widely used in academic research. Code in the LLVM project is licensed under the "Apache 2.0 License with LLVM exceptions".

####LLVM's implementation of the three-phase design
The following picture despites the three-phase design of llvm.
![Image [2]](/assets/Image%20[2].png)
In an LLVM-based compiler, a front end is responsible for parsing, validating and diagnosing errors in the input code, then translating the parsed code into LLVM IR (usually, but not always, by building an AST and then converting the AST to LLVM IR). This IR is optionally fed through a series of analysis and optimization passes which improve the code, then is sent into a code generator to produce native machine code.
####The time about LLVM
![Image [3]](/assets/Image%20[3].png)
As mentioned earlier, LLVM IR can be efficiently (de)serialized to/from a binary format known as LLVM bitcode. Since LLVM IR is self-contained, and serialization
is a lossless process, we can do part of compilation, save our progress to disk, then continue work at some point in the future. This feature provides a number of
interesting capabilities including support for link-time and install-time optimization, both of which delay code generation from "compile time".

###How to open the JIT in PostgreSQL 11?
####Build with --with-llvm
PostgreSQL has builtin support to perform JIT compilation using LLVM when PostgreSQL is built with --with-llvm.
```
./configure --prefix=/home/postgres/pg11JIT --with-openssl --with-libxml --with-libxslt --with-llvm
make -j6; make install
cd contrib
make -j6; make install
```

####Turn on the JIT
>The configuration variable jit determines whether JIT compilation is enabled or disabled. If it is enabled, the configuration variables jit_above_cost, jit_inline_above_cost, and jit_optimize_above_cost determine whether JITcompilation is performed for a query, and how much effort is spent doing so.
```
set jit=on;
set jit_above_cost=0;
set jit_inline_above_cost=0;
set jit_optimize_above_cost=0;
```
jit_provider determines which JIT implementation is used. It is rarely required to be changed.

###How does the JIT work?
####Reduce the redundant logical and the virtual function calls
We could use 'select 3<4;' as an example:
![3compare4](/assets/3compare4_9a1pmtntz.png)

###Do the performance testing
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
max_worker_processes = 32
max_parallel_maintenance_workers = 2
max_parallel_workers_per_gather = 6
max_parallel_workers = 0
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
max_worker_processes = 32
max_parallel_workers_per_gather = 6
max_parallel_workers = 0
```

#### Get data and tools
1. Get the tpch from http://www.tpc.org/default.asp:
```
mkdir tpch; cd tpch
```
2. Unzip and generate Makefile:
```
[postgres@shawnc tpch]$ ls
tpc-h-tool2.18.0_rc2.zip
[postgres@shawnc tpch]$ unzip tpc-h-tool2.18.0_rc2.zip; ls
2.18.0_rc2  tpc-h-tool2.18.0_rc2.zip
[postgres@shawnc tpch]$ mv 2.18.0_rc2/ tpch
[postgres@shawnc tpch]$ mv tpc-h-tool2.18.0_rc2.zip ..
[postgres@shawnc tpch]$ ls
tpch
[postgres@shawnc tpch]$ cd tpch/dbgen/
[postgres@shawnc dbgen]$ cp makefile.suite Makefile
```
3. modify the Makefile so that it contains:
```
################
## CHANGE NAME OF ANSI COMPILER HERE
################
CC      = gcc
# Current values for DATABASE are: INFORMIX, DB2, TDAT (Teradata)
#                                  SQLSERVER, SYBASE, ORACLE, VECTORWISE
# Current values for MACHINE are:  ATT, DOS, HP, IBM, ICL, MVS,
#                                  SGI, SUN, U2200, VMS, LINUX, WIN32
# Current values for WORKLOAD are:  TPCH
DATABASE= ORACLE
MACHINE = LINUX
WORKLOAD = TPC
```
4. Generate some tools:
```
[postgres@shawnc dbgen]$ make
gcc -g -DDBNAME=\"dss\" -DLINUX -DORACLE -DTPCH -DRNG_TEST -D_FILE_OFFSET_BITS=64    -c -o build.o build.c
.
.
.
bm_utils.o qgen.o rnd.o varsub.o text.o bcd2.o permute.o speed_seed.o rng64.o -lm
```
5. Generating 10G data：
```
[postgres@shawnc dbgen]$ ./dbgen -s 10
TPC-H Population Generator (Version 2.18.0)
Copyright Transaction Processing Performance Council 1994 - 2010
```
6. Which creates a bunch of .tbl files in Oracle-like CSV format:
```
[postgres@shawnc dbgen]$ ls *.tbl
customer.tbl  lineitem.tbl  nation.tbl  orders.tbl  partsupp.tbl  part.tbl  region.tbl  supplier.tbl
```
7. To convert them to a CSV format compatible with PostgreSQL, do this:
```
[postgres@shawnc dbgen]$ for i in `ls *.tbl`; do sed 's/|$//' $i > ${i/tbl/csv}; echo $i; done;
customer.tbl
lineitem.tbl
nation.tbl
orders.tbl
partsupp.tbl
part.tbl
region.tbl
supplier.tbl

[postgres@shawnc dbgen]$ ls *.csv
customer.csv  lineitem.csv  nation.csv  orders.csv  part.csv  partsupp.csv  region.csv  supplier.csv
```
8. Move these data to the 'dss/data' directory or somewhere else, and create a symlink to /tmp/dss-data:
```
[postgres@shawnc dbgen]$ mkdir -p dss/data
[postgres@shawnc dbgen]$ mv *.csv dss/data/
[postgres@shawnc dbgen]$ ls dss/data/
customer.csv  lineitem.csv  nation.csv  orders.csv  part.csv  partsupp.csv  region.csv  supplier.csv

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


*note: If there is a problem, please check the files in the results/error directory. I have a problem, there is not a time command in my OS, so I installation it.*
```
yum install time -y
```
#### Processing the results
All the results are written into the output directory (first parameter). To get useful results (timing of each query, various statistics), you can use script process.php. It expects two parameters - input dir (with data collected by the tpch.sh script) and output file (in CSV format). For example like this:
```
php process.php ./results output.csv
```
####Test Plan
As we all know, PG 10 and 11 have the function of parallel query.
#####What is Parallel Query?
>PostgreSQL can devise query plans which can leverage multiple CPUs in order to answer queries faster. This feature is known as parallel query.

In order to avoid the impact of parallel query, I turn off this feature for testing in the first.
My configuration parameters are:
1. Turn on JIT in PostgreSQL 11:
```
set jit=on;
set jit_above_cost=0;
set jit_inline_above_cost=0;
set jit_optimize_above_cost=0;

max_worker_processes = 32
max_parallel_maintenance_workers = 0
max_parallel_workers_per_gather = 0
max_parallel_workers = 0
```
2. Turn on JIT in PostgreSQL 11:
```
set jit=off;
set jit_above_cost=0;
set jit_inline_above_cost=0;
set jit_optimize_above_cost=0;

max_worker_processes = 32
max_parallel_maintenance_workers = 0
max_parallel_workers_per_gather = 0
max_parallel_workers = 0
```
3. PostgreSQL 10:
```
max_worker_processes = 32
max_parallel_maintenance_workers = 0
max_parallel_workers_per_gather = 0
max_parallel_workers = 0
```
We could get the results like this(unit is second):
![results for JIT whit PG 10](/assets/results%20for%20JIT%20whit%20PG%2010.png)
The query_1, query_2 ... and the query_22 is the 22 SQL for the testing of TPCH.
The total is the sum of all 22 SQL execution times.
The db_cache_hit_radio is the db cache hit radio. The results of the three tests are almost the same.
To be more intuitive, we can look at the histogram:
![the histogram of turn on JIT](/assets/the%20histogram%20of%20turn%20on%20JIT_xcqbb7an5.png)
We will find that the execution time is greatly reduced with turning on the JIT.
And we could get this:
![percentage1](/assets/percentage1_wjhhuri8s.png)
Now, let's turn on the parallel query.
My configuration parameters are:
1. Turn on JIT in PostgreSQL 11:
```
set jit=on;
set jit_above_cost=0;
set jit_inline_above_cost=0;
set jit_optimize_above_cost=0;

max_worker_processes = 32
max_parallel_maintenance_workers = 2
max_parallel_workers_per_gather = 6
max_parallel_workers = 6
```
2. Turn on JIT in PostgreSQL 11:
```
set jit=off;
set jit_above_cost=0;
set jit_inline_above_cost=0;
set jit_optimize_above_cost=0;

max_worker_processes = 32
max_parallel_maintenance_workers = 2
max_parallel_workers_per_gather = 6
max_parallel_workers = 6
```
3. PostgreSQL 10:
```
max_worker_processes = 32
max_parallel_maintenance_workers = 2
max_parallel_workers_per_gather = 6
max_parallel_workers = 6
```
We could get the results like this(unit is second):
![results for JIT whit PG 10 2](/assets/results%20for%20JIT%20whit%20PG%2010%202.png)
To be more intuitive, we can look at the histogram:
![the histogram of turn on JIT 2](/assets/the%20histogram%20of%20turn%20on%20JIT%202_hl1mm9q63.png)
We will find that the execution time is greatly reduced with turning on the JIT.
And we could get this:
![percentage2](/assets/percentage2.png)

###overall summary of JIT performance:
This JIT depends on the situation. If we turn on the JIT, we could get a 30% performance boost. But we must turn off the parallel query.
So I think there are some issues in JIT on with Parallel on.

###Suggestions for Improvement
I know that llvm can improve the utilization of cpu cache. But I could not find it in PostgreSQL 11. Maybe I still need to learn further.
####How the llvm improve the utilization of cpu cache?
You could see it in the following picture:
We could use 'select column from table where condition group by column order by  column;' as an example:
![improve cpu cache](/assets/improve%20cpu%20cache_rfwpyjg8u.png)
Without the JIT, data is obtained one by one(like the left).
With the JIT, we could get a lot of data to process together(like the right).
This may be to improve the peformance with enhance the optimization options or changing the order of optimization.

###Reference
https://www.postgresql.org/docs/11/runtime-config-developer.html
https://www.postgresql.org/docs/9.6/parallel-query.html
《JIT-Compiling SQL Queries in PostgreSQL Using LLVM》
《The Architecture of Open Source Applications: LLVM》
