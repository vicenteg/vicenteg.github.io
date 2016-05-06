# TPCH on AWS

## Download the TPC-H (SF 100) data in Parquet format
You can download the tpch parquet dataset from here:

http://drill-public.s3.amazonaws.com/tpch/sf100/parquet/tpch_sf100_parquet.tgz

Parallelize with:

```
hadoop distcp s3a://drill-public/tpch/sf100/parquet/tpch_sf100_parquet.tgz /benchmarks/tpch_sf100_parquet.tgz
```


## Generate the data

dbgen can only generate delimited data. Useful maybe for comparison.


