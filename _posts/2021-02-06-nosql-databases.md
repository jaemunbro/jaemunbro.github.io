---
title: Features of NoSQL Databases (NoSQL 데이터베이스별 특징)
author: Jaemun Jung
date: 2021-02-06 00:00:00 +0900
categories: [NoSQL]
tags: [nosql]
---

초기의 파일 기반이나 계층형 데이터베이스 이후 수십년동안 데이터베이스가 곧 관계형 데이터베이스로 이해될 만큼 관계형 데이터베이스 중심의 시장이 지속되어 왔다. 그러나 최근 몇 년간 대용량 데이터에 대한 요구가 급격히 커지면서 대용량 데이터의 처리에 강점을 가진 NoSQL과 같은 다양한 데이터베이스 모델이 개발되었다.

관계형 데이터베이스와 NoSQL 데이터베이스의 특징과 차이에 대해 간단히 정리해보자.

- 관계형 데이터베이스는 엔터티의 구조와 엔터티 간의 관계를 토대로 설계하지만, NoSQL에서는 일반적으로 엔터티 관계 구조를 갖지 않는다.
- 관계형 모델은 데이터 간의 관계를 정의하고 데이터 정합성, 일관성을 유지하기 위한 모델로서 출현했다.
NoSQL은 대용량 데이터의 읽기와 쓰기 작업을 위해 기존 관계형 데이터베이스 구조로는 부족한 점에 대한 대안으로서 출현했다.

### NoSQL의 일반적인 장단점
-------------------
- 장점  
    - NoSQL은 대용량 데이터의 읽기와 쓰기에 강점을 가진다. (수평적 확장이 용이하다)
    - 관계형 데이터베이스에서 명확하게 정의된 스키마를 강제함을 통해 데이터 무결성을 보장한다면, NoSQL에서는 유연하게 스키마를 정의할 수 있다. 이러한 장점을 활용해서 document DB등에서 스키마 디자인등에 구애받지 않는 빠른 테스트를 진행해볼 수도 있다.  
- 단점  
    - 대용량 데이터의 읽기와 쓰기에 대한 trade-off로, 관계형 데이터의 주요한 특성인 즉각적인 일관성(consistency)을 만족시키지 못하는 경우가 많다.  일반적으로 최종적인 일관성(Eventually consistent)을 지원한다.
    - ACID 트랜잭션을 지원하지 않을 수 있다.
    - 스키마를 강제하지 않으므로 그 정의가 유연하고 편리한 대신, 개발자가 애플리케이션 레벨에서 데이터에 대한 무결성 검증을 직접 해야만 한다.
    - 보편적인 SQL 문법을 지원하지 않는 경우가 많으므로 이에 대한 Learning Curve가 상대적으로 큰 편이다.

# CAP Theorem
-------------------
CAP 정리는 NoSQL에 대해 처음에 이해하기 위해 가장 많이 언급되는 이론이다.  
하지만 CAP 정리는 하나의 일관성 모델과 한 종류의 결함(네트워크 분단)만 고려하므로 실제 시스템에서 고려해야 하는 네트워크 지연, 죽은 노드나 다른 트레이드오프들에 대해서 고려하지 않는다는 한계가 있다. 따라서 CAP 정리는 초기에 NoSQL 컨셉에 대한 이해를 돕기 위한 목적 외에 실제 시스템을 설계할 때 쓰기 위한 실용적인 가치는 크지 않다는 점을 알아두자.  
- Consistency(일관성) : 모든 노드들은 같은 시간에 동일한 항목에 대하여 같은 내용의 데이터를 사용자에게 보여준다.
- Availability(가용성) : 모든 사용자들이 읽기 및 쓰기가 가능해야 하며, 몇몇 노도의 장애 시에도 다른 노드에 영향을 미치지 않는다.
- Partition Tolerance(분할내성) : 메시지 전달이 실패하거나 시스템 일부가 망가져도 시스템이 계속 동작할 수 있어야 한다.  

![image](https://user-images.githubusercontent.com/29077671/117692092-a0ef2e00-b1f7-11eb-8846-2828f79b87c5.png)*Database Systems according to the CAP theorem, from https://www.researchgate.net/*

위의 조건에서 P를 기본조건으로 두고, Consistency와 Availability가 충돌하는 상황의 예시를 보자.
- **CP** : 애플리케이션에서 일관성을 요구하는 상황에서 일부 복제 서버가 네트워크 문제로 다른 서버와 연결이 끊기면 그동안은 연결을 처리할 수 없다. 네트워크 문제가 고쳐질 때까지 기다리거나 오류를 반환해야한다. 어떻게 해도 가용성(Availability)이 보장되지 않는다.
- **AP** : 애플리케이션에서 일관성을 요구하지 않는다면, 각 복제 서버가 연결이 끊기더라도 독립적으로 요청을 처리하는 식으로 가용성(Availability)을 제공할 수 있다. 단 일관성은 깨어진 상태일 것이다.

# NoSQL 데이터베이스별 특성
-------------------
NoSQL 데이터베이스를 세분화해서 Key-Value, Document등 각 NoSQL 데이터베이스별 특성을 알아보고, 우리 애플리케이션에 맞는 데이터베이스는 무엇일지 고민해보았다.

NoSQL 데이터베이스의 특성을 크게 네가지로 나누면 다음과 같이 나눌 수 있다.
- Key-Value Database
- Document Database
- Column Family Database
- Graph Database

본문에서 Graph Database를 제외한 나머지 세개의 데이터베이스의 특성에 대해 간단히 정리하고 어떤 상황에 어떤 데이터베이스가 적합할지 알아보고자 한다.

# Key-Value Database
-------------------
키와 값으로 이루어진, 저장과 조회라는 가장 간단한 원칙에 충실한 데이터베이스. 더 넓은 의미로 Key-Value Database를 사용하는 경우 Column Family Database들도 포함하여 이야기하는 경우도 많다. 본문에서는 조금 더 세분화해서 그 둘을 구분해서 분류하였다.  

**Key-Value Database의 Key 값은 unique한 고유값**으로 유지되어야 한다.  
**테이블간 조인을 고려하지 않으므로 RDB(Relational Database)에서 관리하는 외부키나, 컬럼별 constraints등이 필요 없다.** 값에 모든 데이터 타입을 허용하며, 그래서 개발자들이 데이터 입력 단계에서 **애플리케이션 레벨에 검증 로직**을 제대로 구현하는 것이 중요하다.(schema-on-read)  
Key-Value Database는, 간단한 데이터 모델을 대상으로 데이터를 자주 읽고 쓰는 애플리케이션에 적합하다. 값은 boolean이나 integer값처럼 단순한 스칼라 값이 일반적이지만, 리스트나 JSON같은 구조화된 값도 가능하다.  

### Key-Value Databases
- Redis
- Riak
- Oracle Berkely
- AWS DynamoDB

### Key-Value Database는 주로 아래와 같은 형태의 애플리케이션에서 사용된다.
1. 성능 향상을 위해 관계형 데이터베이스에서 데이터 캐싱
2. 장바구니 같은 웹 애플리케이션에서 일시적인 속성 추적
3. 모바일 애플리케이션용 사용자 데이터 정보와 구성 정보 저장
4. 이미지나 오디오 파일 같은 대용량 객체 저장

# Document Database
-------------------
Document Database 또는 Document-Oriented Database는 위의 Key-Value Database와 같이 데이터 저장에 Key-Value Type를 사용한다.  
하지만 Key-Value Database와의 중요한 차이는 **Document Database는 값을 문서로 저장**한다는 점이다. 여기서 문서란 **semi-structured entity이며 보통 JSON이나 XML 같은 표준 형식**을 말한다.  
값을 저장하기 전에 schema를 별도로 정의하지 않으며, 문서를 추가하면 그게 바로 schema가 된다.  
각 문서별로 다른 필드를 가질 수 있으며, 따라서 이 역시 **개발자가 애플리케이션에서 데이터를 입력하는 단계에서 컬럼과 필드의 관리가 제대로 이루어지도록 보장**하는 것이 매우 중요하다. 예를 들어 필수 속성(Null을 허용하지 않는 속성)에 대한 관리도 애플리케이션 레벨에서 관리가 이루어져야 한다.  

예시로 MongoDB의 AirBnB DataSet의 일부를 보자. 다음과 같은 JSON 형태의 문서로 관리된다.  

```json
{
"_id": "10006546",
"listing_url": "https://www.airbnb.com/rooms/10006546",
"name": "Ribeira Charming Duplex",
"summary": "Fantastic duplex apartment with three bedrooms, located in the historic area of Porto, Ribeira (Cube)...",
"house_rules": "Make the house your home...",
"property_type": "House",
"calendar_last_scraped": {
    "$date": {
       "$numberLong": "1550293200000"
        }
    },
"amenities": [
    "TV",
    "Cable TV",
    "Wifi",
    "Kitchen",
    "Paid parking off premises",
    "Smoking allowed",
    "Microwave"
    ]
}
```

### Document Databases
- MongoDB
- CouchDB
- Couchbase

### Document Database는 다음과 같은 목적으로 주로 활용된다.
1. 대용량 데이터를 읽고 쓰는 웹 사이트용 백엔드 지원
2. 제품처럼 다양한 속성이 있는 데이터 관리
3. 다양한 유형의 메타데이터 추적
4. JSON 데이터 구조를 사용하는 애플리케이션
5. 비정규화된 중첩 구조의 데이터를 사용하는 애플리케이션

# Column Family Database
-------------------
컬럼 패밀리 데이터베이스는 대용량 데이터, 읽기와 쓰기 성능, 고가용성을 위해 설계되었다. 구글에서 Big Table을 도입하고, 페이스북은 Cassandra를 개발했다.  

Column과 Row와 같이 Relation Database와 동일한 용어를 사용하여 스키마를 정의한다.  
컬럼 수가 많다면 관련된 컬럼들을 컬렉션으로 묶을 수 있다. 예를 들어 이름의 성과 이름을 하나로 묶고, 사무실, 핸드폰 등의 전화번호들을 하나로 묶을 수 있다. 이렇게 묶인 컬럼들을 **Column Family**라고 한다.  

Document Database와 마찬가지로 미리 정의된 스키마를 사용하지 않으므로 개발자가 데이터를 입력하는 시점에 원하는대로 컬럼을 추가할 수 있다.  
테이블간 조인을 지원하지 않는다.  
관계형 데이터베이스에서 한 객체에 대한 정보를 저장하기 위해 여러 테이블에 나누어 저장한다. 예를 들면 고객의 기본 정보 테이블(이름, 주소, 연락처)와 고객의 주문 정보 테이블 등을 나누어서 저장하고 조인을 통해 이객체의 정보를 활용한다.  
컬럼패밀리 데이터베이스는 일반적으로 **비정규화 되어 있으며 한 객체에 관련된 모든 정보를 가능한 매우 너비가 넓은 단일 Row에 넣어서 보관**한다. 따라서 한 Row에 수백만개의 컬럼을 보관하는 경우도 비정상적인 것은 아니다.  
컬럼 패밀리 데이터베이스는 여러대로 구성된 클러스터에서 운영된다. 단일 서버에서 운영해도 될만큼 데이터가 적다면 문서나 키-값 데이터베이스를 고려하는 것이 나을 것이다.  

![image](https://user-images.githubusercontent.com/29077671/117672530-e6563000-b1e4-11eb-96f2-9363b51c2ab4.png "Hbase의 Column Families(from https://www.tutorialspoint.com/hbase/hbase_create_data.html)")*Hbase의 Column Families(from tutorialspoint*

### Column Family Databases
- Hbase
- Cassandra
- GCP BigTable
- Microsoft Azure Cosmos DB

### Column Family Database는 다음과 같은 경우에 활용을 고려하면 좋다.
1. 데이터베이스에 쓰기 작업이 많은 애플리케이션
2. 지리적으로 여러 데이터 센터에 분산되어 있는 애플리케이션
3. 복제본 데이터가 단기적으로 불일치하더라도 큰 문제가 없는 애플리케이션
4. 동적 필드를 처리하는 애플리케이션
5. 수백만 테라바이트 정도의 대용량 데이터를 처리할 수 있는 애플리케이션


## HBase vs Cassandra
-------------------
Column Family Database의 대표격인 Hbase와 Cassandra를 비교해보자.  
두 데이터베이스 중 더 보편적으로 활용되는 것은 Cassandra이다.  

![image](https://user-images.githubusercontent.com/29077671/117672932-464cd680-b1e5-11eb-9e7c-dcec38066f8d.png "Column Family Database(A.K.A Wide Column Stores)의 트렌드 차트(DB-Engines.com)")*Column Family Database(A.K.A Wide Column Stores)의 트렌드 차트(DB-Engines.com)*

아래는 Hbase와 Cassandra간의 Benchmark 테스트 자료(2017)가 있어 첨부하였다.  
high-io가 일어나는 프로덕션 환경을 가정하여 SSD 기반의 머신 5대에서 수행하였다. 벤치마크에 사용된 것은 YCSB(Yahoo! Cloud Serving Benchmark) 툴이다.

![image](https://user-images.githubusercontent.com/29077671/117673071-68465900-b1e5-11eb-8fb1-18013019e2f3.png "hbase와 cassandra benchmark test")*(hbase와 cassandra benchmark test from https://blog.cloudera.com/hbase-cassandra-benchmark/)*

위의 벤치마크 테스트 결과를 보면 Hbase가 대부분의 heavy한 read 작업에서 Cassandra보다 더 나은 성능을 보이고 있다. HBase의 주요한 사용 사례인 search engine이나 high-frequency transaction 애플리케이션, 대용량 로그 분석에 잘 들어맞는 결과이다. 반면에 Cassandra는 write-heavy한 작업에서 데이터일관성을 유지하며 좋은 결과를 보였다. 따라서 빠른 수행 시간보다 분석 데이터 수집이나 센서 데이터 수집 등 일관성 있는 데이터의 수집이 더 중요한 경우에 적절한 대안이 될 수 있을 것이다.  

위와 같이 전반적으로 Hbase가 Cassandra보다 일반적으로 더 나은 성능을 보임에도 불구하고 Cassandra가 더 보편적으로 선호되는 이유는 무엇일까?  
이는 시스템 복잡도나 Learning Curve가 HBase가 더 높기 때문으로 보인다.  
Hbase 자체로는 SQL-Like Language를 제공하지 않아 JRuby 베이스의 HBase shell을 활용해서 쿼리해야 한다. 이 때문에 일단 SQL 대비 Learning curve가 있는 편이다. 혹은 Apache Phoenix라는 쿼리를 위한 별도의 솔루션을 추가로 구성해야한다. 혹은 기존Spark나 Hive와 같은 커넥터를 활용할 수 있는데, 이러한 커넥터를 활용하는 DFS경우 상대적으로 high latency가 발생하게 된다.  

![image](https://user-images.githubusercontent.com/29077671/117673308-a6437d00-b1e5-11eb-9423-c1a7edc6ec19.png)*(Hive와 Phoenix를 활용한 Hbase 쿼리 성능 from https://phoenix.apache.org/performance.html)*
클러스터 구축면에서도 HBase가 더 난이도가 높은 편이다. Hadoop Cluster의 Data nodes에 구성하므로 스케일링은 편리하지만, 최소 5대의 데이터 노드와 하나의 네임노드를 필요로 하는 만큼 최소 유지비용이 많이 드는 편이다.  
또한 HBase는 단독 클러스터 구성이 불가하며 HDFS를 스토리지로 활용하고 Apache Zookeeper를 status 관리와 metadata에 활용하므로, 이러한 기술에 익숙치 않다면 환경 설정이 상대적으로 어려울 수 있다.  
이렇게 HBase 환경을 구축하고 원활하게 운영하기 위해서는 Cassandra 대비 좀 더 난이도 있는 엔지니어링 리소스와 Learning curve를 필요로 하기 때문에 Cassandra가 더 보편적으로 활용되는 것으로 보인다.  


# Reference
Dan Sullivan, Perfect Introduction to NoSQL(NoSQL 철저입문), 2015, 길벗  
[https://blog.cloudera.com/hbase-cassandra-benchmark/](https://blog.cloudera.com/hbase-cassandra-benchmark/)
[Cassandra vs. MongoDB vs. Hbase: A Comparison of NoSQL Databases](https://logz.io/blog/nosql-database-comparison/)
[DynamoDB vs. Cassandra: from "no idea" to "it's a no-brainer" - KDnuggets](https://www.kdnuggets.com/2018/08/dynamodb-vs-cassandra.html)

