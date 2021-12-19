---
title: "[기본 자료구조] 부분합"
date: 2021-05-06 18:37:31 -0400
categories: algorithm
tags: partial-sum range-sum math
---

## 부분합(Partial Sum) or 누적합  
배열의 각 위치마다 시작위치 ~ 현재위치까지의 원소의 합을 구한 배열  
부분합 배열의 정의 : psum\[i] = score\[0] ~ score\[i]의 합  
=> 특정 구간의 합을 O(1)에 구할 수 있음  
=> score\[a] ~ score\[b] 까지의 합 = psum[b] - psum[a-1]  

<br>

## 2차원 배열에서의 구간합  

<br>
<br>
<br>
