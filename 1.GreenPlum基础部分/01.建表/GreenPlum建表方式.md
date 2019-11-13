
# GreenPlum表的创建

## heap表
```
    drop table if exists t01;
    create table t01(id int primary key) distributed by (id);
```
AO表不压缩 (AO表是否不能有主键？)
    drop table if exists t02;
    create table t02(id int) with (appendonly=true) distributed by (id);
AO表压缩
    drop table if exists t03;
    create table t03(id int) with (appendonly=true, compresslevel=5)
    distributed by (id);
AO表列存压缩 与上表的压缩方式不同
    drop table if exists t04;
    create table t04(id int) with (appendonly=true,compresslevel=5, orientation=column) distributed by (id);
分区表日期分区
    drop table if exists t05;
    create table t05(id int, createOn date) with (appendonly=true, compresslevel=5) distributed by (id)
    Partition By Range(createOn)
    (
        partition p2016 start ('2016/01/01'::date) end ('2016/12/31'::date),
        partition p2017 start ('2017/01/01'::date) end ('2017/12/31'::date),
        partition p2018 start ('2018/01/01'::date) end ('2018/12/31'::date),
        default partition otherTime
    );
分区表数字分区
    drop table if exists t06;
    create table t06(id int,dayOfPeriod int) distributed by (id)
    partition by range(dayOfPeriod)
    (
        start (1) end (31) every(1),
        default partition none
    );
分区表列表分区
    drop table if exists t07;
    create table t07(id int , gender char(2)) with(appendonly=true,compresslevel=5) distributed by (id)
    partition by list(gender)
    (
        partition man values('m'),
        partition woman values('f'),
        default partition Unkown
    );
二级分区表
    drop table if exists t08;
    create table t08(id int, period date, Sales decimal(9,6), Region varchar(500)) distributed by (id)
    partition by range (period)
    subpartition by list (region)
    subpartition template
    (
        subpartition usa values ('usa'),
        subpartition asia values ('asia'),
        subpartition europe values ('europe'),
        default subpartition other
    )
    (
        start ('2018-01-01'::date) inclusive
        end ('2019-01-01'::date) exclusive
        every (interval '1 month'),
        default partition other
    );
可读外部表
    gpfdist -p 9001 -d /home/gpadmin/install/BudgetFFC.csv -l /home/gpadmin/log/external_table.log &
    create external table t09 (id integer,blobaldescib text,foh numeric,voh numeric)
    location ('gpfdist://mdw:9001/BudgetFFC5.9.csv')
    format 'csv' (header) encoding 'utf-8';
可写外部表
    drop table if exists t10;
    create writable external table t10(like t09)
    location ('gpfdist://mdw:9001/t10.out')
    format 'text' (delimiter '|' null ' ')
    distributed by (id);
file协议外部表
    drop table if exists t12;
    create external table t12(like t10)
    location (
            'file://sdw1:9002/home/gpadmin/install/t10.out',
            'file://sdw1:9002/home/gpadmin/install/t10_split.out',
         )
    format 'text' (delimiter '|') encoding 'utf-8';
下面执行结果

     testDB=# select * from t12;
     id | blobaldescib |    foh    |    voh
    ----+--------------+-----------+-----------
      1 | SA           | 32423.234 | 23423.000
      2 | SC           | 32423.234 | 23423.000
      3 | CA           |  10000.88 | 23423.000
      4 | BA           |  10000.88 |      8988
      5 | CP           |      2000 |      8988
      6 | SC           |      2000 |      8988
      7 | TW           |      1000 |       500
    (7 rows)
web外部表
    drop table if exists t11;
    create external web table t11(like t09)
    execute 'cat /home/gpadmin/install/BudgetFFC5.9.csv'
    format 'csv' (header) encoding 'utf-8';
将文件传送到segment所在机器（我这里计算节点都在sdw1上）

gpscp -h sdw1 BudgetFFC5.9 =:/home/gpadmin/install/
/* result
  testDB=# select gp_segment_id ,count(*) from t11 group by 1;
  gp_segment_id | count
  --------------+-------
              1 | 94383
              0 | 94383
  (2 rows)           */
