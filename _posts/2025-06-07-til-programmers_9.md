---
title: "250607 Programmers 가위 바위 보"
date: 2025-06-07 21:00:00 +0900
categories: [til]
tags: [Programmers, 코딩테스트, 입문, 코테]
layout: single
---

Programmers 코딩 테스트 입문 문제 풀이입니다.

---

## 📌 문제 요약

# [level 0] 가위 바위 보 - 120839 

[문제 링크](https://school.programmers.co.kr/learn/courses/30/lessons/120839) 

### 성능 요약

메모리: 4.39 MB, 시간: 0.01 ms

### 구분

코딩테스트 연습 > 코딩테스트 입문

### 채점결과

정확성: 100.0<br/>합계: 100.0 / 100.0

### 제출 일자

2025년 06월 07일 23:50:56

### 문제 설명

<p>가위는 2 바위는 0 보는 5로 표현합니다. 가위 바위 보를 내는 순서대로 나타낸 문자열 <code>rsp</code>가 매개변수로 주어질 때, rsp에 저장된 가위 바위 보를  모두 이기는 경우를 순서대로 나타낸 문자열을 return하도록 solution 함수를 완성해보세요.</p>

<hr>

<h5>제한사항</h5>

<ul>
<li>0 &lt; <code>rsp</code>의 길이 ≤ 100</li>
<li> <code>rsp</code>와 길이가 같은 문자열을 return 합니다.</li>
<li> <code>rsp</code>는 숫자 0, 2, 5로 이루어져 있습니다.</li>
</ul>

<hr>

<h5>입출력 예</h5>
<table class="table">
        <thead><tr>
<th>rsp</th>
<th>result</th>
</tr>
</thead>
        <tbody><tr>
<td>"2"</td>
<td>"0"</td>
</tr>
<tr>
<td>"205"</td>
<td>"052"</td>
</tr>
</tbody>
      </table>
<hr>

<h5>입출력 예 설명</h5>

<p>입출력 예 #1</p>

<ul>
<li>"2"는 가위이므로 바위를 나타내는 "0"을 return 합니다.</li>
</ul>

<p>입출력 예 #2</p>

<ul>
<li>"205"는 순서대로 가위, 바위, 보이고 이를 모두 이기려면 바위, 보, 가위를 순서대로 내야하므로 “052”를 return합니다.</li>
</ul>


> 출처: 프로그래머스 코딩 테스트 연습, https://school.programmers.co.kr/learn/challenges
---

## * 내 풀이 (C 언어)

```c
#include <stdio.h>
#include <stdbool.h>
#include <stdlib.h>
#include <string.h>

// 파라미터로 주어지는 문자열은 const로 주어집니다. 변경하려면 문자열을 복사해서 사용하세요.
char* solution(const char* rsp) {
    // return 값은 malloc 등 동적 할당을 사용해주세요. 할당 길이는 상황에 맞게 변경해주세요.
    char* answer = (char*)malloc(strlen(rsp));
    int i=0;
    
    for(i=0; i<strlen(rsp); i++){
        if(rsp [i] =='2'){
            answer[i] = '0';
        }else if(rsp [i] =='0'){
            answer[i] ='5';
        }else if(rsp[i]=='5'){
            answer[i] = '2';     
        }
    }
    answer[i]='\0';
    return answer;
}
```


## ✍️ 회고

> 이번에 여러 번 실수했던 부분은 늘 String 타입을 int len = strlen(test); 이런식으로 선언해서 사용했는데, 배열의 길이를 구하는 것은 조금 다르다는 점. 따로 복습이 필요하다. 그리고 rsp[i]=='2' 와 같이 비교할때 작은 따옴표를 붙이지 않는 실수를 했다. 이과 같은 점을 주의해야겠다.