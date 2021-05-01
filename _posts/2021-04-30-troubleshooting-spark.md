---
title: Apache Spark - Troubleshooting Cheatsheet (스파크 트러블슈팅 가이드)
author: Jaemun Jung
date: 2021-04-30 01:43:00 +0900
categories: [Apache Spark]
tags: [Spark, Troubleshooting]
---

스파크의 문제 사례들과 그 해결 방법들에 대해 알아보자.  
문제 케이스들은 일부 직접 겪은 것들과 본문 하단 링크의 케이스들을 취합하였다.

# Troubleshooting Tips
-----------
### 트러블슈팅을 위해 verbose mode를 활용하자
-----------
`spark-submit --driver-memory 10g --verbose --master yarn --executor memory...`  
다음 정보들이 프린트된다.
- all default properties
- command line options
- setting from spark 'conf' file
- setting from CLI

### executor thread & heap dump
-----------
`jmap, jstack, jstat, jhat`과 같은 OpenJDK 툴을 통해 executor의 thread dump나 heap dump를 떠볼 수 있다.
YARN 컨테이너의 pid를 찾아서 사용한다.
- for full thread dump
```bash
jstack -l 355583 > javacore.355583
```
- for full heap dump
```bash
jmap -dump:live,format=b,file=heapdump.355583 355583 
```


# Error Cases
-----------
### Compiled OK, but run-time NoClassDefFoundError 
```java
NoClassDefFoundError: Exception in thread "main" java.lang.NoClassDefFoundError: org/apache/kafka/clients/producer/KafkaProducer at java.lang.Class.getDeclaredMethods0(Native Method) at java.lang.Class.privateGetDeclaredMethods(Class.java:2701) at java.lang.Class.privateGetMethodRecursive(Class.java:3048) at java.lang.Class.getMethod0(Class.java:3018)
```
`--packages` 를 통해 Maven Jar 포함
e.g.)
```java
spark-submit --driver-memory 12g --verbose --master yarn-client --executor-memory 4096m --num-executors 20  --packages org.apache.spark:spark-streaming-kafka_2.10:1.5.1
```
- repo look up 순서
1. local Maven repo - local machine
2. Maven centoral - Web
3. Additional remote repositories specified in - repositories



### No space left on device 
-----------
```java
stage 89.3 failed 4 times, most recent failure: Lost task 38.4 in stage 89.3 (TID 30100, rhel4.cisco.com): java.io.IOException: No space left on device at java.io.FileOutputStream.writeBytes(Native Method) at java.io.FileOutputStream.write(FileOutputStream.java:326) at org.apache.spark.storage.TimeTrackingOutputStream.write(TimeTrackingOutputStream.java:58) at java.io.BufferedOutputStream.flushBuffer(BufferedOutputStream.java:82) at java.io.BufferedOutputStream.write(BufferedOutputStream.java:126)
```
얼마전 운영중인 클러스터에서 발생했던 에러다.  
스파크가 map output file들과 RDD를 저장해두는 `/tmp`가 꽉찬 경우다. 일단은 cron에 정기적으로 tmp 정리를 통해 해결했다.
- `spark.local.dir` 파라미터값의 디폴트값이 `/tmp`인데, 근본적으로 `/tmp`를 Spark의 scratch 공간으로 두는 것 자체가 적절치 않다. 아래 두가지 이유 때문인데
1. `/tmp`는 일반적으로 작은 공간이 할당되며 OS를 위한 공간이다.
2. `/tmp`는 보통 single disk로 IO bottleneck의 원인이 될 수 있다.
- `spark-defualts.conf`에 아래와 같은 내용을 추가하자.  
`spark.local.dir /data/disk1/tmp,/data/disk2/tmp,/data/disk3/tmp,/data/disk4/tmp,...`

### BrodcastTimeout Error
-----------
역시 최근 클러스터에서 발생했는데, 명확하게 BroadcastTimeout Error라고 떨어지는 경우도 있지만, surface상에는 Catalyst error로 떨어지는 경우도 있는 것 같다.
```java
 Typical error stream7/query_07_24_48.sql.out:Error: org.apache.spark.sql.catalyst.errors.package$TreeNodeException: execute, tree: at org.apache.spark.sql.execution.exchange.ShuffleExchange$$anonfun$doExecute $1.apply(ShuffleExchange.scala:122) at org.apache.spark.sql.execution.exchange.ShuffleExchange$$anonfun$doExecute $1.apply(ShuffleExchange.scala:113) at org.apache.spark.sql.catalyst.errors.package$.attachTree(package.scala:49) ... 96 more Caused by: java.util.concurrent.TimeoutException: Futures timed out after [800 seconds] at scala.concurrent.impl.Promise$DefaultPromise.ready(Promise.scala:219) at scala.concurrent.impl.Promise$DefaultPromise.result(Promise.scala:223) at scala.concurrent.Await$$anonfun$result$1.apply(package.scala:190) at scala.concurrent.BlockContext$DefaultBlockContext$.blockOn(BlockContext.scala:53) at scala.concurrent.Await$.result(package.scala:190) at org.apache.spark.util.ThreadUtils$.awaitResult(ThreadUtils.scala:190) ... 208 more §  On surface appears to be Catalyst error (optimizer) 
 ```
`spark.sql.broadcastTimeout 1200` 처럼 parameter를 늘려주는 설정을 통해 해결한다. 
broadcast하는 size의 limit이 있으므로, 무제한으로 broadcast 되지는 않을 것이라 보았다. 클러스터에서 수행되는 긴 쿼리 기준으로 세팅할 수 있을 것이다. 


## Out of Memory Exceptions
-----------
Spark Job이 Executor 또는 Driver의 out of memory exception으로 인해 실패했을 수 있다.
일반적으로는 Executor의 메모리가 부족한 상황을 많이 만나게 된다. 
Executor의 사이즈를 늘려주는 방법을 통해 해결할 수도 있지만, 근본적으로는 애플리케이션이 얼마나 많은 메모리를 필요로 하는지 이해할 수 있어야 한다. 이 부분은 스파크 애플리케이션 최적화에 있어서 가장 기본적이고 필수적인 파라미터 부분이므로 반드시 알아두는 것이 좋다.  
아래 부터 Driver와 Executor의 메모리 에러 상황들에 대해서 더 알아보자.


## Driver Memory Exceptions
-----------
드라이버 메모리가 부족한 경우는 보통 (휴먼 에러가 아니라면) `--driver-memory` 설정을 통해 해결한다. Default값인 512M는 일반적으로 운영환경에서는 너무 작은 값이다.  
**Spark SQL과 Spark Strmeaing은 일반적으로 큰 driver heap size를 요구하는 spark job의 형태**다.


### Exception due to Spark driver running out of memory
-----------
명시적으로 collect() action 등의 driver memory를 사용하지 않는데, driver memory exception이 나서 의아했던 적이 있다. Spark SQL의 Optimizer가 relation을 broadcasting 하기 위해서 중간 과정으로 필요할 수 있다.
드라이버 메모리가 부족한 경우 아래와 같은 형태의 메시지를 볼 수 있다.
```java
Exception in thread "broadcast-exchange-0" java.lang.OutOfMemoryError: Not enough memory to build and broadcast the table
to all worker nodes. As a workaround, you can either disable broadcast by setting spark.sql.autoBroadcastJoinThreshold to -1
or increase the spark driver memory by setting spark.driver.memory to a higher value
```
에러 메시지 상 workaround를 제시된대로, 해당 job에 대해서 브로드캐스트 조인을 끄거나, 브로드캐스트 조인의 threshold를 낮추는 것을 고려할 수도 있다. 아니면 드라이버 메모리의 세팅을 늘려줘서 해결할 수 있다. 
메모리가 허용한다면 당연히 후자가 좋을 것이다. 
`--conf spark.driver.memory= <XX>g`


### Job failure because the Application Master that launches the driver exceeds memory limits
-----------
Application Master(AM)이 드라이버를 메모리 리밋을 넘겨서 런칭했고 YARN에 의해 terminated된 경우.
```java
Diagnostics: Container [pid=<XXXXX>,containerID=container_<XXXXXXXXXX>_<XXXX>_<XX>_<XXXXXX>] is running beyond physical memory limits.
Current usage: <XX> GB of <XX> GB physical memory used; <XX> GB of <XX> GB virtual memory used. Killing container
```

## Executor Memory Exceptions 
### Exception because executor runs out of memory
-----------
스파크 운영 중 종종 마주치게 되는 전형적인 GC issue.
- garbage collection에 98% 이상의 total time이 쓰여지고 있는 경우
- gc를 통해 2% 이하의 heap이 회복된 경우
- top command를 통해 확인했을 때, 1 cpu core가 100% 사용률을 치고 있는데 완료되고 있는 job은 없는 경우
```java
Executor task launch worker for task XXXXXX ERROR Executor: Exception in task XX.X in stage X.X (TID XXXXXX)
java.lang.OutOfMemoryError: GC overhead limit exceeded
```  

1. Executor의 사이즈를 늘려주는 방법을 통해 해결한다.  
`--conf spark.executor.memory= <XX>g`  

2. GC policy를 변경한다. 
- Spark default는 -XX:UseParallelGC
- -XX:G1GC 로 overwrite을 시도해볼 수 있다.
- Default가 일반적으로 좋지만, 상황에 따라 다를 수 있다. (자세한 내용은 별도의 포스트에서 다루고자 한다)


### FetchFailedException due to executor running out of memory
-----------
```java
ShuffleMapStage XX (sql at SqlWrapper.scala:XX) failed in X.XXX s due to org.apache.spark.shuffle.FetchFailedException:
failed to allocate XXXXX byte(s) of direct memory (used: XXXXX, max: XXXXX)
Copy to clipboard
```
executor 메모리를 더 늘려주거나,
`--conf spark.executor.memory= <XX>g`
shuffle partition의 수를 더 늘려줄 수 있다.
`--spark.sql.shuffle.partitions`

### Executor container killed by YARN for exceeding memory limits
-----------
Executor를 호스닝하는 container가 overhead task나 executor task를 위해서 더 많은 메모리를 필요로 하는 경우 아래와 같은 에러가 발생할 수 있다.
```java
org.apache.spark.SparkException: Job aborted due to stage failure: Task X in stage X.X failed X times,
most recent failure: Lost task X.X in stage X.X (TID XX, XX.XX.X.XXX, executor X): ExecutorLostFailure
(executor X exited caused by one of the running tasks) Reason: Container killed by YARN for exceeding memory limits. XX.X GB
of XX.X GB physical memory used. Consider boosting spark.yarn.executor.memoryOverhead
```
executor의 memory overhead 비중을 더 높게 세팅해줄 수 있다.
executor의 메모리 오버헤드 사이즈는 executor의 사이즈에 비례해서 커진다.(대략 6-10%)
best practice는 executor size에 맞춰서 memory overhead size도 조정해주는 것이다.  
`--conf spark.executor.memoryOverhead=XXXX`  
위의 방법이 통하지 않는다면, 더 큰 인스턴스로 옮기거나, 코어의 개수를 줄여볼 수도 있다.  
코어의 개수를 줄이면 메모리가 낭비되겠지만, job은 일단 돌릴 수 있을 것이다.  
`--executor-cores=XX`  

### FileAlreadyExistsException
-----------
이전에 실패한 task에서 파일들을 남겨서 FileAlreadyExistsException를 발생시킬 수 있다.
executor가 메모리 부족으로 실패하고 다른 executor가 다시 해당 task를 이어받았을 때 발생할 수 있다.
어떤 Spark executor가 실패했을 때, Maximum number만큼 retry하고 나서 이 Exception을 남길 수 있다.
```java
org.apache.spark.SparkException: Task failed while writing rows
at org.apache.spark.sql.execution.datasources.FileFormatWriter$.org$apache$spark$sql$execution$datasources$FileFormatWriter$$executeTask(FileFormatWriter.scala:272)
at org.apache.spark.sql.execution.datasources.FileFormatWriter$$anonfun$write$1$$anonfun$apply$mcV$sp$1.apply(FileFormatWriter.scala:191)
at org.apache.spark.sql.execution.datasources.FileFormatWriter$$anonfun$write$1$$anonfun$apply$mcV$sp$1.apply(FileFormatWriter.scala:190)
at org.apache.spark.scheduler.ResultTask.runTask(ResultTask.scala:87)
at org.apache.spark.scheduler.Task.run(Task.scala:108)
at org.apache.spark.executor.Executor$TaskRunner.run(Executor.scala:335)
at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1142)
at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:617)
... 1 more
Caused by: org.apache.hadoop.fs.FileAlreadyExistsException: s3://xxxxxx/xxxxxxx/xxxxxx/analysis-results/datasets/model=361/dataset=encoded_unified/dataset_version=vvvvv.snappy.parquet already exists
at org.apache.hadoop.fs.s3a.S3AFileSystem.create(S3AFileSystem.java:806)
at org.apache.hadoop.fs.FileSystem.create(FileSystem.java:914)
```
- FileAlreadyExistsException의 root cause인, 가장 앞서 실패한 original executor의 실패 원인을 밝힌다.

### Max result size exceeded
-----------
```java
 Typical error stream5/query_05_22_77.sql.out:Error: org.apache.spark.SparkException: Job aborted due to stage failure: Total size of serialized results of 381610 tasks (5.0 GB) is bigger than spark.driver.maxResultSize (5.0 GB) (state=,code=0)) §  Likely to occur with complex SQL on large data volumes
```
큰 데이터 볼륨을 처리하기 위한 복잡한 SQL에서 발생할 가능성이 있다.  
Spark Driver Max Result Size값보다 return된 result가 클 때 발생한다.  
`--conf spark.driver.maxResultSize` 재설정을 통해 해결한다.

### Too Large Frame error
-----------
shuffle data size block의 size가 스파크가 처리 할 수 있는 한계인 2GB보다 큰 경우 발생.
```
org.apache.spark.shuffle.FetchFailedException: Too large frame: XXXXXXXXXX
at org.apache.spark.storage.ShuffleBlockFetcherIterator.throwFetchFailedException(ShuffleBlockFetcherIterator.scala:513)
at org.apache.spark.storage.ShuffleBlockFetcherIterator.next(ShuffleBlockFetcherIterator.scala:444)

Caused by: java.lang.IllegalArgumentException: Too large frame: XXXXXXXXXX
at org.spark_project.guava.base.Preconditions.checkArgument(Preconditions.java:119)
at org.apache.spark.network.util.TransportFrameDecoder.decodeNext(TransportFrameDecoder.java:133)
```
1. `spark.sql.shuffle.partitions`의 value를 default 200에서 큰 값으로 조정 -> `spark.default.parallelism` 을 `spark.sql.shuffle.partitions`과 동일한 값으로 변경

2. issue가 발생하는 dataframe을 밝혀내보자. dataframe 생성 뒤에 action(어떤 action이든, count 등)을 붙여서 dataframe별로 action을 시켜보고, 문제가 생기는 데이터프레임을 밝혀내볼 수 있다. 해당 데이터프레임을 repartition하고 cache해놓는다. 이 때 파티션이 된 데이터의 skewness가 심하다면 코드의 튜닝이 필요할 수 있다.

### Network Timeout
-----------
```java
16/07/09 01:14:18 ERROR spark.ContextCleaner: Error cleaning broadcast 28267 org.apache.spark.rpc.RpcTimeoutException: Futures timed out after [800 seconds]. This timeout is controlled by spark.rpc.askTimeout at org.apache.spark.rpc.RpcTimeout.org$apache$spark$rpc$RpcTimeout$ $createRpcTimeoutException(RpcTimeout.scala:48) at org.apache.spark.rpc.RpcTimeout$$anonfun$addMessageIfTimeout$1.applyOrElse(RpcTimeout.scala:63) at org.apache.spark.rpc.RpcTimeout$$anonfun$addMessageIfTimeout$1.applyOrElse(RpcTimeout.scala:59) at scala.PartialFunction$OrElse.apply(PartialFunction.scala:167) at org.apache.spark.rpc.RpcTimeout.awaitResult(RpcTimeout.scala:83) at org.apache.spark.storage.BlockManagerMaster.removeBroadcast(BlockManagerMaster.scala:143) And timeout exceptions related to the following: spark.core.connection.ack.wait.timeout spark.akka.timeout spark.storage.blockManagerSlaveTimeoutMs spark.shuffle.io.connectionTimeout spark.rpc.askTimeout spark.rpc.lookupTimeout
```
시스템 리소스 상황에 따라 발생할 수 있다. 시스템 리소스의 튜닝이 최우선이고, 안전장치로 timeout setting을 늘려줄 수 있다.  
`spark.network.timeout` 파라미터를 늘려준다. default는 120이다. 에러가 발생한 timeout 시간만큼 늘려줘 본다.  

### Spark job fails with throttling in S3 when using MFOC (AWS)
-----------
무겁고 높은 로드를 일으키는 job에서, Multipart Upload를 활성화한 upload가 실패할 수 있다.  

Spark Override configuration에 아래와 같은 설정들을 잡아준다.
- 해당 작업이 실패할 때, 하둡이 다른 pending된 upload까지 다 abort 시킬 수 있다. 이는 연관된 다른 작업들까지 실패될 수 있으므로, 
`spark.hadoop.fs.s3a.committer.staging.abort.pending.uploads` 설정을 false로 잡아준다. 이후에 Bucket Lifecycle Policy를 통해 실패된 Multipart Uploaded file을 expire시킬 수 있다.
- `spark.hadoop.fs.s3a.committer.threads`의 default 값은 8인데, thread의 수를 더 줄여준다.
- `spark.hadoop.fs.s3a.committer.threads.max` 값을 위의 thread 수와 맞춰준다.
(일반적으로 위의 thread 수를 늘려서 s3 loading 작업을 더 빠르게 만들 수 있지만, S3에서 너무 높은 로드로 실패하는 경우가 생긴다면 이를 줄여서 작게 설정해볼 수 있다.)
- `spark.hadoop.fs.s3a.connection.timeout`값을 default 200000 ms에서 더 높은 값으로 잡아줄 수 있다.

### HTTP 503 "Slow Down" AmazonS3Exception (AWS)
-----------
S3에 스파크로 많은 양의 데이터를 쓰려고 시도하다보면 가끔식 마주치게 되는 에러인 듯 하다.
위의 에러와 같이 엄밀히 말하면 Spark 자체의 에러는 아니지만 S3를 데이터 저장소르 쓰는 경우 스파크로 데이트를 쓰거나 읽을때 발생할 수 있다. prefix마다 초당 3,500개의 PUT/COPY/POST/DELETE 및 5,500개의 GET/HEAD 요청을 넘어갈 때 발생한다.
```
java.io.IOException: com.amazon.ws.emr.hadoop.fs.shaded.com.amazonaws.services.s3.model.AmazonS3Exception: Slow Down (Service: Amazon S3; Status Code: 503; Error Code: 503 Slow Down; Request ID: 2E8B8866BFF00645; S3 Extended Request ID: oGSeRdT4xSKtyZAcUe53LgUf1+I18dNXpL2+qZhFWhuciNOYpxX81bpFiTw2gum43GcOHR+UlJE=), S3 Extended Request ID: oGSeRdT4xSKtyZAcUe53LgUf1+I18dNXpL2+qZhFWhuciNOYpxX81bpFiTw2gum43GcOHR+UlJE=

```
1. 가장 기본적인 해결법으로 버킷의 prefix를 더 나누는 방법이 있다.
```
s3://awsexamplebucket/images
s3://awsexamplebucket/videos
s3://awsexamplebucket/documents
```
그러면 prefix 3개로 나뉘어져서 해당 버킷에 대해 초당 10,500건의 쓰기 요청 또는 16,500건의 읽기 요청을 할 수 있다.   
이 때 prefix란 bucket + 1 depth까지의 namespace까지를 말한다. 즉 s3://awsexamplebucket/images/prefix_depth2, s3://awsexamplebucket/images/prefix_depth3 으로 2 depth 이하로 prefix를 나누는 경우는 이 로직이 동작하지 않는다. 우리 시스템 같은 경우 이렇게 여러 depth를 들어간 이후 table구조를 구성하고 있기 때문에 한 때 이 에러를 자주 마주쳤다.  
2. Amazon S3 요청 수 줄이기  
디렉토리 구조를 통으로 바꾸는 것은 간단한 일이 아니기 때문에 우리는 Spark Job의 병렬도를 낮춰서 해결했다. 
위와 같은 제약사항을 고려하여 S3 bucket 구조를 설계할 때, 한 bucket에 모든 데이터를 담고 그 아래 prefix를 여러 depth로 쪼개는 것보다 목적에 따라 여러 bucket과 1depth의 prefix를 가져가는 구조로 설계하는 것이 좋아보인다.  
3. EMRFS 재시도 제한 증가
```
[
    {
      "Classification": "emrfs-site",
      "Properties": {
        "fs.s3.maxRetries": "20",
        "fs.s3.consistent.retryPeriodSeconds": "10",
        "fs.s3.consistent": "true",
        "fs.s3.consistent.retryCount": "5",
        "fs.s3.consistent.metadata.tableName": "EmrFSMetadata"
      }
    }
]
```



## Reference
[https://docs.qubole.com/en/latest/troubleshooting-guide/spark-ts/troubleshoot-spark.html](https://docs.qubole.com/en/latest/troubleshooting-guide/spark-ts/troubleshoot-spark.html)  
[https://medium.com/ibm-data-ai/beginners-troubleshooting-guide-for-spark-ibm-analytics-engine-199019cfc6b4](https://medium.com/ibm-data-ai/beginners-troubleshooting-guide-for-spark-ibm-analytics-engine-199019cfc6b4)  
[https://www.slideshare.net/jcmia1/a-beginners-guide-on-troubleshooting-spark-applications](https://www.slideshare.net/jcmia1/a-beginners-guide-on-troubleshooting-spark-applications)
[https://aws.amazon.com/ko/premiumsupport/knowledge-center/emr-s3-503-slow-down/](https://aws.amazon.com/ko/premiumsupport/knowledge-center/emr-s3-503-slow-down/)

