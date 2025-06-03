---
title: "250603 Programmers 진료 순서 정하기"
date: 2025-06-03 10:00:00 +0900
categories: [til]
tags: [Programmers, 코딩테스트, 입문, 코테, sort, 내림차순]
layout: single
---

Programmers 코딩 테스트 입문 문제 풀이입니다.

---

## 📌 문제 요약

# [level 0] 진료 순서 정하기 - 120835 

[문제 링크](https://school.programmers.co.kr/learn/courses/30/lessons/120835) 

### 성능 요약

메모리: 4.16 MB, 시간: 0.01 ms

### 구분

코딩테스트 연습 > 코딩테스트 입문

### 채점결과

정확성: 100.0<br/>합계: 100.0 / 100.0

### 제출 일자

2025년 06월 03일 20:30:30

### 문제 설명

<p>외과의사 머쓱이는 응급실에 온 환자의 응급도를 기준으로 진료 순서를 정하려고 합니다. 정수 배열 <code>emergency</code>가 매개변수로 주어질 때 응급도가 높은 순서대로 진료 순서를 정한 배열을 return하도록 solution 함수를 완성해주세요.</p>

<hr>

<h5>제한사항</h5>

<ul>
<li>중복된 원소는 없습니다.</li>
<li>1 ≤ <code>emergency</code>의 길이 ≤ 10</li>
<li>1 ≤ <code>emergency</code>의 원소 ≤ 100</li>
</ul>

<hr>

<h5>입출력 예</h5>
<table class="table">
        <thead><tr>
<th>emergency</th>
<th>result</th>
</tr>
</thead>
        <tbody><tr>
<td>[3, 76, 24]</td>
<td>[3, 1, 2]</td>
</tr>
<tr>
<td>[1, 2, 3, 4, 5, 6, 7]</td>
<td>[7, 6, 5, 4, 3, 2, 1]</td>
</tr>
<tr>
<td>[30, 10, 23, 6, 100]</td>
<td>[2, 4, 3, 5, 1]</td>
</tr>
</tbody>
      </table>
<hr>

<h5>입출력 예 설명</h5>

<p>입출력 예 #1</p>

<ul>
<li><code>emergency</code>가 [3, 76, 24]이므로 응급도의 크기 순서대로 번호를 매긴 [3, 1, 2]를 return합니다.</li>
</ul>

<p>입출력 예 #2</p>

<ul>
<li><code>emergency</code>가 [1, 2, 3, 4, 5, 6, 7]이므로 응급도의 크기 순서대로 번호를 매긴 [7, 6, 5, 4, 3, 2, 1]를 return합니다.</li>
</ul>

<p>입출력 예 #3</p>

<ul>
<li><code>emergency</code>가 [30, 10, 23, 6, 100]이므로 응급도의 크기 순서대로 번호를 매긴 [2, 4, 3, 5, 1]를 return합니다.</li>
</ul>


> 출처: 프로그래머스 코딩 테스트 연습, https://school.programmers.co.kr/learn/challenges

---

## 🧠 내 풀이 (C 언어)

```c
#include <stdio.h>
#include <stdbool.h>
#include <stdlib.h>

// emergency_len은 배열 emergency의 길이입니다.
int* solution(int emergency[], size_t emergency_len) {
    // return 값은 malloc 등 동적 할당을 사용해주세요. 할당 길이는 상황에 맞게 변경해주세요.
    int* answer = (int*)malloc(sizeof(size_t)*emergency_len);
    int* answer_2 = (int*)malloc(sizeof(size_t)*emergency_len);
    int temp;
    
    for(int a=0; a<emergency_len; a++){
        answer_2[a]=emergency[a];
    }
    
    for(int i=0; i<emergency_len; i++){
        for(int j=0; j<emergency_len; j++){
            if(emergency[i] > emergency[j]){
                temp = emergency[i];
                emergency[i] = emergency[j];
                emergency[j] = temp;
            }
        }
    }
    
    for(int b=0; b<emergency_len; b++){
        for(int c=0; c<emergency_len; c++){
            if(emergency[b]==answer_2[c]){
                answer[c] = b+1;
            }
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

// emergency_len은 배열 emergency의 길이입니다.
int* solution(int emergency[], size_t emergency_len) {
    // return 값은 malloc 등 동적 할당을 사용해주세요. 할당 길이는 상황에 맞게 변경해주세요.
    int* answer = (int*)malloc(sizeof(int) * emergency_len);
    for(int i = 0; i < emergency_len; i++){
        answer[i] = 1;
        for(int j = 0; j < emergency_len;j ++){
            if (emergency[j] > emergency[i]) answer[i]++;
        }
    }
    return answer;
}
```

## ✍️ 회고

> 오름차순/내림차순을 구현하는 알고리즘 (SORT)가 필요하다고 생각되어

```c
#include <stdio.h>

int main(void){
    int i;
    int j;
    int temp;
    int num[5] = {0};

    printf("값을 5개 입력해주세요:");
    for(i=0; i<5; i++){
        scanf_s("%d", &num[i]);
    }

    for(i=0; i<5; i++){
        for(j=0; j<5; j++){
            if(num[i]>num[j]){ // <면 오름차순
                temp = num[i];
                num[i] = num[j];
                num[j] = temp;
            }
        }
    }

    printf("정렬 결과는 : ");
    for(i=0;i<5;i++){
        printf("%d ",num[i]);
    }

}
```
>위와 같은 코드를 참고했다. 그래서 내림차순으로 정렬한 배열과 기존 배열 두 가지를 비교하여 최종 정답 배열에 순위를 넣어 답을 내었다.
그런데 다른 사람들의 풀이를 보니 자기보다 큰 값들의 개수를 카운트해서 넣어서 푼 방법을 보았다. 단순 순위를 도출하는것이니 배열 선언과 for문을 줄여 푸는 방식이 더 효율적이라고 생각해서 작성하게 되었다.