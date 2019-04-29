Recently, I am learning JIT in PostgreSQL 11. I found that the
JIT is not enabled by default in PG 11. I guess there are some
issues.So I completed the benchmark test in 6 cases.
* Close the JIT and Parallel in postgresql.conf in PostgreSQL 11;
* Open the JIT and CLose Parallel in postgresql.conf in PostgreSQL 11;
* Close the JIT and Open Parallel in postgresql.conf in PostgreSQL 11;
* Open the JIT and Parallel in postgresql.conf in PostgreSQL 11;
* Close Parallel in postgresql.conf in PostgreSQL 10;
* Open Parallel in postgresql.conf in PostgreSQL 10.
### The following is my operation process:
#### Hardware Configuration：
* CPU : i5 8400
* Memory : 16G DDR4
* Disk : Inter SSD 1T
* OS : CentOS Linux release 7.6.1810 (Core)
#### Installation PostgreSQL 11 and 10：
1. Install the PostgreSQL 11 by source code:
```
git clone -b REL_11_STABLE git://git.postgresql.org/git/postgresql.git
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
export PATH=//home/postgres/pg11JIT/bin/:$PATH
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


* * *

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

* * *
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

* * *
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
#### The Results

1. Close the JIT and Parallel in postgresql.conf in PostgreSQL 11:
```
set jit=off;
set jit_above_cost=0;
set jit_inline_above_cost=0;
set jit_optimize_above_cost=0;

max_worker_processes = 32
max_parallel_maintenance_workers = 2
max_parallel_workers_per_gather = 6
max_parallel_workers = 0
```
2. Open the JIT and CLose Parallel in postgresql.conf in PostgreSQL 11:
```
set jit=on;
set jit_above_cost=0;
set jit_inline_above_cost=0;
set jit_optimize_above_cost=0;

max_worker_processes = 32
max_parallel_maintenance_workers = 2
max_parallel_workers_per_gather = 6
max_parallel_workers = 0
```
3. Close the JIT and Open Parallel in postgresql.conf in PostgreSQL 11:
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
4. Open the JIT and Parallel in postgresql.conf in PostgreSQL 11:
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

5. Close Parallel in postgresql.conf in PostgreSQL 10:
```
max_worker_processes = 32
max_parallel_workers_per_gather = 6
max_parallel_workers = 0
```

6. Open Parallel in postgresql.conf in PostgreSQL 10:
```
max_worker_processes = 32
max_parallel_workers_per_gather = 6
max_parallel_workers = 6
```
I got the results like this:
![The results of time of JIT on or off in PostgreSQL 11 and Parallel on or off in PostgreSQL 10](/assets/The%20results%20of%20time%20of%20JIT%20on%20or%20off%20in%20PostgreSQL%2011%20and%20Parallel%20on%20or%20off%20in%20PostgreSQL%2010.png)

you could see the Histogram:
1. Parallel off with JIT on or off in PostgreSQL 11, JIt on is 36.35% higher than off performance with Parallel off:
![Parallel off with JIT on or off in PostgreSQL 11](/assets/Parallel%20off%20with%20JIT%20on%20or%20off%20in%20PostgreSQL%2011.png)
2. Parallel on with JIT on or off in PostgreSQL 11, JIt on is 8.59% lower than off performance with Parallel off:
![Parallel on with JIT on or off in PostgreSQL 11](/assets/Parallel%20on%20with%20JIT%20on%20or%20off%20in%20PostgreSQL%2011.png)
3. JIT on or off in PostgreSQL 11 and in PostgreSQL 10 with Parallel off, JIt on in PostgreSQL 11 is 51.02% higher than off in PostgreSQL 10 performance with Parallel off:
![JIT on or off in PostgreSQL 11 and in PostgreSQL 10 with Parallel off](/assets/JIT%20on%20or%20off%20in%20PostgreSQL%2011%20and%20in%20PostgreSQL%2010%20with%20Parallel%20off.png)
4. JIT on or off in PostgreSQL 11 and in PostgreSQL 10 with Parallel on, JIt on in PostgreSQL 11 is 27.33% higher than off in PostgreSQL 10 performance with Parallel off:
![JIT on or off in PostgreSQL 11 and in PostgreSQL 10 with Parallel on](/assets/JIT%20on%20or%20off%20in%20PostgreSQL%2011%20and%20in%20PostgreSQL%2010%20with%20Parallel%20on.png)

Written at the end:
There are some issues in JIT on with Parallel on. I think this is the reason why the JIT is turned off by default.

After my study, the process of JIT is as follows:
![JIT](/assets/JIT_kf27x0loh.png)
```
1. Use the clang to compile the c files.
2. Load the bc when the PostgreSQL Start sevice
3. SQL query
4. Check for cost
5. if high cost, do it in standard executor, else do it in the LLVM JIT
```
i think the cost of optimization(step 5) is very high in Parallel.
