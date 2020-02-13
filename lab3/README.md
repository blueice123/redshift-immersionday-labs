# LAB 3 - Table Design and Query Tuning
In this lab you will analyze the affects of Compression, De-Normalization, Distribution and Sorting on Redshift query performance.

## Contents
* [Before You Begin](#before-you-begin)
* [Compressing and De-Normalizing](#compressing-and-de-normalizing)
* [Distributing and Sorting](#distributing-and-sorting)
* [Result Set Caching and Execution Plan Reuse](#result-set-caching-and-execution-plan-reuse)
* [Selective Filtering](#selective-filtering)
* [Join Strategies](#join-strategies)
* [Before You Leave](#before-you-leave)
 
## Before You Begin
This lab assumes you have launched a Redshift cluster, loaded it with TPC Benchmark data and can gather the following information.  If you have not launched a cluster, see [LAB 1 - Creating Redshift Clusters](../lab1/README.md).  If you have not yet loaded it, see [LAB 2 - Data Loading](../lab2/README.md).
* [Your-Redshift_Hostname]
* [Your-Redshift_Port]
* [Your-Redshift_Username]
* [Your-Redshift_Password]

It also assumes you have access to a configured client tool. For more details on configuring SQL Workbench/J as your client tool, see [Lab 1 - Creating Redshift Clusters : Configure Client Tool](../lab1/README.md#configure-client-tool). As an alternative you can use the Redshift provided online Query Editor which does not require an installation.
```
https://console.aws.amazon.com/redshift/home?#query:
```
## Compressing and De-Normalizing
### Standard layout
Redshift는 많은 양의 데이터에서 작동합니다. Redshift 워크로드를 최적화하기 위해 핵심 원칙 중 하나는 저장된 데이터 양을 줄이는 것입니다. 압축 알고리즘 세트를 사용하여이 볼륨을 줄입니다. 다른 유형과 기능의 값을 포함하는 전체 데이터 행을 처리하는 대신 Redshift는 열 방식으로 작동하므로 단일 열의 데이터에서 작동 할 수있는 알고리즘을 구현할 수있어 효율성이 크게 향상됩니다. 이 예에서는 데이터를 테이블에로드하고 사용할 수있는 압축 체계를 테스트합니다.

Note: COPY 명령을 빈 테이블로 사용할 때 테이블의 열에 자동으로 압축 인코딩을 적용 할 수 있습니다. 그러나이 실습에서는 데이터가로드 된 후 수동으로 압축을 분석하고 적용하여 올바른 압축을 사용할 때의 성능 향상을 보여줍니다.

1. 고객 키(c_custkey)에 DISTKEY 만 지정하여 기본 설정을 사용하여 고객 테이블을 작성하십시오..
```
CREATE TABLE customer_v1 (
  c_custkey int8 NOT NULL PRIMARY KEY                      ,
  c_name varchar(25) NOT NULL                              ,
  c_address varchar(40) NOT NULL                           ,
  c_nationkey int4 NOT NULL REFERENCES nation(n_nationkey) ,
  c_phone char(15) NOT NULL                                ,
  c_acctbal numeric(12,2) NOT NULL                         ,
  c_mktsegment char(10) NOT NULL                           ,
  c_comment varchar(117) NOT NULL
) DISTKEY (c_custkey);
```

2. 데이터를 Customer 테이블에서 로드하고, 테이블 통계를 갱신(Analyze) 하십시요. 
```
INSERT INTO customer_v1
SELECT * FROM customer;
```
```
ANALYZE customer_v1;
```

3. 이 테이블의 스토리지 최적화 옵션을 분석하십시오. 원하는 압축 구성표를 선택하거나 Redshift가 자동 압축 구성표를 결정하도록 할 수 있습니다. 이 쿼리는 테이블의 데이터를 분석하고이 테이블의 스토리지 및 성능을 개선하는 방법에 대한 권장 사항을 제공합니다. 이 명령문의 결과는 중요하며 테이블에 저장된 현재 실제 데이터를 기반으로 스토리지를 최적화하는 방법에 대한 통찰력을 제공합니다.
```
ANALYZE COMPRESSION customer_v1;
```

### Compression Optimization
앞서 수행한 "Analyze compression" 결과에 따라, 우리는 새로운 테이블을 생성하고, 소트리지에 다른 형태로 레코드를 등록할 수 있으며, 그 차이를 확인할 수 있습니다. 

4. 마지막 Analyze compression 결과에서 제공된 힌트를 이용하여 새롭게 테이블을 구성합니다.  
```
CREATE TABLE customer_v2 (
  c_custkey int8 NOT NULL ENCODE DELTA PRIMARY KEY   ,
  c_name varchar(25) NOT NULL ENCODE ZSTD            ,
  c_address varchar(40) NOT NULL ENCODE ZSTD         ,
  c_nationkey int4 NOT NULL ENCODE ZSTD              ,
  c_phone char(15) NOT NULL ENCODE ZSTD              ,
  c_acctbal numeric(12,2) NOT NULL ENCODE ZSTD       ,
  c_mktsegment char(10) NOT NULL ENCODE ZSTD         ,
  c_comment varchar(117) NOT NULL ENCODE ZSTD
) DISTKEY (c_custkey);
```

5. 기존 테이블로부터 자료를 로드하고, 다시 테이블 통계를 갱신(Analyze 수행) 합니다.
```
INSERT INTO customer_v2
SELECT * FROM customer_v1;
```
```
ANALYZE customer_v2;
```

6. 압축 전후에 이러한 테이블의 스토리지 공간을 분석하십시오. 테이블은 MiB에 사용 된 스토리지 양을 열별로 저장합니다. 첫 번째 테이블에 비해 두 번째 테이블의 스토리지가 약 50 % 절약됩니다. 이 쿼리는 각 테이블의 열당 스토리지 요구 사항을 제공 한 다음 테이블의 총 스토리지 (각 라인에서 동일하게 반복)를 제공합니다..
```
SELECT
  CAST(d.attname AS CHAR(50)),
  SUM(CASE WHEN CAST(d.relname AS CHAR(50)) LIKE '%v1%'
THEN b.size_in_mb ELSE 0 END) AS size_in_mb_v1,
  SUM(CASE WHEN CAST(d.relname AS CHAR(50)) LIKE '%v2%'
THEN b.size_in_mb ELSE 0 END) AS size_in_mb_v2,
  SUM(SUM(CASE WHEN CAST(d.relname AS CHAR(50)) LIKE '%v1%'
THEN b.size_in_mb ELSE 0 END)) OVER () AS total_mb_v1,
  SUM(SUM(CASE WHEN CAST(d.relname AS CHAR(50)) LIKE '%v2%'
THEN b.size_in_mb ELSE 0 END)) OVER () AS total_mb_v2
FROM (
  SELECT relname, attname, attnum - 1 as colid
  FROM pg_class t
  INNER JOIN pg_attribute a ON a.attrelid = t.oid
  WHERE t.relname LIKE 'customer\_v%') d
INNER JOIN (
  SELECT name, col, MAX(blocknum) AS size_in_mb
  FROM stv_blocklist b
  INNER JOIN stv_tbl_perm p ON b.tbl=p.id
  GROUP BY name, col) b
ON d.relname = b.name AND d.colid = b.col
GROUP BY d.attname
ORDER BY d.attname;
```

### Data De-Normalizing
압축을 진행 할 경우, 팩트 테이블 내에 "참조"데이터를 저장할 수 있으므로 스토리지 최적화만 신경 쓰면 됩니다. (SORTKEY 제외) "스타"또는 "스노 플레이크" 데이터베이스 디자인과 같이 압축 알고리즘을 같이 고려할 필요가 없습니다.  이 실습 섹션에서는 국가 및 지역 정보를 고객 테이블로 비정규 화하여 이러한 열을 통합하고 차이점을 분석합니다.

7. 새로운 customer 테이블을 생성하고, nation, region 이를에 대한 반정규화를 수행합니다. 이를 통하여 custom table에서 직접 두 자료를 확인할 수 있게 됩니다. 
```
CREATE TABLE customer_v3 (
  c_custkey int8 NOT NULL ENCODE DELTA PRIMARY KEY                ,
  c_name varchar(25) NOT NULL ENCODE ZSTD                         ,
  c_address varchar(40) NOT NULL ENCODE ZSTD                      ,
  c_nationname char(25) NOT NULL ENCODE ZSTD                      ,
  c_regionname char(25) NOT NULL ENCODE ZSTD                      ,
  c_phone char(15) NOT NULL ENCODE ZSTD                           ,
  c_acctbal numeric(12,2) NOT NULL ENCODE ZSTD                    ,
  c_mktsegment char(10) NOT NULL ENCODE ZSTD                      ,
  c_comment varchar(117) NOT NULL ENCODE ZSTD
) DISTKEY (c_custkey);
```

8. 기존 테이블로부터 자료를 로드해 주십시요. Note join을 통하여, 스키마가 평탄화됩니다. 이후 통계 갱신을 수행하여 주세요. 
```
INSERT INTO customer_v3(c_custkey, c_name, c_address, c_nationname, c_regionname, c_phone, c_acctbal, c_mktsegment, c_comment)
SELECT c_custkey, c_name, c_address, n_name, r_name, c_phone, c_acctbal, c_mktsegment, c_comment
FROM customer_v2
INNER JOIN nation ON c_nationkey = n_nationkey
INNER JOIN region ON n_regionkey = r_regionkey;
```
```
ANALYZE customer_v3;
```

9. 이 세 가지 버전의 고객 테이블에 대한 스토리지 공간의 차이를 분석하십시오. 열을 추가하면이 압축 열에 필요한 공간이 추가되었으며 압축 알고리즘을 사용하면 차이가 테이블 크기의 2 % 미만입니다.  아래 쿼리는 각 테이블의 각 열이 필요로하는 스토리지 크기와 전체 테이블 크기를 제공합니다. 
```
SELECT
  CAST(d.attname AS CHAR(50)),
  SUM(CASE WHEN CAST(d.relname AS CHAR(50)) LIKE '%v1%'
THEN b.size_in_mb ELSE 0 END) AS size_in_mb_v1,
  SUM(CASE WHEN CAST(d.relname AS CHAR(50)) LIKE '%v2%'
THEN b.size_in_mb ELSE 0 END) AS size_in_mb_v2,
  SUM(CASE WHEN CAST(d.relname AS CHAR(50)) LIKE '%v3%'
THEN b.size_in_mb ELSE 0 END) AS size_in_mb_v3,
  SUM(SUM(CASE WHEN CAST(d.relname AS CHAR(50)) LIKE '%v1%'
THEN b.size_in_mb ELSE 0 END)) OVER () AS total_mb_v1,
  SUM(SUM(CASE WHEN CAST(d.relname AS CHAR(50)) LIKE '%v2%'
THEN b.size_in_mb ELSE 0 END)) OVER () AS total_mb_v2,
  SUM(SUM(CASE WHEN CAST(d.relname AS CHAR(50)) LIKE '%v3%'
THEN b.size_in_mb ELSE 0 END)) OVER () AS total_mb_v3
FROM (
  SELECT relname, attname, attnum - 1 as colid
  FROM pg_class t
  INNER JOIN pg_attribute a ON a.attrelid = t.oid
  WHERE t.relname LIKE 'customer\_v%') d
INNER JOIN (
  SELECT name, col, MAX(blocknum) AS size_in_mb
  FROM stv_blocklist b
  INNER JOIN stv_tbl_perm p ON b.tbl=p.id
  GROUP BY name, col) b
ON d.relname = b.name AND d.colid = b.col
GROUP BY d.attname
ORDER BY d.attname;
```

### Queries
이번 세션에서는, Redshift Query의 모든 것을 다루지는 않지만, 3개의 테이블을 처리하는 단순 쿼리에 대한 예제를 보여줄 것입니다. DW 시스템은 기본적으로 WORM(Write One Read Many) 워크로드를 주로 처리하도록 설계되어 있으며, 해당 시스템에서 어떤 쿼리가 자주 이용되는지 그리고 이를 위한 테이블 설계가 최적화가 이루어져야 합니다. Redshift는 쿼리 성능을 분석할 수 있는 다양한 시스템 테이블(뷰)를 제공합니다. 

10. 첫번째 테이블에서 "Asia" 지역에 있는 고객들이 몇 분인지 확인해 보세요. 
```
SELECT COUNT(c_custkey)
FROM customer_v1 c
INNER JOIN nation n ON c.c_nationkey = n.n_nationkey
INNER JOIN region r ON n.n_regionkey = r.r_regionkey
WHERE r.r_name = 'ASIA';
```

11. 두번째 테이블에서..
```
SELECT COUNT(c_custkey)
FROM customer_v2 c
INNER JOIN nation n ON c.c_nationkey = n.n_nationkey
INNER JOIN region r ON n.n_regionkey = r.r_regionkey
WHERE r.r_name = 'ASIA';
```

12. 마찬가지로 세번째 테이블에서..
```
SELECT COUNT(c_custkey)
FROM customer_v3 c
WHERE c.c_regionname = 'ASIA';
```

13. 각 쿼리의 성능을 분석하십시오. 이 쿼리는 데이터베이스에 대해 마지막으로 실행 된 3 개의 쿼리를 가져오고 마지막 쿼리를 3 개 가져옵니다. 첫 번째 실행은 세 개의 쿼리에서 비슷할 수 있지만 실행을 여러 번 반복하면 (데이터웨어 하우징 환경의 경우) 압축 된 테이블 (두 번째 테이블)을 대상으로하는 쿼리는 최대 약 수행하는 것을 볼 수 있습니다. 비 압축 테이블 (첫 번째 테이블)을 대상으로하는 쿼리보다 40 % 더 우수하고 비정규 화 된 테이블 (세 번째 테이블)을 대상으로하는 쿼리의 속도가 두 번째 테이블에 비해 최대 50 % 향상됩니다 (또는 대략 75 % 향상). 비 압축 테이블에있는 것). (수는 클러스터 토폴로지에 따라 다를 수 있습니다)

```
SELECT query, TRIM(querytxt) as SQL, starttime, endtime, DATEDIFF(microsecs, starttime, endtime) AS duration
FROM STL_QUERY
WHERE TRIM(querytxt) like '%customer%'
ORDER BY starttime DESC
LIMIT 3;
```
 
## Distributing and Sorting
### Distributing Data
Redshift의 스케일을 가능하게하는 주요 기능 중 하나는 노드 전체에서 데이터를 동적으로 슬라이스 할 수 있다는 것입니다. 이는 균일하게 또는 라운드 로빈 방식 (기본적으로 수행됨), 배포 키 (또는 열) 또는 모두에 의해 수행 될 수 있습니다 모든 조각에 모든 데이터를 넣는 분포. 이러한 옵션을 사용하면 쿼리의 병렬 처리 가능성을 최대한 확보하면서, 클러스터에 데이터를 분산시킬 수 있습니다.
쿼리가 빠르게 실행되도록 하려면 정기적으로 조인을 수행하는 테이블에 대하여, 분산 배포를 사용하는 것이 좋습니다. Redshift는 서로 다른 테이블(Entity)에 대하여 동일 키로 분산 배포를 허용하기 때문에, IO 및 네트워크 교환을 줄여줄 수 있습니다. 향후, 다양한 배포 방법과 데이터 및 쿼리 성능에 미치는 영향을 살펴 보겠습니다.
또한 Redshift는 특정 정렬 열을 사용하여 주어진 블록에있는 열의 값을 미리 알고 포함 된 값이 쿼리 범위에 해당하지 않으면 전체 블록을 읽는 것을 건너 뜁니다.
이 샘플에서 쿼리는 고객 관련 정보 (지역)를 기반으로하므로 고객 키가 배포 키에 적합하고 필터가 주문 날짜 범위에 따라 만들어 지므로 정렬 키로 사용하면 실행에 도움이됩니다.

1.  DISTKEY를 customer key 로 SORTKEY를 order date로 둔, 신규 테이블을 생성합니다. 
```
CREATE TABLE orders_v1 (
  o_orderkey int8 NOT NULL PRIMARY KEY                             ,
  o_custkey int8 NOT NULL                                          ,
  o_orderstatus char(1) NOT NULL                                   ,
  o_totalprice numeric(12,2) NOT NULL                              ,
  o_orderdate date NOT NULL                                        ,
  o_orderpriority char(15) NOT NULL                                ,
  o_clerk char(15) NOT NULL                                        ,
  o_shippriority int4 NOT NULL                                     ,
  o_comment varchar(79) NOT NULL
) DISTKEY (o_custkey) SORTKEY(o_orderdate);
```

2. 기존 테이블에서 자료를 가지고 온 후 테이블 통계를 갱신합니다. 
```
INSERT INTO orders_v1
SELECT * FROM orders;
````
```
ANALYZE orders_v1;
```

3. 압축 옵션을 분석하십시오. 이전 항목에서 변경된 것을 볼 수 있습니다. 압축은 데이터가 디스크에 저장 될 때 데이터에 직접 의존하며, 배포 및 정렬 옵션으로 스토리지가 수정됩니다.
```
ANALYZE COMPRESSION orders_v1;
```

### All Together
This last step will use the new distribution and sort keys, and the compression settings proposed by Redshift.

4. Create the orders table using the recommended compression propositions, keeping DISTKEY to customer key and SORTKEY to order date.  Note: Encoding has not been added to the order date field as per the best practices because the order date is used as a sort key.
```
CREATE TABLE orders_v2 (
  o_orderkey int8 NOT NULL PRIMARY KEY ENCODE ZSTD                 ,
  o_custkey int8 NOT NULL ENCODE ZSTD								               ,
  o_orderstatus char(1) NOT NULL ENCODE ZSTD                       ,
  o_totalprice numeric(12,2) NOT NULL ENCODE ZSTD                  ,
  o_orderdate date NOT NULL                                        ,
  o_orderpriority char(15) NOT NULL ENCODE ZSTD                    ,
  o_clerk char(15) NOT NULL ENCODE ZSTD                            ,
  o_shippriority int4 NOT NULL ENCODE ZSTD                         ,
  o_comment varchar(79) NOT NULL ENCODE ZSTD
) DISTKEY (o_custkey) SORTKEY(o_orderdate);
```

5. Import data and build statistics.
```
INSERT INTO orders_v2
SELECT * FROM orders_v1;
```
```
ANALYZE orders_v2;
```

6. Finally, let do one more version using ALL distribution type to put a copy of all the data in every slice of the cluster creating the largest data foot print but putting this data as close to all other data as possible.
```
CREATE TABLE orders_v3 (
  o_orderkey int8 NOT NULL PRIMARY KEY ENCODE ZSTD                     ,
  o_custkey int8 NOT NULL REFERENCES customer_v3(c_custkey) ENCODE ZSTD,
  o_orderstatus char(1) NOT NULL ENCODE ZSTD                           ,
  o_totalprice numeric(12,2) NOT NULL ENCODE ZSTD                      ,
  o_orderdate date NOT NULL SORTKEY                                    ,
  o_orderpriority char(15) NOT NULL ENCODE ZSTD                        ,
  o_clerk char(15) NOT NULL ENCODE ZSTD                                ,
  o_shippriority int4 NOT NULL ENCODE ZSTD                             ,
  o_comment varchar(79) NOT NULL ENCODE ZSTD
) DISTSTYLE ALL;
```

7. Import data and build statistics.
```
INSERT INTO orders_v3
SELECT * FROM orders_v2;
```
```
ANALYZE orders_v3;
```

### Storage Analysis
As for the customers, this query will analyze the storage used by the four representations of the orders table.

8. Analyze the difference in storage space for these 3 versions of the order table. Compression allows a 50% to 60% storage reduction on the data. The third version is the largest amount of data as it stores all data in this table in all slices of the cluster.   If you wish to learn more detailed information about distributing data in a redshift cluster please see the following:
http://docs.aws.amazon.com/redshift/latest/dg/t_Distributing_data.html.
This query gives you the storage requirements per column for each table, then the total storage for the table (repeated identically on each line).
```
SELECT
  CAST(d.attname AS CHAR(50)),
  SUM(CASE WHEN CAST(d.relname AS CHAR(50)) LIKE '%v1%'
THEN b.size_in_mb ELSE 0 END) AS size_in_mb_v1,
  SUM(CASE WHEN CAST(d.relname AS CHAR(50)) LIKE '%v2%'
THEN b.size_in_mb ELSE 0 END) AS size_in_mb_v2,
  SUM(CASE WHEN CAST(d.relname AS CHAR(50)) LIKE '%v3%'
THEN b.size_in_mb ELSE 0 END) AS size_in_mb_v3,
  SUM(SUM(CASE WHEN CAST(d.relname AS CHAR(50)) LIKE '%v1%'
THEN b.size_in_mb ELSE 0 END)) OVER () AS total_mb_v1,
  SUM(SUM(CASE WHEN CAST(d.relname AS CHAR(50)) LIKE '%v2%'
THEN b.size_in_mb ELSE 0 END)) OVER () AS total_mb_v2,
  SUM(SUM(CASE WHEN CAST(d.relname AS CHAR(50)) LIKE '%v3%'
THEN b.size_in_mb ELSE 0 END)) OVER () AS total_mb_v3
FROM (
  SELECT relname, attname, attnum - 1 as colid
  FROM pg_class t
  INNER JOIN pg_attribute a ON a.attrelid = t.oid
  WHERE t.relname LIKE 'orders\_v%') d
INNER JOIN (
  SELECT name, col, MAX(blocknum) AS size_in_mb
  FROM stv_blocklist b
  INNER JOIN stv_tbl_perm p ON b.tbl=p.id
  GROUP BY name, col) b
ON d.relname = b.name AND d.colid = b.col
GROUP BY d.attname
ORDER BY d.attname;
```
### Queries
The query execution speed is also impacted by the distribution settings. This last part will issue the same query on the four versions of the order table, and analyze the time taken to execute these queries.

9. Get, for the year 1995, some information on the orders passed by the customers depending on their market segment, in Asia. This query is for the first table.
```
SELECT c_mktsegment, COUNT(o_orderkey) AS orders_count,
AVG(o_totalprice) AS medium_amount,
SUM(o_totalprice) AS orders_revenue
FROM orders_v1 o
INNER JOIN customer_v3 c ON o.o_custkey = c.c_custkey
WHERE o_orderdate BETWEEN '1995-01-01' AND '1995-12-31' AND
c_regionname = 'ASIA'
GROUP BY c_mktsegment;
```

10. Same query for the second table.
```
SELECT c_mktsegment, COUNT(o_orderkey) AS orders_count,
AVG(o_totalprice) AS medium_amount,
SUM(o_totalprice) AS orders_revenue
FROM orders_v2 o
INNER JOIN customer_v3 c ON o.o_custkey = c.c_custkey
WHERE o_orderdate BETWEEN '1995-01-01' AND '1995-12-31' AND
c_regionname = 'ASIA'
GROUP BY c_mktsegment;
```

11. For the third table. You will notice that the order of results has changed. This is due to the change in sorting and distribution, since we did not order the resultset (no ORDER clause), the “natural” (storage) order applies.
```
SELECT c_mktsegment, COUNT(o_orderkey) AS orders_count,
AVG(o_totalprice) AS medium_amount,
SUM(o_totalprice) AS orders_revenue
FROM orders_v3 o
INNER JOIN customer_v3 c ON o.o_custkey = c.c_custkey
WHERE o_orderdate BETWEEN '1995-01-01' AND '1995-12-31' AND
c_regionname = 'ASIA'
GROUP BY c_mktsegment;
```

12. Analyze the performances of each query. This query gets the 3 last queries ran against the database. The results go up to around 75% query time improvement with the right distribution, sort and compression schemes (orders_v1 vs orders_v2).  The ALL distribution (v3) should really be used only if a dimension table cannot be collocated with the fact table or other important joining tables, you can improve query performance significantly by distributing the entire table to all of the nodes. Using ALL distribution multiplies storage space requirements and increases load times and maintenance operations, so you should weigh all factors before choosing ALL distribution. (The numbers will vary depending on the cluster topology)
```
SELECT query, TRIM(querytxt) as SQL, starttime, endtime, DATEDIFF(microsecs, starttime, endtime) AS duration
FROM STL_QUERY
WHERE TRIM(querytxt) like '%orders_v%JOIN%'
ORDER BY starttime DESC
LIMIT 3;
```

## Result Set Caching and Execution Plan Reuse
Redshift enables a result set cache to speed up retrieval of data when it knows that the data in the underlying table has not changed.  It can also re-use compiled query plans when only the predicate of the query has changed.

1. Execute the following query and note the query execution time.  Since this is the first execution of this query Redshift will need to compile the query as well as cache the result set.
```
SELECT c_mktsegment, o_orderpriority, sum(o_totalprice)
FROM Customer_v3 c
JOIN Orders_v2 o on c.c_custkey = o.o_custkey
GROUP BY c_mktsegment, o_orderpriority;
```

2. Execute the same query a second time and note the query execution time.  In the second execution redshift will leverage the result set cache and return immediately.
```
SELECT c_mktsegment, o_orderpriority, sum(o_totalprice)
FROM Customer_v3 c
JOIN Orders_v2 o on c.c_custkey = o.o_custkey
GROUP BY c_mktsegment, o_orderpriority;
```

3. Update data in the table and run the query again. When data in an underlying table has changed Redshift will be aware of the change and invalidate the result set cache associated to the query.  Note the execution time is not as fast as Step 2, but faster than Step 1 because while it couldn’t re-use the cache it could re-use the compiled plan.
```
UPDATE customer_v3
SET c_mktsegment = c_mktsegment
WHERE c_mktsegment = 'MACHINERY';
```
```
VACUUM DELETE ONLY customer_v3;
```
```
SELECT c_mktsegment, o_orderpriority, sum(o_totalprice)
FROM Customer_v3 c
JOIN Orders_v2 o on c.c_custkey = o.o_custkey
GROUP BY c_mktsegment, o_orderpriority;
```

4. Execute a new query with a predicate and note the query execution time.  Since this is the first execution of this query Redshift will need to compile the query as well as cache the result set.
```
SELECT c_mktsegment, count(1)
FROM Customer_v3 c
WHERE c_mktsegment = 'MACHINERY'
GROUP BY c_mktsegment;
```
5. Execute the query with a slightly different predicate and note that the execution time is faster than the prior execution even though a very similar amount of data was scanned and aggregated.  This behavior is due to the re-use of the compile cache because only the predicate has changed.  This type of pattern is typical for BI reporting where the SQL pattern remains consistent with different users retrieving data associated to different predicates.
```
SELECT c_mktsegment, count(1)
FROM Customer_v3 c
WHERE c_mktsegment = 'BUILDING'
GROUP BY c_mktsegment;
```

## Selective Filtering
Redshift takes advantage of zone maps which allows the optimizer to skip reading blocks of data when it knows that the filter criteria will not be matched.   In the case of the orders_v3 table, because we have defined a sort key on the o_order_date, queries leveraging that field as a predicate will return much faster.

6. Execute the following two queries noting the execution time of each.  The first query is to ensure the plan is compiled.  The second has a slightly different filter condition to ensure the result cache cannot be used.
```
SELECT count(1), sum(o_totalprice)
FROM orders_v3
WHERE o_orderdate between '1992-07-05' and '1992-07-07';
```
```
SELECT count(1), sum(o_totalprice)
FROM orders_v3
WHERE o_orderdate between '1992-07-07' and '1992-07-09';
```
7. Execute the following two queries noting the execution time of each.  The first query is to ensure the plan is compiled.  The second has a slightly different filter condition to ensure the result cache cannot be used. You will notice the second query takes significantly longer than the second query in the previous step even though the number of rows which were aggregated is similar.  This is due to the first query's ability to take advantage of the Sort Key defined on the table.
```
SELECT count(1), sum(o_totalprice)
FROM orders_v3
where o_orderkey < 600001;
```
```
SELECT count(1), sum(o_totalprice)
FROM orders_v3
where o_orderkey < 600002;
```

## Join Strategies
Because or the distributed architecture of Redshift, in order to process data which is joined together, data may have to be broadcast from one node to another.  It’s important to analyze the explain plan on a query to identify which join strategies is being used and how to improve it.

8. Execute an EXPLAIN on the following query.  If you recall, both of these tables are distributed on the custkey.  This results in a join strategy of “Hash Join DS_DIST_NONE” and a relatively low overall “cost”.
```
EXPLAIN
SELECT c_mktsegment, o_orderpriority, sum(o_totalprice)
FROM Customer_v3 c
JOIN Orders_v2 o on c.c_custkey = o.o_custkey
GROUP BY c_mktsegment, o_orderpriority;
```

9. Execute an EXPLAIN on the following query.  If you recall, this version of the orders is distributed on the orderkey.  This results in a join strategy of “Hash Join DS_BCAST_INNER” and a relatively high overall “cost”.
```
EXPLAIN
SELECT c_mktsegment, o_orderpriority, sum(o_totalprice)
FROM Customer_v3 c
JOIN Orders o on c.c_custkey = o.o_custkey
GROUP BY c_mktsegment, o_orderpriority;
```

10. Create a new version of the orders tables which is both distributed and sorted on the values as the customer table.  Execute an EXPLAIN and notice this results in a join strategy of “Merge Join DS_DIST_NONE” with the lowest cost of the three.
```
CREATE TABLE orders_v4
DISTKEY(o_custkey) SORTKEY (o_custkey) as
SELECT * FROM orders_v2;
```
```
EXPLAIN
SELECT c_mktsegment, o_orderpriority, sum(o_totalprice)
FROM Customer_v3 c
JOIN Orders_v4 o on c.c_custkey = o.o_custkey
GROUP BY c_mktsegment, o_orderpriority;
```

11. Execute an EXPLAIN plan on the following query which is missing the join condition.  This results in a join strategy of “XN Nested Loop DS_BCAST_INNER” and throws a warning about the cartesian product.  
```
EXPLAIN
SELECT * FROM region, nation;
```

## Before You Leave
If you are done using your cluster, please think about decommissioning it to avoid having to pay for unused resources.
