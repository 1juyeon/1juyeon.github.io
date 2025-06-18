---
title: "250618 Programmers 문자열 출력하기"
date: 2025-06-18 21:00:00 +0900
categories: [til]
tags: [Programmers, 코딩테스트, 기초, 코테]
layout: single
---

Programmers 코딩 테스트 기초 문제 풀이입니다.

---

## 📌 문제 요약

# [level 0] 문자열 출력하기 - 181952 

[문제 링크](https://school.programmers.co.kr/learn/courses/30/lessons/181952) 

### 성능 요약

메모리: 4.19 MB, 시간: 2.45 ms

### 구분

코딩테스트 연습 > 코딩 기초 트레이닝

### 채점결과

정확성: 100.0<br/>합계: 100.0 / 100.0

### 제출 일자

2025년 06월 04일 20:02:56

### 문제 설명

<p>문자열 <code>str</code>이 주어질 때, <code>str</code>을 출력하는 코드를 작성해 보세요.</p>

<hr>

<h5>제한사항</h5>

<ul>
<li>1 ≤ <code>str</code>의 길이 ≤ 1,000,000</li>
<li><code>str</code>에는 공백이 없으며, 첫째 줄에 한 줄로만 주어집니다.</li>
</ul>

<hr>

<h5>입출력 예</h5>

<p>입력 #1</p>
<div class="highlight"><pre class="codehilite"><code>HelloWorld!
</code></pre></div>
<p>출력 #1</p>
<div class="highlight"><pre class="codehilite"><code>HelloWorld!
</code></pre></div>

> 출처: 프로그래머스 코딩 테스트 연습, https://school.programmers.co.kr/learn/challenges
---

## * 내 풀이 (C 언어)

```c
#include <stdio.h>
#include <string.h>
#define LEN_INPUT 11

int main(void) {
    char s1[LEN_INPUT];
    int a;
    scanf("%s %d", s1, &a);
    
    int len = strlen(s1);
    
    for (int j=0; j<a; j++){
        for(int i=0; i<len; i++){
            printf("%c", s1[i]);
        } 
    }

    return 0;
}
```

## 다른 사람 풀이
```c
#include <stdio.h>
#define LEN_INPUT 11

int main(void) {
    char s1[LEN_INPUT];
    int a;
    scanf("%s %d", s1, &a);
    for (int i = 0; i < a; ++i) {
        printf("%s", s1);
    }
    return 0;
}
```

## ✍️ 회고
> 이 문제에서는 굳이 문자열 하나하나에 접근해서 출력하지 않아도 조금 더 효율적으로 문자열 자체를 출력하는것이 좋은 것 같다.

