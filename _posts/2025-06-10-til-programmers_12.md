---
title: "250610 Programmers 2차원으로 만들기"
date: 2025-06-10 21:00:00 +0900
categories: [til]
tags: [Programmers, 코딩테스트, 입문, 코테]
layout: single
---

Programmers 코딩 테스트 입문 문제 풀이입니다.

---

## 📌 문제 요약

# [level 0] 2차원으로 만들기 - 120842 

[문제 링크](https://school.programmers.co.kr/learn/courses/30/lessons/120842) 

### 성능 요약

메모리: 4.19 MB, 시간: 0.09 ms

### 구분

코딩테스트 연습 > 코딩테스트 입문

### 채점결과

정확성: 100.0<br/>합계: 100.0 / 100.0

### 제출 일자

2025년 06월 10일 23:53:14

### 문제 설명

<p>정수 배열 <code>num_list</code>와 정수&nbsp;<code>n</code>이 매개변수로 주어집니다. <code>num_list</code>를 다음 설명과 같이 2차원 배열로 바꿔 return하도록 solution 함수를 완성해주세요.</p>

<p><code>num_list</code>가 [1, 2, 3, 4, 5, 6, 7, 8] 로 길이가 8이고 <code>n</code>이 2이므로 <code>num_list</code>를 2 * 4 배열로 다음과 같이 변경합니다. 2차원으로 바꿀 때에는 num_list의 원소들을 앞에서부터 n개씩 나눠 2차원 배열로 변경합니다.</p>
<table class="table">
        <thead><tr>
<th>num_list</th>
<th>n</th>
<th>result</th>
</tr>
</thead>
        <tbody><tr>
<td>[1, 2, 3, 4, 5, 6, 7, 8]</td>
<td>2</td>
<td>[[1, 2], [3, 4], [5, 6], [7, 8]]</td>
</tr>
</tbody>
      </table>
<hr>

<h5>제한사항</h5>

<ul>
<li><code>num_list</code>의 길이는&nbsp;<code>n</code>의 배 수개입니다.</li>
<li>0 ≤ <code>num_list</code>의 길이 ≤ 150</li>
<li>2 ≤ <code>n</code> &lt; <code>num_list</code>의 길이</li>
</ul>

<hr>

<h5>입출력 예</h5>
<table class="table">
        <thead><tr>
<th>num_list</th>
<th>n</th>
<th>result</th>
</tr>
</thead>
        <tbody><tr>
<td>[1, 2, 3, 4, 5, 6, 7, 8]</td>
<td>2</td>
<td>[[1, 2], [3, 4], [5, 6], [7, 8]]</td>
</tr>
<tr>
<td>[100, 95, 2, 4, 5, 6, 18, 33, 948]</td>
<td>3</td>
<td>[[100, 95, 2], [4, 5, 6], [18, 33, 948]]</td>
</tr>
</tbody>
      </table>
<hr>

<h5>입출력 예 설명</h5>

<p>입출력 예 #1</p>

<ul>
<li><code>num_list</code>가 [1, 2, 3, 4, 5, 6, 7, 8] 로 길이가 8이고 <code>n</code>이 2이므로 2 * 4 배열로 변경한 [[1, 2], [3, 4], [5, 6], [7, 8]] 을 return합니다.</li>
</ul>

<p>입출력 예 #2</p>

<ul>
<li><code>num_list</code>가 [100, 95, 2, 4, 5, 6, 18, 33, 948] 로 길이가 9이고 <code>n</code>이 3이므로 3 * 3 배열로 변경한 [[100, 95, 2], [4, 5, 6], [18, 33, 948]] 을 return합니다.</li>
</ul>


> 출처: 프로그래머스 코딩 테스트 연습, https://school.programmers.co.kr/learn/challenges

---

## * 내 풀이 (C 언어)

{% raw %}
```
#include <stdio.h>
#include <stdbool.h>
#include <stdlib.h>

// dot_len은 배열 dot의 길이입니다.
int solution(int dot[], size_t dot_len) {
    int answer = 0;
    
    if(dot[0]>0&&dot[1]>0){
        answer=1;
    }else if(dot[0]<0&&dot[1]>0){
        answer=2;
    }else if(dot[0]<0&&dot[1]<0){
        answer=3;
    }else if(dot[0]>0&&dot[1]<0){
        answer=4;
    }
    
    return answer;
}
```

## * 다른 사람의 풀이

```c
#include <stdio.h>
#include <stdbool.h>
#include <stdlib.h>

// dot_len은 배열 dot의 길이입니다.
int solution(int dot[], size_t dot_len) 
{	
    {% raw %}
int ans[2][2] = {{3,2}, {4,1}};
{% endraw %}
    return ans[dot[0] > 0][dot[1] > 0];
}
```
{% endraw %}


## ✍️ 회고

- `dot[0] > 0`: x 좌표가 양수인지 검사 → 결과는 `0` 또는 `1`
- `dot[1] > 0`: y 좌표가 양수인지 검사 → 결과는 `0` 또는 `1`

이 두 개는 각각 2가지 경우가 있으므로  
총 **2 × 2 = 4가지** 경우를 2차원 배열로 표현할 수 있음

---

## 🗂️ 배열 `ans[2][2]` 설명

```c
int ans[2][2] = {
    {3, 2},  // x <= 0일 때 (행 0)
    {4, 1}   // x >  0일 때 (행 1)
};
```

| x > 0 (행) | y > 0 (열) | ans[x][y] 값 | 사분면 |
|------------|------------|--------------|--------|
| 0 (false)  | 0 (false)  | ans[0][0] = 3 | 3사분면 |
| 0 (false)  | 1 (true)   | ans[0][1] = 2 | 2사분면 |
| 1 (true)   | 0 (false)  | ans[1][0] = 4 | 4사분면 |
| 1 (true)   | 1 (true)   | ans[1][1] = 1 | 1사분면 |

---

## 📎 함수 vs 배열 차이

| 항목     | 함수                          | 배열                         |
|----------|-------------------------------|------------------------------|
| 선언     | `int func() { ... }`         | `int arr[2][2] = {...};`   |
| 역할     | 어떤 동작(계산, 처리) 수행    | 여러 값 저장 및 관리         |
| 접근 방법 | `func()`                     | `arr[i][j]` (인덱스 접근)     |

→ `ans`는 함수가 아니라 **사분면 번호를 저장한 2차원 배열**입니다.

---

## ✅ 결론

`if` 문 없이 `dot[0] > 0`과 `dot[1] > 0` 결과를 배열의 인덱스로 활용해서  
**간단하고 효율적으로 사분면을 판별**하는 방식
