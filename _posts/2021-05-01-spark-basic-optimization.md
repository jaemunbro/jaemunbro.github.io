---
title: Apache Spark - Core Optimization(스파크 최적화)
author: Jaemun Jung
date: 2021-05-01 01:43:00 +0900
categories: [Apache Spark]
tags: [Spark, Optimization]
---

스파크의 기본적인 최적화 세팅 방법들에 대해 정리해보자.
본문 작성에 주되게 참조한 소스는 Spark + AI Summit Europe 2019에서 발표한 Databricks의 Daniel Tomes의 세션인 [Apache Spark Core – Practical Optimization](https://databricks.com/session_eu19/apache-spark-core-practical-optimization)이다.


# General Optimization
--------------
아래 Advanced Optimization에서 조금 깊이 파보기 전에, 기본적인 최적화 방법들에 대해서 간단히 살펴보자.

### File Format 최적화 
- Parquet, ORC와 같은 columnar file 활용
- splittable file format 사용 (snappy, bzip2와 같은 compression format)
- 너무 많은 small file은 compaction 한다.

### Parallelism
- 데이터 사이즈에 맞춘 스파크 파티션 생성. 너무 적은 파티션 수는 executor를 idle하게 만들 수 있고 반대로 너무 많은 파티션은 task scheduling 오버헤드를 지나치게 높게 만들 수 있다.
- executor 최소한 2-3개 이상의 코어를 할당
- `spark.sql.shuffle.partitions` 기본값 200은 빅데이터 프로덕션 환경에서는 너무 작으므로 기본적으로 튜닝이 필요하다.

### Shuffle 줄이기
shuffle은 네트워크와 disk I/O를 포함하는 노드간 데이터 이동을 불러일으키는 가장 비싼 연산이다. shuffle을 줄이는 것이 튜닝의 시작이다.
-  `spark.sql.shuffle.partitions`을 튜닝한다.
- 각 task 사이즈가 너무 저지지 않도록 input data를 파티셔닝한다.
- 일반적으로, 파티션당 100MB에서 200MB 사이를 타게팅하도록 한다. 
- `spark.sql.shuffle.partitions` = (shuffle stage input size/target size)/total cores) * total cores

### Filter/Reduce dataSet size
- partition된 데이터를 활용한다.
- 최대한 이른 stage에서 데이터를 filter out 한다. 

### 적절한 Cache 사용
- pipeline에서 여러번 계산이 필요한 df에서 활용한다.
- unpersist를 하자.

### Join
- BroadcastHashJoin을 최대한 활용하자. 가장 성능이 빠른 Join이다.

### Cluster 리소스 튜닝
-  spark.driver.memory, executor-memory, num-executors, and executor-cores 튜닝

### 비싼 연산은 피하자
- order by는 필수적인 경우만 쓴다.
- 모든 컬럼을 선택하기 위한 (*)는 지양하고, 필요한 컬럼을 선택해서 쓴다.
- count()도 꼭 필요한 경우에만.

### Skew
- salting 통해 해결 (아래 Advanced Optimization 참조)
- partition이 불균형한 경우, 균형잡힌 파티션을 재생성하기 위해 repartition을 하고 cache해두는 것을 고려해볼 수도 있다
- SparkUI에서 partition size와 task duration을 확인하자

### UDF
- UDF 사용은 최대한 지양한다. (아래 Advanced Optimization 참조)

# Advanced Optimization
## 하드웨어 스펙에 대한 이해
-----------
- **Core Count & Speed**
- **Memory Per Core** (메모리당 코어가 얼마나 되는가)
- local disk type, count, size, speed 
- network speed, toplology
- cost / core / hour

## Get A Baseline
-----------
- Is your action efficient?
    - Long Stages, Spills, Laggard Tasks  
    duration으로 sorting해서 가장 오래 돌고 있는 Stage를 파악하는 것부터 시작할 수 있다.  
    disk spill이 심하게 일어나고 있는 Stage인 경우를 찾는다. 보통 가장 오래 도는 Job과 동일한 경우가 많다.

- CPU Utilization
    - Ganglia / Yarn  
    CPU 활용률이 너무 낮다면, CPU 활용률이 70% 내외로 활용될 수 있도록 타겟으로 잡아볼 수 있다.

## 데이터 스캔 최소화 - Minimize Data Scans
-----------
- 가장 간단하지만 가장 중요한 최적화 기법  
**Lazy loading** - 데이터를 파티셔닝을 최대한 필터하고 로딩해서 사용한다.
- Hive Partition  
**Partition pruning**을 통한 스캔 대상 데이터 최소화
- Bucketing (Only for experts - 제대로 유지하는 것이 거의 불가능에 가까움..)

## Spark Partitions - Input/Shuffle/Output
-----------
스파크 파티션을 컨트롤하는 두가지 핵심은
1. **Spill을 피하도록 한다**
2. **최대한의 Core를 활용할 수 있도록 한다**
이다. 데이터의 Input 단계부터 Shuffle 단계, Output 단계로 나눠서 알아보자.
#### (1) Input  
    - Size를 컨트롤한다.  
    `spark.sql.file.maxPartitionBytes` default value : 128MB  
        - Increase Parallelism: 코어의 활용률을 병렬도를 더 올리기 위해 쪼개서 읽을 수 있다. 
        예시)
        ![image](https://user-images.githubusercontent.com/29077671/116790207-11a69400-aaee-11eb-8b26-79e65539e52c.png)
        shuffle이 없는 map job이므로 오직 read/write 속도에만 dependant한 task인데, 더 빠르게 처리하기 위해 maxPartitionSize를 core 수에 맞게 튜닝했고 그것만으로 시간을 50% 줄일 수 있었다. 더 많은 코어가 나누어서 처리했기 때문이다.
        - Heavily Nested/Repetitive Data: 메모리에서 풀어냈을때 데이터가 커질 수 있으므로 더 작게 읽는 것이 필요할 수도 있다.
        - Generating Data - Explode: 이 역시 새로운 데이터 컬럼을 만들면서 데이터가 메모리 상에서 커질 수 있으므로. 
        - Source Structure is not optimal(upstream)
        - UDFs  

#### (2) Shuffle  
    - Count를 컨트롤한다.
    - `spark.sql.shuffle.partitions` default value : 200
    - 가장 큰 셔플 스테이지의 타겟 사이즈를 파티션당 200MB 이하로 잡는 것이 적절하다.   
    (target size <= 200 MB/partition)    
        - 예시를 들어보자.
        Shuffle Stage Input = 210GB  
        210000MB / 200MB = 1050
        -> `spark.conf.set("spark.sql.shuffle.partitions", 1050)`  
        하지만 만약 클러스터의 가용 코어가 2000 이라면 다 활용하면 더 좋다.  
        -> `spark.conf.set("spark.sql.shuffle.partitions", 2000)`    

        - 또 다른 예시 - spark history server의 stage 모니터링
        ![image](https://user-images.githubusercontent.com/29077671/116789923-7c56d000-aaec-11eb-9fc7-0e7b1fcbe557.png)
        위 예시의 우측 Summary Metrics를 보면 Median 값이 270MB이다. 아주 나쁘진 않지만 그래도 파티션이 부족하다고 봐야한다.
        위의 예시에서 stage 19와 stage 20이 stage 21의 셔플의 소스이므로, 19와 20의 shuffle write이 21의 shuffle input이 라고 볼 수 있다.
        간단하게 이야기해서, shuffle spill(memory/disk)가 일어나고 있다면 파티션이 더 필요하다고 이해해도 좋다.
        - partition count = stage input data / target size

#### (3) Output  
    - Count를 컨트롤한다.
    - coalesce(n) : 2000개의 partition이 shuffle하고 나서, write할 때 100개가 나눠서 할 수 있도록.
    - Repartition(n) : partition을 증가시킬 때 사용. shuffle을 발생시키므로 꼭 필요한 경우가 아니면 사용하지 말자.
    - df.write.option("maxRecordsPerFile", N) : 추천하는 방법은 아님.
    ![image](https://user-images.githubusercontent.com/29077671/116790347-998c9e00-aaee-11eb-85be-baf026979fc3.png)
    전체 160GB의 파일을 처리하는데 10개의 코어가 1.6GB씩 처리할 때와 100개의 코어가 처리할 때의 차이는 아주 크다.

## Skew Join Optimization
---------
어떤 파티션이 다른 파티션보다 훨씬 큰 경우.
예시) 어떤 키 하나에는 수백만개의 count가 몰려 있고, 나머지 키에는 수십개 정도씩만 있다고 가정하자.
![image](https://user-images.githubusercontent.com/29077671/116791009-85e33680-aaf2-11eb-9ba1-a4cd5196d7b2.png)
**Salting**이라 불리는 방법을 통해 해결해볼 수 있다. 노이즈 데이터를 뿌려넣는다는 의미이다.
- 0부터 spark.sql.suffle.partitoins - 1 사이의 random int값을 가진 column을 양쪽 데이터프레임에 생성한다. 
- Join 조건에 새로운 salting column을 포함한다.
- result 표출 시 salting column을 drop한다.
```scala
df.groupBy("city", "state").agg(<f(x)>).orderBy(co.desc)
val saltVal = random(0, spark.conf.get(org...shuffle.partitions) - 1)
df.withColumn("salt", lit(saltVal))
    .groupBy("city", "state", "salt")
    .agg(<f(x)>)
    .drop("salt")
    .orderBy(col.desc)
```

## Minimize Data Scans (Persistence)
-----------
- Persistence는 무료가 아니다.
- 반복작업이 있는 경우에 사용한다.
- 기본 `df.cache == df.persist(StorageLevel.MEMORY_AND_DISK)`  
- 다른 타입들의 장단점에 대해서도 알아보자. 
    - Default StorageLevel.MEMORY_AND_DISK  
    (deserialized)
    - deserialized = Faster = Bigger
    - serialized = Slower = Smaller
- **df.unpersist 를 잊지말자!!**  
cache memory를 사용하는 만큼 working memory가 줄어들 것이므로.

## Join Optimization
-----------
- SortMerge Join : 양쪽 데이터가 다 클 때
- Broadcast Join : 한쪽 데이터가 작을때
    - 옵티마이저에 의해 자동으로 활성화됨 (one side < spark.sql.autoBroadcastJoinThreshold) 기본값은 10M.
    - Risk
        - Not Enough Driver Memory
        - DF > spark.driver.maxResultSize
        - DF > single executor available working memory
    - ![image](https://user-images.githubusercontent.com/29077671/116790846-8a5b1f80-aaf1-11eb-9ccf-f82a30430e94.png)
        - broadcast join을 조심해야하는 이유 : driver에서 hashmap format datafame으로 바꿔서 각 executor로 보낸다. hashmap을 만들면서 기존에 compression이 되어 있던 것도 풀리므로 데이터 사이즈도 처음 partition 안에 있을 때보다 몇배 커진 상태가 될 수 있다. (e.g., 126MB -> 270MB)

## BroadcastNestedLoopJoin (BNLJ)
-----------
SparkSQL에서는 NOT IN 조건 대신 NOT EXISTS를 사용하라. 훨씬 효율적이다.
![image](https://user-images.githubusercontent.com/29077671/116791238-db6c1300-aaf3-11eb-854b-fe73059567b4.png)

## 비싼 오퍼레이션을 제외시키자 - Omit Expensive Ops
-----------
튜닝의 시작을 IntelliJ에서 아래 키워드들을 찾는 것부터 해볼 수도 있다.
- Repartition  
coalesce를 쓰거나 shuffle partition count 세팅을 바꿔서 써라.
- count()  
진짜 필요한 경우에만 써라.  
습관적으로 많이 쓰긴 하지만, 프로덕션 배포 전에 제외시키는 걸 잊지 말자!
- distinctCount
-> 꼭 정확한 숫자가 필요한 것이 아니라면 가능하면 approxCountDistinct()를 써라. 2% 내외의 오차가 발생한다.
- distinct
    - distinct 대신 dropDuplicates을 써라.
    - dropDuplicates를 JOIN 전에 써라.
    - dropDuplicates를 groupBy 전에 써라.

### UDF penalties
- UDF 사용은 최대한 피하는게 좋다.
- 일반적으로 scala udf가 python udf보다 빠르다. r udf는 가~끔 작동한다.
- org.apache.spark.sql.functions를 최대한 활용하자.
- UDF 만들기 전에 PandasUDF(vectorized UDFs)에 원하는 기능이 없는지 알아보자. 

### Advanced Parallelism
- Driver Parallelism  
델타 처리를 위한 예시
![image](https://user-images.githubusercontent.com/29077671/116791534-0f483800-aaf6-11eb-9cd8-fd0d86925c62.png)
- Horizontal Parallelsim
- Executor Parallelism

#### 더 알아보기 좋은 글
GC https://databricks.com/blog/2015/05/28/tuning-java-garbage-collection-for-spark-applications.html


## Reference
[https://databricks.com/session_eu19/apache-spark-core-practical-optimization](https://databricks.com/session_eu19/apache-spark-core-practical-optimization)
[https://developer.ibm.com/technologies/artificial-intelligence/blogs/spark-performance-optimization-guidelines/](https://developer.ibm.com/technologies/artificial-intelligence/blogs/spark-performance-optimization-guidelines/)

