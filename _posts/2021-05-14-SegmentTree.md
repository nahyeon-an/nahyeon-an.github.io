---
title: "[Tree] Segment Tree, 세그먼트 트리"
date: 2021-05-14 23:49:09 -0400
categories: algorithm
tags: python segment-tree partial-sum
---

<br>
<br>

## Segment Tree  
특별한 자료 구조  
특정한 구간에 대한 조회 및 **업데이트** 쿼리를 *여러번* 처리할 때 유리함  
(배열을 이용하여 부분합을 구하는 방법은 업데이트 시 효율성이 떨어짐)  
ex) 구간 합/곱, 구간 내 최대/최소값  

<br>

### Example  
구간 내 최솟값을 저장하는 세그먼트 트리  
![세그먼트 트리 예시](https://img1.daumcdn.net/thumb/R1280x0/?scode=mtistory2&fname=https%3A%2F%2Fblog.kakaocdn.net%2Fdn%2FdkqGkB%2Fbtq4X6IMZt5%2FkI5r7DO7D9ruQyQxdduEvK%2Fimg.jpg)  
![초기화 경우의 수](https://img1.daumcdn.net/thumb/R1280x0/?scode=mtistory2&fname=https%3A%2F%2Fblog.kakaocdn.net%2Fdn%2FQsebw%2Fbtq5aqVDUPM%2Fs7A7R4CRN8x3HQegqkiIBk%2Fimg.jpg)  
![업데이트 방법](https://img1.daumcdn.net/thumb/R1280x0/?scode=mtistory2&fname=https%3A%2F%2Fblog.kakaocdn.net%2Fdn%2Fc72G8X%2Fbtq5hkTrahw%2FJ62u4OOTK650wSas3RD4y1%2Fimg.jpg)  

<br>

### 구현  
<script src="https://gist.github.com/nahyeon-an/b1885a90f9cf4d0b4033f44dfa740a8c.js"></script>

<br>

#### References
[백준 세그먼트 트리에 관한 설명 게시글](https://www.acmicpc.net/blog/view/9)

<br>
<br>
<br>