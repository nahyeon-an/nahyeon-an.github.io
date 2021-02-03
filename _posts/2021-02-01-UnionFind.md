---
title: "[Graph Algorithm] Union-Find"
date: 2021-02-01 18:30:28 -0400
categories: update
---

### Union-Find
상호 배타적 집합(disjoint set)을 표현할 때 사용하는 트리 자료구조  

#### 상호 배타적 집합(disjoint set)
부분 집합으로 나누었을 때 공통 원소가 없는 집합  
1. _"배열"로 표현_  

    belongTo[i] := i번 원소가 속하는 집합 번호 배열  
    초기화 : belongTo[i] = i, 0 <= i < n  
    union : O(n), 모든 원소를 순회하여 집합을 이동  
    find : O(1)  
2. _"트리"로 표현_  

    같은 집합의 원소들끼리 트리를 구성 = 트리들의 집합  
    모든 노드는 부모에 대한 포인터만 가지면 됨  
    (루트노드의 경우 자기 자신을 가리킴)    
    각 노드를 <span style="color:#56373c">*1차원 배열*</span>로 표현 (자신의 부모 번호를 저장)  
    union : 각 트리의 루트를 찾아, 하나를 다른 한 쪽의 자손으로 이동시킴, O(트리의 높이)  
    find : 각 원소가 포함된 트리의 루트를 찾아 비교, O(트리의 높이)  

#### 3개의 연산
1. __초기화__  
    n개의 원소가 각각 집합을 가지도록 초기화  
2. __union__  
    두 원소 a, b가 주어질 때 이들이 속한 두 집합을 하나로 합치기  
3. __find__  
    어떤 원소 a가 주어질 때 이 원소가 속한 집합을 반환  

#### 최적화


[union-find]: https://brenden.tistory.com/33
[others..]:   https://brenden.tistory.com/38
