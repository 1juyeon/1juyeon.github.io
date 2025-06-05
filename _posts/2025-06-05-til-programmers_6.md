---
title: "250605 Programmers 순서 쌍의 개수"
date: 2025-06-05 21:00:00 +0900
categories: [til]
tags: [Programmers, 코딩테스트, 입문, 코테]
layout: single
---

Programmers 코딩 테스트 입문 문제 풀이입니다.

---

## 📌 문제 요약

# [level 0] 순서쌍의 개수 - 120836 

[문제 링크](https://school.programmers.co.kr/learn/courses/30/lessons/120836) 

### 성능 요약

메모리: 4.13 MB, 시간: 2.37 ms

### 구분

코딩테스트 연습 > 코딩테스트 입문

### 채점결과

정확성: 100.0<br/>합계: 100.0 / 100.0

### 제출 일자

2025년 06월 05일 21:20:38

### 문제 설명

<p>순서쌍이란 두 개의 숫자를 순서를 정하여 짝지어 나타낸 쌍으로 (a, b)로 표기합니다. 자연수 <code>n</code>이 매개변수로 주어질 때 두 숫자의 곱이 <code>n</code>인 자연수 순서쌍의 개수를 return하도록 solution 함수를 완성해주세요.</p>

<hr>

<h5>제한사항</h5>

<ul>
<li>1 ≤ n ≤ 1,000,000</li>
</ul>

<hr>

<h5>입출력 예</h5>
<table class="table">
        <thead><tr>
<th>n</th>
<th>result</th>
</tr>
</thead>
        <tbody><tr>
<td>20</td>
<td>6</td>
</tr>
<tr>
<td>100</td>
<td>9</td>
</tr>
</tbody>
      </table>
<hr>

<h5>입출력 예 설명</h5>

<p>입출력 예 #1</p>

<ul>
<li><code>n</code>이 20 이므로 곱이 20인 순서쌍은 (1, 20), (2, 10), (4, 5), (5, 4), (10, 2), (20, 1) 이므로 6을 return합니다.</li>
</ul>

<p>입출력 예 #2</p>

<ul>
<li><code>n</code>이 100 이므로 곱이 100인 순서쌍은 (1, 100), (2, 50), (4, 25), (5, 20), (10, 10), (20, 5), (25, 4), (50, 2), (100, 1) 이므로 9를 return합니다.</li>
</ul>


> 출처: 프로그래머스 코딩 테스트 연습, https://school.programmers.co.kr/learn/challenges
---

## 🧠 내 풀이 (C 언어)

```c
#include <stdio.h>
#include <stdbool.h>
#include <stdlib.h>

int solution(int n) {
    int answer = 0;
    int temp=0;
    int cnt=0;
    
    for(int i=1; i<n+1; i++){
        temp=n/i;
        if(i*temp==n) {
            answer++;
        }
    }

    return answer;
}
```

## 다른 사람의 풀이

```c
#include <stdio.h>
#include <stdbool.h>
#include <stdlib.h>

int solution(int n) {
    int answer = 0;
    int count = 0;
    for(int i=1;i<=n;i++){
        if(n%i==0)
            answer++;
    }
    return answer;
}
```

## ✍️ 회고

> 이번에는 개수만 카운트하도록 간단하게 잘 풀었다고 생각했는데 더 간단한 방법이 있었다.