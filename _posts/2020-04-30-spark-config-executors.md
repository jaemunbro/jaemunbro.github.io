---
title: Spark - Config Executors (스파크 - 최적의 익스큐터 사이즈와 개수 정하기)
author: Jaemun Jung
date: 20-04-30 00:00:00 +0900
categories: [Spark]
tags: [spark, executor]
---

*Original Post: [[Apache Spark] Executor 사이즈와 개수 정하기](https://jaemunbro.medium.com/spark-executor-%EA%B0%9C%EC%88%98-%EC%A0%95%ED%95%98%EA%B8%B0-b9f0e0cc1fd8) 에서 옮겨왔습니다.*

--------  

Spark Application을 띄울때 가장 기본적으로 설정해야하는 요소.  
Executor의 사이즈와 개수는 어떻게 정하는 것이 좋을까?  


Executor에 관한 몇 가지 기본 전제를 먼저 확인해보자.
- executor는 캐싱과 실행을 위한 공간을 갖고 있는 JVM이다.
- executor와 driver의 사이즈는 하나의 노드나 컨테이너에 할당된 자원보다 많은 메모리나 코어를 가질 수 없다.
- executor의 일부 공간은 스파크의 내부 메타 데이터와 사용자 자료구조를 위해 예약되어야 한다. (평균 약 25%) 이 공간은 spark.memory.fraction 설정으로 변경 가능하며 기본값은 0.6으로, 60%의 공간이 저장과 실행에 쓰이고 40%는 캐싱에 쓰인다.
- 하나의 partition이 여러개 executor에서 처리될 수 없다 :  
하나의 partition은 하나의 executor에서 처리.

# 다수의 작은 executor VS. 소수의 큰 executor?
## 다수의 작은 executor의 문제
--------
1. **하나의 파티션을 처리할 자원이 충분하지 않을 수도 있다.**  
하나의 파티션이 여러개의 executor에서 계산될 수는 없다. 따라서 셔플, skewed 데이터의 캐시, 복잡한 연산의 transformation 수행 시 OOME 또는 disk spill이 생길 수 있다.

2. **자원의 효율적 사용이 힘들다.**  
같은 노드내 executor끼리 통신에도 약간의 비용이 필요하다.
1GB executor를 갖고 있는 경우 연산을 제외한 오버헤드에만 250MB, 거의 25% 수준의 공간을 써야할 수도 있다.

따라서 **자원이 허용된다면, executor는 최소 4GB 이상**으로 설정하는 것을 추천한다.

## 소수의 큰 executor의 문제
--------
1. 너무 큰 executor는 힙 사이즈가 클 수록 GC가 시작되는 시점을 지연시켜 **Full GC로 인한 지연**이 더욱 길어질 수 있다.
2. executor당 많은 수의 코어를 쓰면 동시 스레드가 많아지면서 스레드를 다루는 HDFS의 제한으로 인해 성능이 더 떨어질 수도 있다.
Sandy Ryza([Advanced Analytics with Spark](https://www.amazon.com/Advanced-Analytics-Spark-Patterns-Learning-ebook/dp/B072KFWZ8S)의 저자)는 **executor당 5개의 코어를 최대**로 보아야 한다고 제안한다.([https://blog.cloudera.com/how-to-tune-your-apache-spark-jobs-part-2/](https://blog.cloudera.com/how-to-tune-your-apache-spark-jobs-part-2/))  
경험적으로 볼 때, 7–8개 이상의 코어 할당은 성능 향상에 도움도 되지 않을 뿐더러 CPU 자원을 불필요하게 소모하게 한다고 한다.
(**Yarn에서 활용하는 경우 64GB 정도를 upper limit**으로 보면 좋다.)

# 효율적 세팅을 위해서
CPU 자원 기준으로 executor의 개수를 정하고,  
**executor 당 메모리는 4GB 이상, executor당 core 개수( 1 < number of CPUs ≤ 5)** 기준으로 설정한다면 일반적으로 적용될 수 있는 효율적인 세팅이라고 할 수 있겠다.  
core와 memory size 세팅의 starting point로는 아래 설정을 잡으면 무난할 듯 하다.
```
—-executor-cores 2 --executor-memory 16GB
```

# 예를 들어보면
AWS EMR에 m5.2xlarge를 master로, m5.24xlarge를 core로 10대 띄운 상황을 예시로 보자.  

![image](https://user-images.githubusercontent.com/29077671/117181500-44170080-ae10-11eb-954b-9350ea1cdb24.png)

- EMR 서버 현황
    - Core 서버 : m5.24xlarge 10대
    - 서버당 vCore : 96개
    - 서버당 Memory : 384GiB

- 서버당 executor 수
executor 당 core 수를 먼저 정의하고, 이를 통해 vCore에서 활용할 수 있는 전체 executor 수가 정의될 수 있다.  
executor당 core 수 4개로 지정 시 : 96 vcore / 4 = 24개  
그러나 **Hadoop과 Application Master가 사용할 core등도 제외해야하므로, Yarn의 모든 리소스를 100% 쓸 수는 없다.** 그래서 노드당 **executor는 23개(23*4 = 92 core)**로 정의했다 : 최종 23 * 10 nodes = 230
```
—-executor-cores 4 --num-executors 230
```

- executor당 memory
(Yarn의 resource 가용 memory는 OS와 hadoop deamon 등의 사용분을 제외해야 함. 여기서는 계산을 위해 임의로 노드당 360GB로 가정)
**: 360GB / 24 = 15G**
```
—-executor-cores 4 --num-executors 230 --executor-memory 15GB
```


(참고) m5.xlarge (4core/16GiB memory)를 core로 2대 띄운 EMR 클러스터 전체의 가용 메모리는 24GB으로 전체의 70% 정도   (yarn.scheduler.maximum-allocation-mb 설정값)

![image](https://user-images.githubusercontent.com/29077671/117181528-4b3e0e80-ae10-11eb-90ad-ed07189180f5.png "리소스 매니저")


# 더 알아보기 좋은 자료
- [Apache Spark] [Partition 개수와 크기 정하기](https://medium.com/@jaemunbro/apache-spark-partition-%EA%B0%9C%EC%88%98%EC%99%80-%ED%81%AC%EA%B8%B0-%EC%A0%95%ED%95%98%EA%B8%B0-3a790bd4675d)
- [Best practices for successfully managing memory for Apache Spark applications on Amazon EMR](https://aws.amazon.com/blogs/big-data/best-practices-for-successfully-managing-memory-for-apache-spark-applications-on-amazon-emr/)

# Reference
Holden Karau, High Performance Spark, JPub, pp.303, 2017  
[https://blog.cloudera.com/how-to-tune-your-apache-spark-jobs-part-2/](https://blog.cloudera.com/how-to-tune-your-apache-spark-jobs-part-2/)  
[https://blog.cloudera.com/how-does-apache-spark-3-0-increase-the-performance-of-your-sql-workloads/](https://blog.cloudera.com/how-does-apache-spark-3-0-increase-the-performance-of-your-sql-workloads/)