---
layout: post
title: Airflow Test 
date: 2021-12-18 23:18 +0800
tags: [jekyll theme, jekyll, tutorial]
toc:  true
---

## 들어가며...

<br>
<br>

## Publish/Subscribe Messaging  

<br>

### Concept  
- sender = publisher (보내는자) : 메시지를 분류해서(classify) 전송하지만, 수신자에게 직접 메시지를 전달하는 것이 아님. (not directing to a receiver)  
- receiver = subscriber (받는자) : 특정 클래스의 메시지를 받도록 구독하는 것  
- broker : 메시지가 발행되는 중앙 지점, 이러한 구조를 운영하는 중앙 포인트  

<br>

### pub/sub 이 왜 필요해졌을까?
- 모니터링 데이터를 전송하는 앱을 생각해보자. 수신자에게 다이렉트로 전달할 수 있다. 하지만 수신자 쪽에서 이 데이터를 좀 더 오래 분석하려고 한다면?  
- 이번에는 분석을 위한 데이터를 전송하는 앱의 수가 2개, 3개 이상으로 많아진다고 생각해보자. 이번에도 수신자 모두에게 다이렉트로 연결하여 전송한다고 하면? <span style="color:#ca8462"><u>커넥션이 너무 복잡해서 추적하기 힘들어진다.</u></span>  
- publish/subscribe messaging system 을 통해 아키텍처의 복잡도를 줄일 수 있다 !  
- 큐 시스템을 이용해서 로그 메시지 pub/sub system, tracking user behavior pub/sub system 등으로 구성할 수도 있다. pub/sub system을 이용해 publisher와 subscriber 사이의 결합도를 낮출 수 있게 된다. 하지만 이 방식은 너무 많은 duplication 이 존재한다.  

<br>
<br>

## Kafka

<br>

### kafka는 무엇인가요?
카프카는 publish/subscribe messaging system 이다.  
distributing streaming platform 이라고도 불린다.  
데이터는 지속적으로, 순서를 유지해서 (stored in order), 분산 저장된다. 

<br>

### Schemas
<span style="color:#c55f4e">메시지의 스키마, Serializer</span>를 의미함  
- serialize (직렬화) : 객체를 string으로 변환하는 방법  
- deserialize (역직렬화) : 직렬화의 역연산  

단순한 포맷으로 JSON, XML 등을 사용가능  
- 장점 : 사용하기 쉽고, 사랍이 읽을 수 있음  
- 단점 : 견고한 타입 핸들링(robust tyoe handling), 스키마 버전 사이의 호환성(compatibility between schema versions) 등이 부족함  

Avro : serialization framework  
- kafka 개발자들이 선호한다고 함, 하둡에 적합하게 개발되었다고 함   
- 장점 : compact serialization format 을 제공함  
- message payload 와 분리된 스키마, 스키마가 변해도 코드를 새로 작성할 필요 없음   

<span style="color:#c55f4e">**지속가능한 데이터 포맷(a consistent data format)**</span>은 중요하다.   
- 메시지를 쓰는 것과 읽는 것의 결합도를 낮춰줌 (subscriber가 새로운 데이터 포맷을 핸들링하기 위해 업데이트 될 필요가 없게 만듦. publisher만 새로운 포맷을 다룰 수 있도록 업데이트 되면 됨.)  
- using well-defined shemas and storing them in a common repository

<br>

### Topic 과 Partition
Topic : 메시지의 카테고리? (Messages are categorized into topics.)  
- 데이터베이스의 테이블, 파일시스템의 폴더와 유사한 개념  
- 전체 토픽에서 메시지의 time-ordering 이 보장되지 않음  

Partition : topic을 이루는 조각? (Topics are broken down into partitions)  
- 하나의 파티션 내에서 메시지들은 <span style="color:#c55f4e">append-only</span> (끝에 연속적으로 붙여짐)  
- 하나의 파티션 내에서 메시지들은 <span style="color:#c55f4e">시작부터 끝의 순서로 읽혀짐</span>  
- 하나의 파티션에 대해서만 메시지의 time-ordering이 보장됨  
- redundancy, scalability 를 제공하는 핵심 개념  
  - <span style="color:#ca8462">각 파티션은 다른 서버에서 호스트될 수 있음</span>
  - 하나의 토픽이 여러 서버에 걸쳐 **수평으로 스케일링**될 수 있다.  
  - 단일 서버보다 나은 성능(performance)을 제공하는 핵심 컨셉 !!  

Stream  
- 파티션 수와 상관없이 단일 토픽 (A single topic of data)  
- producer 에서 consumer 로 이동하는 단일 데이터 흐름  
- 실시간으로 메시지에 연산하는 프레임워크에서 흔히 메시지를 표현하는 방법임    

<br>

### Producer 와 Consumer  
Kafka Client : 시스템 사용자  
- basic type : (1) producer (2) consumer  
- client api 는 고수준의 producer, consumer 기능을 제공함  

Producer : publisher, writer
- 새로운 메시지를 생성  
- 메시지는 특정 토픽에 생성됨  
- 기본적으로, 어떤 파티션에 메시지를 생성할 것인지 고려하지 않음. 토픽 내 모든 파티션에 고르게 메시지를 밸런싱  
- 그러나 때때로 특정 파티션에 메시지를 쓰고 싶은 경우, message key를 이용. partitioner가 이 키를 이용하여 해싱.  
- partitioner도 custom partitioner 사용 가능  
  - 메시지를 매핑하는 비즈니스 룰 등을 따를 때 

Consumer : subscriber, reader
- 메시지를 읽음
- 하나 이상의 토픽을 구독함. 메시지를 순서대로 읽어옴. 
- 이미 소비된 메시지를 추적함. 
  - how? offset 을 추적함  
  - <span style="color:#c55f4e">offset</span> 
    - 연속적으로 증가하는 정수
    - 메시지가 생성될 때 카프카가 각 메시지마다 이 정수를 붙임.
    - 파티션내의 메시지들은 unique offset을 가짐  
  - 각 파티션마다 마지막으로 소비된 메시지의 오프셋을 저장함 (Zookeeper 또는 Kafka 자체에 저장함)  
  - 컨슈머는 읽은 위치를 잃지 않고 stop, restart 가능  
- 컨슈머들은 <span style="color:#c55f4e">Consumer Group</span> 을 이루어 동작함  
  - 같은 컨슈머 그룹의 컨슈머들은 하나의 토픽을 함께 소비함  
  - 그룹은 *하나의 컨슈머가 하나의 파티션을 소비하도록 보장함*  
  - ownership : 컨슈머와 파티션의 매핑  
- consumer는 수평적으로 스케일 될 수 있다.  
  - 하나의 컨슈머가 실패하면 rebalance (파티션을 재분배)  

<br>

### Broker 와 Cluster
Broker : A single Kafka Server  
- producer 로 부터 메시지를 받음  
- 메시지에 오프셋 추가  
- 디스크에 메시지를 저장하는 커밋  
- 컨슈머 서비스  
  - 파티션별 fetch 요청에 응답
  - 디스크 상의 메시지를 보내줌  
- 하드웨어의 성능에 따라 많은 파티션, 메시지들을 초단위로 처리 가능  

Cluster : brokers  
- 하나의 브로커는 cluster controller 로 선정됨 
  - 관리 역할 
  - 파티션을 브로커에 할당하는 것, 브로커의 실패를 모니터링하는 것 등의 관리 
- <span style="color:#ca8462">**하나의 파티션은 하나의 브로커가 소유함**</span>  
  - leader : 파티션을 소유하는 브로커
  - 하나의 파티션은 여러 브로커에 할당됨 (=하나의 파티션이 브로커들에 복제됨)
    - 파티션 내의 메시지들의 redundancy 를 제공하는 방법  
    - 리더 브로커가 실패하면 다른 브로커가 leadership을 가져감  
- producer, consumer는 리더 브로커와 연결됨

Retention : 메시지가 저장되는 기간  
- 브로커는 다음과 같은 설정을 가질 수 있음
  - 토픽별 retention
  - 메시지를 저장하는 기간
  - 토픽의 저장 사이즈 (bytes, 1GB) 
- retention을 넘어가면 메시지는 자동 삭제된다.  

<br>
<br>

## Why Kafka?
수많은 pub/sub 시스템 중 왜 카프카를 선택할까?

<br>

### Multiple Producers and Consumers
다양한 시스템에서 데이터를 모으기에 이상적임  
- 다수의 producer들이 공통 포맷을 이용하여 write 가능  

컨슈머가 독립적으로 하나의 메시지 스트림을 읽을 수 있음  
메시지 스트림 공유도 쉬움  

<br>

### Disk-Based Retention  
컨슈머들이 실시간으로 메시지를 처리하지 않아도 메시지가 보존됨  
토픽별 retention rule 적용 가능  
- 컨슈머의 필요에 따라 보존 정책이 달라짐  
- 컨슈머의 느린 처리 또는 트래픽 증가와 같은 상황에서도 데이터를 잃을 위험이 없음  
- 컨슈머가 멈춰도 메시지는 저장됨 

<br>
<br>

## 글을 마치며...


<br>
<br>

#### References
 

<br>
<br>
<br>