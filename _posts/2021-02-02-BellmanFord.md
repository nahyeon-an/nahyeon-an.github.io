---
title: "[Graph Algorithm] Bellman-Ford 최단경로 찾기"
date: 2021-02-02 19:17:48 -0400
categories: algorithm
tags: graph shortest-path
---

### 벨만-포드 최단 경로 알고리즘
단일 시작점 최단 경로 알고리즘  
<span style="color:#c55f4e">**음수 간선**</span>이 있는 그래프에서도 최단경로를 찾을 수 있음  
음수 사이클의 존재 여부를 알 수 있음. 이를 통해 최단경로가 정의되는지 아닌지도 판별.  

### 구현
<span style="color:#c55f4e">**Edge 객체(정점 from, to, 가중치 w)**</span>를 구현해야 함  
Edge들을 모두 담는 배열 edgeList 생성  
이중 for문으로 구성 (모든 정점에 마다 모든 간선에 대해 거리 갱신)  
음수 사이클 확인 필수  
INF 값을 가장 긴 경로의 가중치보다 크도록 *잘* 설정해야 함 (실제로 INF값 때문에 값이 이상하게 저장된 경우가 있었다..)  
**시간복잡도 : O(|V||E|)**  

### Example
<img src="https://img1.daumcdn.net/thumb/R1280x0/?scode=mtistory2&fname=https%3A%2F%2Fblog.kakaocdn.net%2Fdn%2FbEW58d%2FbtqV08pCqRT%2FQXBXD12j0Bvtyd0chifwe0%2Fimg.jpg" />
<img src="https://img1.daumcdn.net/thumb/R1280x0/?scode=mtistory2&fname=https%3A%2F%2Fblog.kakaocdn.net%2Fdn%2FDqb3c%2FbtqWsI91KQd%2FKrNT1aeVZIkGqFCHtpiQq0%2Fimg.png">

### Pseudo-code
```
dist[ |V| ] = INF
dist[start] = 0

for |V-1| times :
    for 모든 e in edgeList :
        if ( dist[e.to] > dist[e.from] + e.w ) :
            dist[e.to] = dist[e.from] + e.w

for 모든 e in edgeList :
    if ( dist[e.to] > dist[e.from] + e.w ):
        isCycle = True
        break

if (isCycle) :
    "최단 경로는 존재하지 않는다"
```

<br>
<br>
<br>