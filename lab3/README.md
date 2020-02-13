# LAB 3 - Table Design and Query Tuning
이번 Lab을 통하여, Table 압축, 반정규화, Dist(분산) 키 구성 및 Sort(정렬) Key 구성에 따른 영향도를 살펴볼 예정입니다. 

## Contents
* [시작하기 전에.](#before-you-begin)
* [컬럼 압축 및 반정규화](#compressing-and-de-normalizing)
* [레코드 분산 및 정렬](#distributing-and-sorting)
* [결과 캐쉬 및 실행 Plan 재활용](#result-set-caching-and-execution-plan-reuse)
* [선택적인 필터링](#selective-filtering)
* [조인 전략](#join-strategies)
* [Before You Leave](#before-you-leave)
 
## 시작하기 전에.
해당 Lab은 이미 Redshift Cluster를 기동했다고 가정하고 있으며, 이후, Sample Table에 대한 Schema생성 및 데이터 로드가 되어 있어야 합니다. 만약 Redshift Cluster가 기동되어 있지 않고, 자료 Loading이 되어 있지 않다면 [LAB 2 - Data Loading](../lab2/README.md)를 참조하여 설치하시기 바랍니다.
* [Your-Redshift_Port]
* [Your-Redshift_Username]
* [Your-Redshift_Password]

또한, Client Tool등을 이용하여, 해당 Redshift에 대한 접근이 되어 있어야 합니다. SQL Workbench/J에 대한 설치/설정 방법은 [Lab 1 - Creating Redshift Clusters : Configure Client Tool](../lab1/README.md#configure-client-tool)를 참고하시기 바랍니다. 이에 대한 또다른 대안으로, Redshift가 제공하는 Query Editor를 이용할 경우, 별도의 설치가 필요없습니다. 
```
https://console.aws.amazon.com/redshift/home?#query:
```
## 압축 및 반정규화
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
마지막 진행 내용은 레코드의 분산 및 정렬 키를 활용하는 것입니다. 물론 맢에서 나왔던 Redshift가 권고하는 압축 설정도 같이 활용할 예정입니다. 

4. 권장 압축 제안을 사용하여 DISTKEY를 고객 키에, SORTKEY를 주문 날짜로 유지하여 주문 테이블을 작성하십시오. 참고 : 주문 날짜는 정렬 키로 사용되므로 모범 사례에 따라 주문 날짜 필드에 인코딩이 추가되지 않았습니다.
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

5. 기존 테이블에서 자료를 Import하고, 통계를 돌려주십시요. 
```
INSERT INTO orders_v2
SELECT * FROM orders_v1;
```
```
ANALYZE orders_v2;
```

6. 마지막으로, "ALL" 분산 형태를 활용해 보도록 하겠습니다. "ALL" 분산 형태는 모든 자료를 모든 클러스터의 슬라이스에 할당하는 방식으로, 매우 큰 용량을 생성할 수 있습니다만, 다른 데이터들과 밀접하게 동일 위치에 묶을 수 있습니다. 
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

7. 데이터를 로드하고, 통계치를 생성합니다. 
```
INSERT INTO orders_v3
SELECT * FROM orders_v2;
```
```
ANALYZE orders_v3;
```

### 스토리지 분석
이 쿼리는 orders 테이블의 네 가지 표현에 사용 된 스토리지를 분석합니다.

8. 이 3 가지 주문 테이블 버전의 스토리지 공간 차이를 분석하십시오. 압축을 통해 데이터의 저장 공간을 50 % ~ 60 % 줄일 수 있습니다. 세 번째 버전은이 테이블의 모든 데이터를 클러스터의 모든 슬라이스에 저장하므로 가장 많은 양의 데이터입니다. 레드 시프트 클러스터에 데이터를 배포하는 방법에 대한 자세한 내용을 보려면 다음을 참조하십시오.

http://docs.aws.amazon.com/redshift/latest/dg/t_Distributing_data.html.
이 쿼리는 각 테이블의 Column이 차지하는 스토리지 요구 사항을 보여주고, 이후 각 테이블의 총 사용율 (각 라인에서 동일하게 반복됨) 제공합니다.
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
쿼리 실행 속도는 분산 형태 설정의 영향을 받습니다. 이 마지막 부분은 네 가지 버전의 주문 테이블에서 동일한 쿼리를 발행하고 이러한 쿼리를 실행하는 데 걸린 시간을 분석합니다.

9. 1995년 대상으로, 'Asia'  마켓 세그먼트에서 커스터머가 주문한 주문 건에 대한 평균 주문 가격, 전체 주문 가격, 건수를 구해 보십시요. 이 쿼리를 첫번째 테이블에서 수행한다면, 아래와 같이 보일 것입니다. 
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

10. 통일한 내용을 두번째 버젼에서 실행하면 아래와 같습니다. 
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

11. 세번째 테이블에서 돌리기 전에, 주문 순선가 변경된 것을 확인할 수 있습니다. 이는 분산 키와 정렬키를 적용했기 때문인데, 쿼리에 order by 절이 없기 때문에, Database에 들어가 있는 데이터 적재, 처리 순으로 나온다는 것을 확인할 수 있습니다. 
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

12. 각 쿼리의 성능을 분석하십시오. 이 쿼리는 데이터베이스에 대해 마지막으로 실행 된 3 개의 쿼리를 가져옵니다. 결과는 올바른 분배, 정렬 및 압축 체계 (orders_v1 대 orders_v2)를 사용하여 쿼리 시간이 약 75 % 향상됩니다. 차원 테이블을 팩트 테이블 또는 기타 중요한 조인 테이블과 함께 배치 할 수없는 경우에만 ALL 분배(v3)를 사용해야합니다. 전체 테이블을 모든 노드에 분배하여 쿼리 성능을 크게 향상시킬 수 있습니다. ALL 분배를 사용하면 스토리지 공간 요구 사항이 증가하고 로드 시간 및 유지 보수 조작이 증가하므로 ALL 분배를 선택하기 전에 모든 요소를 평가해야합니다. (수는 클러스터 토폴로지에 따라 다릅니다)
```
SELECT query, TRIM(querytxt) as SQL, starttime, endtime, DATEDIFF(microsecs, starttime, endtime) AS duration
FROM STL_QUERY
WHERE TRIM(querytxt) like '%orders_v%JOIN%'
ORDER BY starttime DESC
LIMIT 3;
```

## 결과 캐쉬 및 실행 계획 재활용 
Redshift는 기본 테이블의 데이터가 변경되지 않았다고 판단할 경우, 결과 세트 캐시를 통하여 데이터 검색 속도를 높일 수 있습니다. 쿼리의 술어만(Where 조건 절의 Parameter) 변경된 경우 컴파일 된 쿼리 계획을 재사용 할 수도 있습니다.

1. 다음 쿼리를 실행하고 쿼리 실행 시간을 기록하십시오. 이것이이 쿼리의 첫 번째 실행이므로 Redshift는 쿼리를 컴파일하고 결과 세트를 캐시해야합니다.
```
SELECT c_mktsegment, o_orderpriority, sum(o_totalprice)
FROM Customer_v3 c
JOIN Orders_v2 o on c.c_custkey = o.o_custkey
GROUP BY c_mktsegment, o_orderpriority;
```

2. 동일한 쿼리를 두 번 실행하고 쿼리 실행 시간을 기록하십시오. 두 번째 실행에서 redshift는 결과 세트 캐시를 활용하고 즉시 리턴합니다.
```
SELECT c_mktsegment, o_orderpriority, sum(o_totalprice)
FROM Customer_v3 c
JOIN Orders_v2 o on c.c_custkey = o.o_custkey
GROUP BY c_mktsegment, o_orderpriority;
```

3. 테이블의 데이터를 업데이트하고 쿼리를 다시 실행하십시오. 기본 테이블의 데이터가 변경되면 Redshift는 변경 사항을 인식하고 쿼리와 관련된 결과 집합 캐시를 무효화합니다. 실행 시간은 2 단계만큼 빠르지는 않지만 1 단계보다 빠릅니다. 캐시를 재사용 할 수는 없지만 컴파일 된 계획을 재사용 할 수 있기 때문입니다.
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

4. 술어를 사용하여 새 쿼리를 실행하고 쿼리 실행 시간을 기록하십시오. 이것이이 쿼리의 첫 번째 실행이므로 Redshift는 쿼리를 컴파일하고 결과 세트를 캐시해야합니다.
```
SELECT c_mktsegment, count(1)
FROM Customer_v3 c
WHERE c_mktsegment = 'MACHINERY'
GROUP BY c_mktsegment;
```
5. 약간 다른 술어를 사용하여 쿼리를 실행하고 매우 유사한 양의 데이터를 스캔하고 집계하더라도 실행 시간이 이전 실행보다 빠릅니다. 술어만 변경 되었기 때문에이 동작은 컴파일 캐시의 재사용으로 인해 발생합니다. 이 유형의 패턴은 SQL 패턴이 다른 술어와 연관된 데이터를 검색하는 다른 사용자와 일관성을 유지하는 BI보고에 일반적입니다.
```
SELECT c_mktsegment, count(1)
FROM Customer_v3 c
WHERE c_mktsegment = 'BUILDING'
GROUP BY c_mktsegment;
```

## 선택적인 필터링
Redshift는 영역 맵을 활용하므로 필터 기준이 일치하지 않을 경우 옵티마이 저가 데이터 블록 읽기를 건너 뛸 수 있습니다. orders_v3 테이블의 경우 o_order_date에 정렬 키를 정의 했으므로 해당 필드를 술어로 사용하는 쿼리가 훨씬 빠르게 리턴됩니다.

6. 각각의 실행 시간을 나타내는 다음 두 쿼리를 실행하십시오. 첫 번째 쿼리는 계획이 컴파일되었는지 확인하는 것입니다. 두 번째는 결과 캐시를 사용할 수 없도록 필터 조건이 약간 다릅니다.
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
7. 다음 두 쿼리를 실행하고 실행 시간을 기록하십시오. 첫번째 쿼리를 통하여, 실행 계획이 확실하게 컴파일 되었다는 것을 보장 받을 수 있습니다. 두 번째 쿼리를 첫번째 쿼리와는 살짝 Where 조건문이 다릅니다. 이로 인하여 결과 캐쉬는 적용되지 않습니다. 따라서 이전 결과 캐쉬가 적용된 쿼리보다는 늦은 응답 속도를 느낄 수 있습니다. 
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

## Join 전략
Redshift가 가지는 분산 아키텍처로 인하여, 두가지 이상의 데이터가 같이 처리되어야 하는 Join 프로세스를 위하여, 자료가 하나의 노드에서 다른 노드로 일부 또는 전체적으로 다시 전파될 수 있습니다. 이는 Query의 실행계획을 분석하여, 어떤 Join 전략이 이용되는지 확인하고 성능 개선을 시도할 때 중요한 단서가 됩니다. 

8. EXPLAIN 명령을 다른 Query에서 수행하십시요. 앞서 수행된 내용을 되돌아보면, 이 두 테이블이 모두 같은 custkey 컬럼을 분산키로 사용했다는 것을 기억할 수 있습니다. 이 결과는 “Hash Join DS_DIST_NONE” 이며, 상대적으로 매우 적은 비용을 지불합니다. 
```
EXPLAIN
SELECT c_mktsegment, o_orderpriority, sum(o_totalprice)
FROM Customer_v3 c
JOIN Orders_v2 o on c.c_custkey = o.o_custkey
GROUP BY c_mktsegment, o_orderpriority;
```

9. 다음 쿼리에 대하여, EXPLAIN 명령을 수행해 보십시요.  쿼리에 사용된 orders 테이블은 orderkey를 분산키로 사용한 경우입니다. 사용된 Join 전략은 “Hash Join DS_BCAST_INNER” 이며, 상대적으로 높은 비용을 지불하게 됩니다.
```
EXPLAIN
SELECT c_mktsegment, o_orderpriority, sum(o_totalprice)
FROM Customer_v3 c
JOIN Orders o on c.c_custkey = o.o_custkey
GROUP BY c_mktsegment, o_orderpriority;
```

10. 새로운 orders 테이블을 만들어 보도록 하겠습니다. Customer 테이블과 동일하게 분산키와 정렬키를 사용해 보도록 하겠습니다. 이후, Explain 문을 실행하고, Join 전략이 변경된 것을 확인해 보십시요. 사용된 Join 전략은 “Merge Join DS_DIST_NONE” 이며, 최저 수준의 비용만을 요구합니다. 
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

11. 다음 Query를 Explain으로 확인해 보십시요. 이 Query는 두 테이블간의 결합 조건이 없기 때문에, Cartesian product라는 두 테이블의 레코드 쌍 전체가 나오게 됩니다. 이럴 경우, Join전략은 “XN Nested Loop DS_BCAST_INNER” 이 선택되며, Cartesian Product 발생이라는 경고 문구도 같이 나옵니다. 
```
EXPLAIN
SELECT * FROM region, nation;
```

## 떠나기 전에
더이상 클러스터 사용을 할 필요가 없다면, 불필요한 예산 낭비를 막기 위해서 사용한 모든 Resource를 Console상에서 제거하시기 바랍니다.
