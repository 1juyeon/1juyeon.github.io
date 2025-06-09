---
title: "250530 Programmers 배열 자르기"
date: 2025-05-30 10:00:00 +0900
categories: [til]
tags: [Programmers, 코딩테스트, 입문, 코테]
layout: single
---

Programmers 코딩 테스트 입문 문제 풀이입니다.

---

## 📌 문제 요약

- **문제 이름:** 배열 자르기
- **난이도:** 입문  
- **설명:**
정수 배열 numbers와 정수 num1, num2가 매개변수로 주어질 때, numbers의 num1번 째 인덱스부터 num2번째 인덱스까지 자른 정수 배열을 return 하도록 solution 함수를 완성해보세요.

---

## 🧠 내 풀이 (C 언어)

```c
#include <stdio.h>
#include <stdbool.h>
#include <stdlib.h>
#include <string.h>

// numbers_len은 배열 numbers의 길이입니다.
int* solution(int numbers[], size_t numbers_len, int num1, int num2) {
    // return 값은 malloc 등 동적 할당을 사용해주세요. 할당 길이는 상황에 맞게 변경해주세요.
    int* answer = (int*)malloc(numbers_len);
    int cnt = 0;
    
    for(int i=num1; i<num2+1; i++){
        answer[cnt] = numbers[i];
        cnt++;
    }
    return answer;
}
```

### 🔎 설명


---

## ✅다른 사람의 코드

```c
#include <stdio.h>
#include <stdbool.h>
#include <stdlib.h>

// numbers_len은 배열 numbers의 길이입니다.
int* solution(int numbers[], size_t numbers_len, int num1, int num2) {
    // return 값은 malloc 등 동적 할당을 사용해주세요. 할당 길이는 상황에 맞게 변경해주세요.
    int* answer = (int*)malloc((num2 - num1 + 1) * sizeof(int));
    for(int i = num1; i <= num2; i++){
        answer[i - num1] = numbers[i]; 
    }
    return answer;
}
```


## ✍️ 회고

> 자꾸 새로 cnt를 선언해서 문제를 풀게 되는데 i-num1 같은 방법을 사용하는것이 문제를 풀 때 생각이 나면 좋을 것 같다
그리고 동적 할당을 할 때 지금은 그냥 넉넉하게 잡지만 좀 더 깊게 생각해서 필요한 만큼만 선언하는 연습을 해야겠다