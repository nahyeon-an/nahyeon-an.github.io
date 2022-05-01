---
layout: post
title: "[Hadoop] 하둡 바이너리"  
date: 2022-05-01 10:36 +0000
categories: hadoop
tags: [hadoop, install]
toc:  true
---

<br>
<br>

## 들어가며...
<hr>
하둡 클러스터 환경을 구성하면서, 설치과정 보다는, 설치한 하둡 바이너리에 들어있는 파일들을 구경하고 소개해보려고 한다.

<br>
<br>

## hadoop-3.3.2 설치  
<hr>  

### bin
보통 bin 디렉터리는 실행 가능한 스크립트들이 들어있다.  
하둡을 설치했을 때 함께 설치되는 바이너리 : hadoop, hdfs, mapred, yarn  
- 즉, 하둡의 기본은 hdfs + mapred + yarn 이라고 볼 수 있다  

```shell
$ ls $HADOOP_HOME
container-executor  hadoop.cmd  hdfs.cmd  mapred.cmd    test-container-executor  yarn.cmd
hadoop              hdfs        mapred    oom-listener  yarn
```

<br>

### sbin  
sbin 아래에는 쉘 스크립트들이 존재함 

```shell
FederationStateStore     refresh-namenodes.sh  start-yarn.cmd    stop-secure-dns.sh
distribute-exclude.sh    start-all.cmd         start-yarn.sh     stop-yarn.cmd
hadoop-daemon.sh         start-all.sh          stop-all.cmd      stop-yarn.sh
hadoop-daemons.sh        start-balancer.sh     stop-all.sh       workers.sh
httpfs.sh                start-dfs.cmd         stop-balancer.sh  yarn-daemon.sh
kms.sh                   start-dfs.sh          stop-dfs.cmd      yarn-daemons.sh
mr-jobhistory-daemon.sh  start-secure-dns.sh   stop-dfs.sh
```

<br>

### etc/hadoop
etc 는 보통 환경 설정 파일이 존재함  
etc/hadoop 아래에 각종 .xml 환경 설정 파일이 존재함  

```shell
capacity-scheduler.xml            httpfs-env.sh               mapred-site.xml
configuration.xsl                 httpfs-log4j.properties     shellprofile.d
container-executor.cfg            httpfs-site.xml             ssl-client.xml.example
core-site.xml                     kms-acls.xml                ssl-server.xml.example
hadoop-env.cmd                    kms-env.sh                  user_ec_policies.xml.template
hadoop-env.sh                     kms-log4j.properties        workers
hadoop-metrics2.properties        kms-site.xml                yarn-env.cmd
hadoop-policy.xml                 log4j.properties            yarn-env.sh
hadoop-user-functions.sh.example  mapred-env.cmd              yarn-site.xml
hdfs-rbf-site.xml                 mapred-env.sh               yarnservice-log4j.properties
hdfs-site.xml                     mapred-queues.xml.template
```  

hadoop-env.sh : 배포판의 bin/ 아래의 하둡 스크립트를 제어하는 환경설정   
```shell
# 하둡 스크립트 실행 시 사용할 java
export JAVA_HOME=/usr/local/java
# pid 파일이 저장될 경로  
export HADOOP_PID_DIR=/home/hadoop/data/hadoop/pids
# daemon log 파일이 저장될 경로 
export HADOOP_LOG_DIR=/home/hadoop/data/hadoop/logs
```

core-site.xml  

hdfs-site.xml  

<br>
<br>

## JVM 메모리 구조
<hr>

하둡을 사용하다 보면 OOM 현상을 종종 마주하게 된다.  
이 메모리 부족 현상을 잘 이해하기 위해서 JVM 메모리 구조를 살펴보자  

Garbage Collection : 메모리 동적할당(+해제)을 JVM 에서 수행함  

<br>

### JVM 구조  

[블로그 글](https://hyoje420.tistory.com/2)

<br>

### jar  
Java Archive 의 줄임말  
여러 클래스 파일, 관련 리소스, 메타데이터를 하나로 모아 배포하기 위한 패키지 파일 포맷  
jar 는 zip 포맷으로 이루어진 압축 파일  

<br>
<br>

## Tez Engine  
<hr>

higher level tool (hive, pig)이 처리 단계를 정의할 수 있게 api 제공  
- 잡의 시작 전에 각 step 이 계산되므로, 잡결과를 메모리에 캐싱하는 이점을 얻을 수 있음  
- 맵리듀는 단계 사이의 데이터들이 hdfs 에 라이트 되어야 함 

dag : job을 완료하기위해 필요한 스텝  

Data Processing API  
- DAG : 전체 job을 정의. 각각의 데이터 처리 잡마다 dag 를 생성  
- Vertex : 로직, 리소스, 환경 등을 정의   
  - vertex 병렬 실행을 지원  
- Edge : producer, consumer vertex 사이의 커넥션, vertex 사이를 연결하기 위해 Edge 객체를 생성해야 함   
  - 유저 태스크를 인스턴스화, 인풋과 아웃풋을 설정, 스케줄, 태스크 사이의 데이터 라우트를 담당  
  - data movement : 태스크 사이 데이터의 라우팅을 정의 (1:1)  
  - broadcast : 프로듀서의 데이터를 모든 컨슈머 태스크에 라우트 하는 것  
  - scatter-gather : 프로듀서 태스크가 데이터를 샤드로 라우트(scatter)하고, 컨슈머 태스크는 샤드들을 모으는 것 (gather)   
  - scheduling : 컨슈머 태스크를 스케줄링 (sequential : 컨슈머 태스크는 프로듀서 태스크가 완료된 이후 스케줄됨)  
  - concurrent : 컨슈머 태스크는 프로듀서 태스크와 반드시 동시에 스케쥴됨 (co-scheduled)  
  - data source : 태스크 아웃풋의 생명주기를 정의함.  

<br>

### Tez 설계  
Tez 는 다음 2가지를 필요로 함. (1) yarn 이 실행된 클러스터 (2) hdfs 인터페이스를 통해 접근 가능한 파일 시스템  

Tez 가 제공하는 서비스  
- runtime components
- api  
  - 전통 map-reduce 기능이 접근되어질 수 있음  
    - Job 인터페이스를 구현한 자바 클래스를 통해 접근 가능  
    - org.apache.hadoop.mapred.Job  
    - org.apache.hadoop.mapreduce.v2.app.job.Job  
  - Dag 기반의 실행은 DAG Api 를 통해 접근 가능  
    - org.apache.tez.dag.api.*
    - org.apache.tez.engine.api.*
  - yarn-site 설정파일에 map-reduce 프레임워크가 Tez 임을 명시해야 함  
- primitive
  - org.apache.tez.engine.common.*
  - Vertex Input/Output
  - Sorting
  - Shuffling
  - Merging
  - Data transfer 

Tez-Yarn 아키텍처  
Tez -> AppMaster  
AppMaster ->  
(1) Yarn ResourceManager 를 호출 -> 워커 컨테이너 할당 요청  
(2) Yarn NodeManager 를 호출 -> 실제 워커 프로세스를 실행하도록 호출  

추가로, AM 의 역할  


<br>
<br>

## 글을 마치며...
<hr>
hadoop 을 설치하고, 설치한 파일들의 구조를 살펴보면서 궁금했던 내용들을 정리해보았다.  
설치한 하둡의 bin, sbin 디렉터리를 실행경로에 포함시켜 hdfs, start-dfs.sh 등을 자융롭게 사용하고 있다.  
- bin/ : 하둡의 주요 커맨드 (hdfs, mapred, yarn)  
- sbin/ : 데몬 쉘 스크립트  
와 같이 생각해볼 수 있겠다.  

또 하둡을 사용하다 보면, 자주 마주하는 현상이 OOM 인데, 이를 이해하기 위해 JVM 의 메모리 구조를 다시 조사해봤다.  

Tez 도 설치하여 tez api 를 이용한 잡을 제출해 보는 것이 목표인데, 과연...?!  

<br>
<br>
<br>