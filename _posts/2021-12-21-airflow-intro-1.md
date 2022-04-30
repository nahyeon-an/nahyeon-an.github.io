---
layout: post
title: "[Airflow] Docker를 이용한 로컬 에어플로우 실행과 DAG 등록 - (1)"  
date: 2021-12-21 01:45 +0000
categories: airflow
tags: [airflow, tutorial, intro]
toc:  true
---

<br>
<br>

## 들어가며...
<hr>
데이터를 모델링, 분석하기 위해서 데이터에 다양한 연산, 가공을 한다. 그러한 작업에는 순서가 있다. 또 이러한 작업이 특정 주기, 시간에 실행되어야 한다.   
이러한 데이터 워크플로우를 쉽게, 시각적으로 관리하기 위한 툴인 Airflow에 대해 알아보려고 한다. (열심히 공식문서를 읽어보자!)  

공식문서의 Quick Start 를 따라하면서 직접 로컬 에어플로우에 dag 를 등록하고 실행하는 것까지 해보려고 한다. 기본 키 컨셉들을 소개하고 자세한 내용들은 이후의 포스트에서 다루자.  

<br>
<br>

## Airflow 란?  
<hr>  
"Workflow를 빌드, 런(build and run)하는 플랫폼" 이라고 공식문서에 나와있다.  

여기서 말하는 Workflow 는 DAG(Directed Acyclic Graph)를 의미한다.  
DAG는 그래프 알고리즘에서 쉽게 접하는 개념으로, 노드가 Task 로 정의되었다고 생각하면 된다.  
즉, Task들이 방향이 있고(directed), 순환하지 않게(Acyclic) 연결된 것이다.  

DAG는 Task 사이의 의존성, 실행 순서, 재시작 횟수 등을 정의한다.  
Task는 우리가 하려는 작업의 정의로, 데이터 가져오기(fetch data), 데이터 분석/모델링(run analysis) 등을 할 수 있다.  

DAG, Task부터 시작하여 이를 구현하는 다양한 컨셉을 정리해보자.  

<br>
<br>

## Airflow Core  
<hr>
Airflow를 구성하는 주요 컴포넌트(Airflow 설치하면 설치되는 것들)는 다음과 같다.   

**Scheduler**   
- 스케줄된 workflow(=DAG)를 트리거(triggering), Task 들을 executor 가 실행하도록 제출한다.  

**Executor**  
- Task 실행을 처리함 

**Web Server**   
- DAG와 Task를 찾기, 트리거, 디버깅 등을 지원하는 유저 인터페이스  

**Folder of DAG Files**    
- scheduler와 executor 가 실제로 읽어가는 DAG 파일과 디렉터리  

**Metadata Database**  
- DAG의 상태저장을 위한 데이터베이스

<br>
<br>

## Airflow 실행해보기 (a.k.a Quick Start) 
<hr>

공식문서에서 제공하는 Quick Start에는 (1) pip 를 이용한 테스트 (2) docker compose를 이용한 테스트 두 가지 방법이 제시되어 있다.  
이 중 (2) docker-compose를 이용한 실습을 진행하며 기본 컨셉을 정리하려고 한다.  
이 가이드에서는 CeleryExecutor를 이용하여 Airflow를 실행해본다.   

사전에 준비되어야 하는 것   
- Docker, Docker Compose, 컨테이너에 4GB 이상의 메모리 할당 가능해야 함  
- 메모리 할당과 관련하여  
  - (1) 도커 환경설정에서 메모리 설정을 4GB 이상으로 설정해야 함 (아마 디폴트는 2GB였을거다)  
    - ![docker-setting](/img/posts/2021/211221_docker_setting.png){: height="50%" width="50%"} ![docker-preference](/img/posts/2021/211221_docker_preference.png)
  - (2) 실제 가용 메모리가 충분해야 한다? ~~(확실하지는 않으나 16G 노트북에서는 웹 서버 실행이 실패했으나 32G 에서는 성공했다.)~~  

<br>

### Fetch Docker Compose
다음의 명령어로 충분한 메모리를 사용할 수 있는지 확인  
```shell
docker run --rm "debian:buster-slim" bash -c 'numfmt --to iec $(echo $(($(getconf _PHYS_PAGES) * $(getconf PAGE_SIZE))))'
```

위의 명령어가 잘 동작했다면 진짜 airflow docker-compose.yaml 파일을 가져온다.  
```shell
mkdir test-airflow && cd test-airflow
curl -LfO 'https://airflow.apache.org/docs/apache-airflow/2.2.2/docker-compose.yaml'
```

결과로 docker-compose.yaml 이 생성되어 있어야 한다.  

이 파일에서 코어 컴포넌트들이 정의되어 있는 것을 확인할 수 있다.  
- airflow-scheduler : 키 컨셉에서 스케줄러의 구현  
- airflow-webserver : 키 컨셉에서 Web UI의 구현 (http://localhost:8080)
- airflow-worker : Task를 실행하는 워커 
- airflow-init : airflow 초기화 
- flower : celery 모니터링 앱 (http://localhost:5555)
- postgres : 데이터베이스 
- redis : 메시지를 스케줄러에서 워커로 전달하는 브로커  

여기서 사용하는 이미지는 도커 허브에 공개된 공식 Airflow latest 이미지이다.  
(파이썬이나 다른 라이브러리의 버전을 바꾸고 싶다면 커스텀하라고 한다...)  

<br>

### 환경 셋팅
에어플로우를 실행하기 전에 실행을 위한 로컬 환경을 셋팅한다.  

docker-compose 환경변수 셋팅 (linux)  
dggs, logs, plugins 디렉터리를 현재 디렉터리에 생성해야 한다. 초기화 시 /sources 하위에 마운트된다.    
- ./dags : 실행하려는 DAG file을 저장
- ./logs : 실행, 스케줄러 등에 의한 로그
- ./plugins : custom plugin  

사용자 id를 에어플로우 변수로 도커 환경변수에 셋팅한다.  
```shell
mkdir -p ./dags ./logs ./plugins
echo -e "AIRFLOW_UID=$(id -u)" > .env
```

docker-compose를 실행한다. 데이터 베이스 마이그레이션, 초기 유저 생성 등을 수행함
```shell
docker-compose up airflow-init
```
초기화가 완료되고 나면 다음 메시지가 보인다.  
> airflow-init_1       | Upgrades done  
> airflow-init_1       | Admin user airflow created  
> airflow-init_1       | 2.2.2  
> start_airflow-init_1 exited with code 0    

id : airflow, password: airflow 로 초기 유저가 생성된다.  

<br>

### Running Airflow  
이제 위에서 말한 서비스들을 실행할 수가 있다.  
```shell
docker-compose up
```
실행하다가 포트 이슈가 있을 수 있다. 나는 8080번 포트를 이미 사용중이라서 30009로 변경하여 실행했다.   

새로운 터미널을 열어서 실행중인 컨테이너들의 상태를 확인할 수 있다.  
```shell
docker ps
```

이렇게 Airflow가 실행되면 Airflow를 사용하는 방법은 3가지가 존재한다.  
1. CLI command  
2. WEB UI 
3. REST API 

<br>

#### 1. CLI Command
CLI Command 를 사용하려면 위에서 생성한 airflow- 로 시작하는 컨테이너에서 airflow command를 수행할 수 있다.
```shell
docker-compose run airflow-worker airflow info
```

linux 계열은 shell script 를 다운받아 쉽게 사용도 가능하다.  
(이 부분은 생략)  

<br>

#### 2. Web UI  
웹의 홈 화면은 다음과 같다. 에어플로우 초기화 단계에서 생성된 id, pw로 로그인한다.  
![airflow web main home](/img/posts/2021/211221_web_default.png)  

<br>

#### 3. REST API  
Web endpoint 로 요청할 수 있다.  
인증방식은 Basic Auth 로 username과 password 를 전달하여 인증한다.  
username과 password 모두 에어플로우 초기화에서 생성된 id, pw 사용하면 된다.  
```shell
ENDPOINT="http://localhost:30009/"
curl -X GET --user "airflow:airflow" "${ENDPOINT}/api/v1/pools"
```
지원되는 uri는 첨부한 문서를 참조하자. ~~(나중에 필요하면 정리할수도?)~~   
[Airflow rest api 공식문서](https://airflow.apache.org/docs/apache-airflow/stable/stable-rest-api-ref.html)  

<br>

### Clean up 
다음의 명령어로 실행했던 도커 컨테이너를 정리할 수 있다.  
하지만 나는 실제 DAG 를 등록하여 실행까지 하기 위해 지금 실행하지 않는다!  
```shell
docker-compose down --volumes --remove-orphans
``` 

<br>
<br>

## Airflow Example DAG 실행해보기 
<hr>
Web UI 를 보면 여러 예시 dag 들이 있다.  
이는 에어플로우 패키지에서 제공되는 예시 대그들로, 직접 등록한 것은 아니다.  
Web UI 에서 실행해보고 어떤 일이 일어나는지, 코드 등을 먼저 살펴보자.  

<br>

### Web UI 살펴보기  
docker-compose 로 실행한 후 웹을 접속했을 때 초기 화면은 다음과 같다. (물론 로그인한 이후다.)  
![airflow web 소개](/img/posts/2021/211221_web_explained.jpeg)
DAGs 라고 하여 에어플로우에 등록된 DAG 리스트들이 나타나있다.  
가장 왼쪽의 버튼은 Dag 를 활성화 여부를 조절하는 버튼이다.  
dag 실행과 관련된 정보를 gui에서 쉽게 확인할 수 있도록 제공한다.  

이제, 하나의 Dag 를 선택하여 들어가보자.  
이번 포스트에서 실행시켜볼 "example_branch_dop_operator_v3" dag 를 클릭한다.  
![에어플로우 트리 뷰](/img/posts/2021/211221_dag_tree_init.png)
![에어플로우 그래프 뷰](/img/posts/2021/211221_dag_graph_init.png)  
해당 dag 코드에 맞게 task 들을 tree 또는 graph 로 표현한다.  
- Tree view : 시간에 걸친 dag 의 표현  
- Graph view : task dependency, 현재 상태를 시각화  

dag를 실행했을 때 이 그래프들에서 무엇을 확인할 수 있는지는 바로 이어서 소개한다.  

<br>

### Dummy Operator 예시 실행하기 
이 예제를 실행하는 이유는 1분마다 스케줄링 되어 있어 바로 확인하기 편리해서 선택했다.    

웹에서 실행하는 방법은 간단하다. 위에서 dag active 조절 버튼을 클릭하여 파란색으로 변경하면 된다.  
파란색으로 버튼 색이 바뀌면 dag가 active 상태로 스케줄링된다.  
![에어플로우 dag active 시키기](/img/posts/2021/211221_active_one_dag.png)  
1분마다 실행되도록 스케줄링된 dag로 빠르게 success 횟수가 증가하는 것을 볼 수 있다.  
또 success task 2개와 skipped task 1개를 task 상태에서 볼 수 있다.  

실행한 dag를 선택하여 dag에 대한 구체적인 뷰를 확인해보자.  
![에어플로우 dag 실행 결과 트리 뷰](/img/posts/2021/211221_dag_tree_view.png)  
이제 트리 뷰에 위에서 확인할 수 없었던 원과 네모가 생겼다.  
여기서 왼쪽의 dag 트리 뷰와 오른쪽의 실행 상태를 매핑하여 보면된다.  

즉, 오른쪽 시간에 따른 상태 뷰의 맨위 원은 dag run 자체를 의미한다. ~~(dag run은 하나의 dag 실행이라고 생각하자... 의미대로...)~~   
이 원이 검은색 굵은 테두리가 존재하면 스케줄된 dag run, 그렇지 않다면 수동으로 실행된 dag run 이다.  
(수동 실행은 오른쪽의 빨간색 휴지통 옆의 play버튼을 누르면 된다.)  

원 아래의 네모표시들은 task 에 매핑된다.  
색깔을 통해 dag run, task 의 상태를 시각화해준다.  
오른쪽 상태 뷰에서 마우스 호버를 하면 각 dag run, task 별로 추가 정보를 볼 수 있다.  
또 클릭해도 추가로 볼 수 있는 정보들이 있다.  

웹에서 볼 수 있는 다양한 기능들은 차차 알아가자!  

무엇을 실행시켰는지, 어떻게 동작하는지 알아보기 전에 dag 를 활성화시켜 에어플로우가 workflow 관리를 시각화하는 web ui에 대해 살펴보았다.    

이제 코드와 동작에 대해 알아볼 차례다.  

<br>

### 코드 살펴보기  
우선 내가 실행했던 예제의 dag 코드는 다음과 같다.  
```python
from datetime import datetime

from airflow import DAG
from airflow.operators.dummy import DummyOperator
from airflow.operators.python import BranchPythonOperator


def should_run(**kwargs):
  print(
    '------------- exec dttm = {} and minute = {}'.format(
      kwargs['execution_date'], kwargs['execution_date'].minute
    )
  )
  if kwargs['execution_date'].minute % 2 == 0:
    return "dummy_task_1"
  else:
    return "dummy_task_2"


with DAG(
  dag_id='example_branch_dop_operator_v3',
  schedule_interval='*/1 * * * *',
  start_date=datetime(2021, 1, 1),
  catchup=False,
  default_args={'depends_on_past': True},
  tags=['example'],
) as dag:

  cond = BranchPythonOperator(
    task_id='condition',
    python_callable=should_run,
  )

  dummy_task_1 = DummyOperator(task_id='dummy_task_1')
  dummy_task_2 = DummyOperator(task_id='dummy_task_2')

  cond >> [dummy_task_1, dummy_task_2]
```  

<br>

1 > DAG 선언 방법  
   1. context manager 를 이용 (이 예제에서 사용한 방식이다)  
      ```python
      with DAG("my_dag_name") as dag:
        op = DummyOperator(task_id="task")
      ```
   2. standard constructor 를 이용  
      ```python
      my_dag = DAG("my_dag_name")
      op = DummyOperator(task_id="task", dag=my_dag)
      ```
   3. @dag decorator 를 이용 (데코레이터는 함수를 DAG 제네레이터로 변경해준다.)  
      ```
      @dag(start_date=days_ago(2))
      def generate_dag():
          op = DummyOperator(task_id="task")

      dag = generate_dag()
      ```

2 > DAG 객체  
DAG를 생성할 때 전달한 인자에 대해 살펴보자.  
- dag_id  
- schedule_interval  
- start_date  
- catchup  
- default_args  
- tags  

3 > Task Dependency 표현하기  
크게 operator(연산자)를 이용하거나 함수를 이용하는 2가지 방법이 있다.  
공식문서에서는 연산자를 이용하는 것을 권장하고 있고, 훨씬 직관적이라서 함수 소개는 생략한다.  
~~(나중에 필요해지면 찾아보자)~~  
<mark>>>, <<</mark> 를 이용하여 task 사이의 의존성을 표현할 수 있다.  

4 > Operator 란?  
우선 공식문서에서는 "사전에 정의된 Task 를 위한 템플릿 (a template for a predefined Task)" 이라고 설명한다.  
DAG 내에서 task 를 정의하는 템플릿, 방법 또는 task 연산자로 이해하면 된다.  
즉, task 가 어떤 연산을 할 것인지 결정한다.  

코어에 내장된 오퍼레이터도 있고, provider가 제공하는 오퍼레이터도 있다.   

가장 흔한 오퍼레이터는 아래 3가지라고 한다.  
- BashOperator : bash 명령어를 실행한다.  
- PythonOperator : python 함수를 호출한다.  
- EmailOperator : email 을 전송한다.  

내가 선택했던 예시에서는 BranchPythonOperator, DummyOperator 두 개를 사용했다.  

먼저, BranchPythonOperator 는 PythonOperator를 상속한다. 즉, python 함수를 전달받아 실행한다.   
(python_callable 인자에 실행하려는 파이썬 함수를 전달)  
이때 workflow를 조건에 따라 분기하도록 동작하는 것이 BranchPythonOperator 이다.  
이 오퍼레이터는 task_id 또는 task_id list 를 리턴해야 한다. 리턴된 task 를 제외한 나머지 task들은 모두 skipped 상태로 표신된다.  

DummyOperator는 아무것도 하지 않는 오퍼레이터이다.  
스케줄러에 의해 스케줄링은 되지만 executor가 실행하지는 않는 task type이다.  
task를 그루핑 하는데에 사용가능하다고 한다.  

오퍼레이터는 추후에 더 자세히 포스팅하자 !  

<br>
<br>

## 글을 마치며...
<hr>
도커를 이용해 에어플로우를 로컬에서 실행시키고, 패키지가 제공하는 예제를 실행시켜 "실습을 통한 기본 컨셉 이해하기"를 진행했다.  

에어플로우를 실행할 때 느꼈던 점은 생각보다 도커가 메모리를 매우 많이 차지한다는 사실이다. 에어플로우 실행중에 도커만 10G정도를 차지했다... 메모리 사이즈가 작은 사람에게는 pip를 이용한 Local 테스트를 추천하고 싶다. (8G이하까지는 무조건 pip로 테스트해야 할 것 같다.)  

이번 글을 작성하며 operator가 task 실행 타입이라고 이해할 수 있게 됐다. 이렇게 생각하니 좀 더 다양하게 오퍼레이터를 이용하여 dag를 구성할 수 있을 것 같다. 또 에어플로우를 사용하는 3가지 방법, dag를 선언하는 3가지 방법에 대해서도 잘 정리된 것 같다.   

하지만 아직 dag 생성 시 전달 인자에 대해서는 제대로 설명할 수 없어 비워뒀다! 추후에 채워넣자.  

이어서 로컬에 실행한 에어플로우에 내가 만든 dag를 등록, 실행하는 글을 작성하려고 한다. 과연 이번엔 얼마나 걸릴지...?  

<br>
<br>

#### References
[Airflow Quick Start in Docker](https://airflow.apache.org/docs/apache-airflow/stable/start/docker.html)  
[Airflow Concepts - 목차](https://airflow.apache.org/docs/apache-airflow/stable/concepts/index.html)  
[Airflow Concepts - DAG](https://airflow.apache.org/docs/apache-airflow/stable/concepts/dags.html)  
[Airflow Concepts - Operators](https://airflow.apache.org/docs/apache-airflow/stable/concepts/operators.html)  
[Airflow Operators and Hooks](https://airflow.apache.org/docs/apache-airflow/stable/operators-and-hooks-ref.html)  
[Airflow Python API - airflow.operators](https://airflow.apache.org/docs/apache-airflow/stable/_api/airflow/operators/index.html)  

<br>
<br>
<br>