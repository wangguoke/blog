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
