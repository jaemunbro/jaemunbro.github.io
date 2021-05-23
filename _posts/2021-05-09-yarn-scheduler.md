---
title: Hadoop - YARN Scheduler setting (하둡 - YARN 스케쥴러 설정)
author: Jaemun Jung
date: 21-05-09 11:45:00 +0900
categories: [Hadoop]
tags: [hadoop, yarn, scheduler]
---
YARN의 스케쥴러 타입에 대해 알아보기 전에 YARN에 대해서도 간단히 살펴보자.

# YARN
--------------
YARN은 (Yet Another Resource Negotiator)은 하둡의 크러스터 자원관리 시스템이다. 
YARN은 클러스터의 자원을 요청하고 사용하기 위한 API를 제공한다.

YARN은 리소스 매니저와 노드 매니저 두가지 장기 실행 데몬을 통해 핵심 서비스를 제공한다.
리소스 매니저는 클러스터에서 유일하며 클러스터 전체 자원의 사용량을 관리하고, 모든 머신에서 실행되는 노드 매니저는 컨테이너를 구동하고 모니터링 하는 역할을 맡는다.

YARN에서 애플리케이션을 구동하는 요청 과정은 다음과 같다.  
1. 클라이언트가 애플리케이션을 구동하기 위해 리소스 매니저에 접속하여 애플리케잇녀 마스터 프로세스의 구동을 요청한다.
2. 리소스 매니저는 컨테이너에서 애플리케이션 마스터를 구동할 수 있는 노드 매니저를 하나 찾는다. 
3. 애플리케이션 마스터가 단순한 계산을 단일 컨테이너에서 수행하고 클라이언트에 반환하고 종료하거나, 
4. 리소스 매니저에 더 많은 컨테이너를 요청한 후 분산 처리를 수행하는 경우도 있다. 4번이 맵리듀스 애플리케이션이 수행하는 방법이다.
![image](https://user-images.githubusercontent.com/29077671/117579456-daf3fd80-b12d-11eb-8cdd-957474f7c116.png "How YARN runs an application. from Hadoop the definitive guide")*How YARN runs an application. from Hadoop the definitive guide*


# Scheduler
--------------
YARN은 FIFO, Capacity, Fair Scheduler를 지원한다. 
- FIFO는 먼저 들어온 애플리케이션을 큐에 넣고 순차대로 실행한다. 간단하고 직관적이지만 공유 클러스터 환경에서는 적합하지 않다. 
- Capacity Scheduler는 잡을 제출되는 즉시 분리된 전용 큐에서 처리한다. 잡을 위한 자원을 미리 예약해두므로 전체 클러스터의 효율성이 떨어질 수 있다.
- Fair Scheduler는 실행 중인 모든 잡의 자원을 동적으로 분배하므로 미리 자원의 가용량을 예약해둘 필요가 없다. 대형 잡이 먼저 시작하면 모든 자원을 할당한다. 다른 잡이 중간에 들어오면 필요한 자원만큼 클러스터의 자원을 해당 잡에 나누어준다.  
두번째 잡이 시작된 후 자원을 받을 때까지 시간은 좀 걸릴 수 있다. 첫번째 잡이 사용하고 있는 컨테이너의 자원이 해제될 때까지 기다려야하기 때문이다. 작은 잡이 완료되고 나면 대형 잡은 다시 전체 클러스터의 자원을 점유할 수 있다. 전체적으로 클러스터의 효율성도 높고 작은 잡도 빨리 처리되는 효과를 얻을 수 있다.  
![image](https://user-images.githubusercontent.com/29077671/117579463-e34c3880-b12d-11eb-9ed7-ec1a0de63508.png "Cluster Utilization over time when running a large job and a small job under the FIFO Scheduler (i), Capacity Scheduler (ii), and Fair Scheduler (iii). from Hadoop the definitive guide")*Cluster Utilization over time when running a large job and a small job under the FIFO Scheduler (i), Capacity Scheduler (ii), and Fair Scheduler (iii).  
from Hadoop the definitive guide*


### Capacity Scheduler
--------------
공유 클러스터에서 회사 내 조직마다 자원을 할당해줄 수 있다. 큐 안에는 잡이 다수 있고 가용 자원이 클러스터에 남아있다면 캐퍼시티 스케쥴러는 여분의 자원을 할당할 수도 있다. 이렇게 할당하는 경우 지정된 큐의 가용량을 초과하게 되는데, 이러한 방식을 queue elasticity라고 한다.  

### Fair Scheduler
--------------
페어 스케쥴러는 실행 중인 모든 애플리케이션에 동일하게 자원을 할당한다. Cloudera에서 사용을 권장하는 Scheduler 타입이다. 기본 Hadoop 스케쥴러는 Capacity Scheduler지만 CDH 배포판은 기본 스케쥴러가 Fair Scheduler이다. 

스케쥴러 설정은 `yarn-site.xml`에서 하고, 각 스케쥴러의 큐 설정은 `capacity-scheduler.xml`, `fair-scheduler.xml`등의 파일에서 지정한다.
- yarn-site.xml의 예시  

```xml
<property>  
 <name>yarn.resourcemanager.scheduler.class</name>  
 <value>org.apache.hadoop.yarn.server.resourcemanager.scheduler.fair.FairScheduler</value>  
</property>
```
페어 스케쥴러는 클래스 경로에 있는 `fair-scheduler.xml`이라는 할당 파일에 속성을 설정한다. 

- fair-scheduler.xml의 예시  

```xml
<?xml version="1.0"?>
<allocations>
  <defaultQueueSchedulingPolicy>fair</defaultQueueSchedulingPolicy>

  <queue name="prod">
    <weight>40</weight>
    <schedulingPolicy>fifo</schedulingPolicy>
  </queue>

  <queue name="dev">
    <weight>60</weight>
    <queue name="eng" />
    <queue name="science" />
  </queue>

  <queuePlacementPolicy>
    <rule name="specified" create="false" />
    <rule name="primaryGroup" create="false" />
    <rule name="default" queue="dev.eng" />
  </queuePlacementPolicy>
</allocations>
```
위 예제에서 prod 큐와 dev 큐는 40:60의 비율로 클러스터의 자원을 할당 받는다. 그리고 eng과 science 큐는 가중치를 지정하지 않았으므로 dev큐의 자원을 동등하게 할당 받는다. 가중치의 합이 정확하게 100%이 되어야할 필요는 없다. 예를 들면 예제의 prod와 dev를 각각 2, 3으로 변경해도 동일한 비율로 분배된다.  
큐 내부의 정책은 schedulingPolicy 항목으로 재정의가 가능하다. 예를 들면 prod 큐는 가장 빨리 끝나는게 중요하므로 내부적으로 FIFO를 적용했다. 

#### **Preemption (선점)**
--------------
클러스터가 꽉 차게 수행되고 있는 상태에서 Fair Scheduler 모드로 새로 제출된 잡은 다른 잡이 자원을 해제해주기 전까지 잡을 시작할 수 없다. 잡의 시작 시간을 어느 정도 예측가능하게 해주기 위해 선점 기능을 세팅할 수 있다. 선점은 스케쥴러가 자원의 균등 공유에 위배되는 큐에서 실행되는 컨테이너를 '죽일' 수 있도록 허용한다. 이렇게 수행되다가 중간에 죽은 컨테이너는 결국 다시 처음부터 수행되어야 하므로 클러스터의 효율은 떨어질 수 있다.  
선점은 `yarn.scheduler.fair.preemption`을 true로 설정하여 활성화 시킬 수 있다.


### Delay Scheduling
--------------
YARN의 모든 스케쥴러는 지역성을 가장 우선시한다. 어떤 애플리케이션이 특정 노드를 요청하면 딱 그 시점에는 그 노드가 바쁠 가능성이 높다. 이 경우 지역성을 조금 포기해서 동일 노드가 아니라 동을 랙에 컨테이너를 할당할 수도 있다. 하지만 대부분의 경우 '조금만' 기다리면 요청한 노드에서 컨테이너를 할당받을 수 있는 기회가 크게 증가한다. 이러한 기능을 지연 스케쥴링(delay scheduling)이라고 한다. 


#### AWS EMR에서 Scheduler 설정 변경
--------------
[https://aws.amazon.com/premiumsupport/knowledge-center/restart-service-emr/](https://aws.amazon.com/premiumsupport/knowledge-center/restart-service-emr/)
[https://stackoverflow.com/questions/34953319/how-to-restart-yarn-on-aws-emr/](https://stackoverflow.com/questions/34953319/how-to-restart-yarn-on-aws-emr/)


# Reference
Tom White, Hadoop The Definitive Guide(하둡 완벽가이드 4판, 2018), 한빛미디어
[https://docs.cloudera.com/documentation/enterprise/6/6.3/topics/admin_fair_scheduler.html](https://docs.cloudera.com/documentation/enterprise/6/6.3/topics/admin_fair_scheduler.html)
[https://wikidocs.net/book/2203](https://wikidocs.net/book/2203)
