---
title: "250609 Programmers 점의 위치 구하기"
date: 2025-06-09 21:00:00 +0900
categories: [til]
tags: [Programmers, 코딩테스트, 입문, 코테]
layout: single
---

Programmers 코딩 테스트 입문 문제 풀이입니다.

---

## 📌 문제 요약

# [level 0] 점의 위치 구하기 - 120841 

[문제 링크](https://school.programmers.co.kr/learn/courses/30/lessons/120841) 

### 성능 요약

메모리: 4.46 MB, 시간: 0.01 ms

### 구분

코딩테스트 연습 > 코딩테스트 입문

### 채점결과

정확성: 100.0<br/>합계: 100.0 / 100.0

### 제출 일자

2025년 06월 09일 22:22:07

### 문제 설명

<p>사분면은 한 평면을 x축과 y축을 기준으로 나눈 네 부분입니다. 사분면은 아래와 같이 1부터 4까지 번호를매깁니다.<br>
<img src="https://grepp-programmers.s3.ap-northeast-2.amazonaws.com/files/production/b58d4788-42fa-44fa-af50-481907e65473/%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA%202022-07-07%20%E1%84%8B%E1%85%A9%E1%84%92%E1%85%AE%203.27.04%20%E1%84%87%E1%85%A9%E1%86%A8%E1%84%89%E1%85%A1%E1%84%87%E1%85%A9%E1%86%AB.png" title="" alt="스크린샷 2022-07-07 오후 3.27.04 복사본.png"></p>

<ul>
<li>x 좌표와 y 좌표가 모두 양수이면 제1사분면에 속합니다.</li>
<li>x 좌표가 음수, y 좌표가 양수이면 제2사분면에 속합니다.</li>
<li>x 좌표와 y 좌표가 모두 음수이면 제3사분면에 속합니다.</li>
<li>x 좌표가 양수, y 좌표가 음수이면 제4사분면에 속합니다.</li>
</ul>

<p>x  좌표 (x, y)를 차례대로 담은 정수 배열 <code>dot</code>이 매개변수로 주어집니다. 좌표 <code>dot</code>이 사분면 중 어디에 속하는지 1, 2, 3, 4 중 하나를 return 하도록 solution 함수를 완성해주세요.</p>

<hr>

<h4>제한사항</h4>

<ul>
<li><code>dot</code>의 길이 = 2</li>
<li><code>dot[0]</code>은 x좌표를, <code>dot[1]</code>은 y좌표를 나타냅니다</li>
<li>-500 ≤ <code>dot</code>의 원소 ≤ 500</li>
<li><code>dot</code>의 원소는 0이 아닙니다. </li>
</ul>

<hr>

<h4>입출력 예</h4>
<table class="table">
        <thead><tr>
<th>dot</th>
<th>result</th>
</tr>
</thead>
        <tbody><tr>
<td>[2, 4]</td>
<td>1</td>
</tr>
<tr>
<td>[-7, 9]</td>
<td>2</td>
</tr>
</tbody>
      </table>
<hr>

<h4>입출력 예 설명</h4>

<p>입출력 예 #1</p>

<ul>
<li><code>dot</code>이 [2, 4]로 x 좌표와 y 좌표 모두 양수이므로 제 1 사분면에 속합니다. 따라서 1을 return 합니다.</li>
</ul>

<p>입출력 예 #2</p>

<ul>
<li><code>dot</code>이 [-7, 9]로 x 좌표가 음수, y 좌표가 양수이므로 제 2 사분면에 속합니다. 따라서 2를 return 합니다.</li>
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
| 선언     | `int func() { ... }`         | `int arr[2][2] = {{...}};`   |
| 역할     | 어떤 동작(계산, 처리) 수행    | 여러 값 저장 및 관리         |
| 접근 방법 | `func()`                     | `arr[i][j]` (인덱스 접근)     |

→ `ans`는 함수가 아니라 **사분면 번호를 저장한 2차원 배열**입니다.

---

## ✅ 결론

`if` 문 없이 `dot[0] > 0`과 `dot[1] > 0` 결과를 배열의 인덱스로 활용해서  
**간단하고 효율적으로 사분면을 판별**하는 방식
