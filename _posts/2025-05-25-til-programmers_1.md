---
title: "250525 Programmers 문자 반복 출력하기 "
date: 2025-05-25 10:00:00 +0900
categories: [til]
tags: [Programmers, 코딩테스트, 입문, 코테]
layout: single
---

Programmers 코딩 테스트 입문 문제 풀이

## 문자 반복 출력하기

## 📌 문제 요약

**문제 이름:** 문자 반복 출력하기  
**난이도:** 입문  

---

## 🔍 문제 설명

> 문자열 `my_string`과 정수 `n`이 매개변수로 주어질 때,  
> `my_string`의 각 문자를 `n`번 반복한 문자열을 return 하도록 함수 작성.

---

## 내 풀이

```C
#include <stdio.h>
#include <stdbool.h>
#include <stdlib.h>
#include <string.h>

// 파라미터로 주어지는 문자열은 const로 주어집니다. 변경하려면 문자열을 복사해서 사용하세요.
char* solution(const char* my_string, int n) {
    // return 값은 malloc 등 동적 할당을 사용해주세요. 할당 길이는 상황에 맞게 변경해주세요.
    int len = strlen(my_string);
    char* answer = (char*)malloc(sizeof(int)*(len*n));
    int cnt = 0;
    
    memset(answer, 0, sizeof(int)*(len*n));
    
    for(int i=0; i<len; i++){
        for(int j=0; j<n; j++){
            answer[i+j+cnt] = my_string[i];
        }
        cnt += n-1;
    }  
    return answer;
}


## 더 좋은 방식의 풀이
```C
#include <stdio.h>
#include <stdbool.h>
#include <stdlib.h>
#include<string.h>

// 파라미터로 주어지는 문자열은 const로 주어집니다. 변경하려면 문자열을 복사해서 사용하세요.
char* solution(const char* my_string, int n) {
    // return 값은 malloc 등 동적 할당을 사용해주세요. 할당 길이는 상황에 맞게 변경해주세요.
    char* answer = (char*)malloc(strlen(my_string) * n + 1);
    for(int i = 0; i < strlen(my_string) * n; i++){
        answer[i] = my_string[i / n];
    }
    answer[strlen(my_string) * n] = '\0';

    return answer;
}


>> for문을 두 줄 사용하는 방식보다 전체 길이를 한 번 돌면서 my_string[i/n] 을 사용한 점이 훨씬 효율적이라고 생각했음