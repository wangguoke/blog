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
customer.csv  lineitem.csv  nation.csv  orders.csv  part.csv  partsupp.csv  region.cs
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
