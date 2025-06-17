---
title: "250618 Programmers "
date: 2025-06-18 21:00:00 +0900
categories: [til]
tags: [Programmers, 코딩테스트, 기초, 코테]
layout: single
---

Programmers 코딩 테스트 기초 문제 풀이입니다.

---

## 📌 문제 요약

# [level 0] 공 던지기 - 120843 

[문제 링크](https://school.programmers.co.kr/learn/courses/30/lessons/120843) 

### 성능 요약

메모리: 4.14 MB, 시간: 0.01 ms

### 구분

코딩테스트 연습 > 코딩테스트 입문

### 채점결과

정확성: 100.0<br/>합계: 100.0 / 100.0

### 제출 일자

2025년 06월 15일 21:43:42

### 문제 설명

<p>머쓱이는 친구들과 동그랗게 서서 공 던지기 게임을 하고 있습니다. 공은 1번부터 던지며 오른쪽으로 한 명을 건너뛰고 그다음 사람에게만 던질 수 있습니다. 친구들의 번호가 들어있는 정수 배열 <code>numbers</code>와 정수 <code>K</code>가 주어질 때, <code>k</code>번째로 공을 던지는 사람의 번호는 무엇인지 return 하도록 solution 함수를 완성해보세요.</p>

<hr>

<h5>제한사항</h5>

<ul>
<li>2 &lt; <code>numbers</code>의 길이 &lt; 100</li>
<li>0 &lt; <code>k</code> &lt; 1,000</li>
<li><code>numbers</code>의 첫 번째와 마지막 번호는 실제로 바로 옆에 있습니다.</li>
<li><code>numbers</code>는 1부터 시작하며 번호는 순서대로 올라갑니다.</li>
</ul>

<hr>

<h5>입출력 예</h5>
<table class="table">
        <thead><tr>
<th>numbers</th>
<th>k</th>
<th>result</th>
</tr>
</thead>
        <tbody><tr>
<td>[1, 2, 3, 4]</td>
<td>2</td>
<td>3</td>
</tr>
<tr>
<td>[1, 2, 3, 4, 5, 6]</td>
<td>5</td>
<td>3</td>
</tr>
<tr>
<td>[1, 2, 3]</td>
<td>3</td>
<td>2</td>
</tr>
</tbody>
      </table>
<hr>

<h5>입출력 예 설명</h5>

<p>입출력 예 #1</p>

<ul>
<li>1번은 첫 번째로 3번에게 공을 던집니다.</li>
<li>3번은 두 번째로 1번에게 공을 던집니다.</li>
</ul>

<p>입출력 예 #2</p>

<ul>
<li>1번은 첫 번째로 3번에게 공을 던집니다.</li>
<li>3번은 두 번째로 5번에게 공을 던집니다.</li>
<li>5번은 세 번째로 1번에게 공을 던집니다.</li>
<li>1번은 네 번째로 3번에게 공을 던집니다.</li>
<li>3번은 다섯 번째로 5번에게 공을 던집니다.</li>
</ul>

<p>입출력 예 #3</p>

<ul>
<li>1번은 첫 번째로 3번에게 공을 던집니다.</li>
<li>3번은 두 번째로 2번에게 공을 던집니다.</li>
<li>2번은 세 번째로 1번에게 공을 던집니다.</li>
</ul>

<p>※ 공지 - 2023년 1월 25일 테스트 케이스가 추가되었습니다. 기존에 제출한 코드가 통과하지 못할 수도 있습니다.</p>


> 출처: 프로그래머스 코딩 테스트 연습, https://school.programmers.co.kr/learn/challenges
---

## * 내 풀이 (C 언어)

```c
#include <stdio.h>
#include <stdbool.h>
#include <stdlib.h>

// numbers_len은 배열 numbers의 길이입니다.
int solution(int numbers[], size_t numbers_len, int k) {    
    return numbers[(k - 1) * 2 % numbers_len];
}
```



## ✍️ 회고

## ✅ 풀이 해석 (쉬운 설명)

```c
return numbers[(k - 1) * 2 % numbers_len];
```

이 코드는 **공을 던지는 규칙**을 수학적으로 계산하여  
**k번째로 공을 던지는 사람의 번호**를 바로 찾아내는 방식입니다.

---

### 🔁 게임 규칙 정리

- 처음엔 **1번** 사람이 공을 가짐.
- 공은 항상 **한 명 건너뛰고**, 그다음 사람에게 던짐  
  → 즉, **2칸씩 이동**
- 배열은 **원형 구조** (끝나면 처음으로 다시 이어짐)

---

### 🧠 핵심 아이디어

- 공은 던질 때마다 **2칸씩 점프**.
- `k번째`로 공을 던지는 사람이 누군지 계산하려면:

  ```
  (k - 1) * 2
  ```

  > → (k번째 던짐이므로 첫 던짐은 자기 자신이므로 -1)

- 이 값을 배열 길이로 나눈 나머지(`% numbers_len`)는  
  **공이 도달한 인덱스 위치**를 의미함.

---

### 📌 예시

#### 예: `numbers = [1, 2, 3, 4]`, `k = 2`

- `(2 - 1) * 2 = 2`
- `2 % 4 = 2`
- `numbers[2] = 3`

✅ 정답: `3`

---

### ✅ 한 줄 요약

> `k번째로 공을 던지는 사람`은  
> **처음 위치에서 2칸씩 (k-1)번 이동한 자리이며**,  
> **배열을 한 바퀴 도는 경우도 고려해 mod 연산으로 위치를 계산**합니다.

