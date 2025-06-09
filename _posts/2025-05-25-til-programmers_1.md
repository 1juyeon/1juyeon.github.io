---
title: "[TIL] 250525 Programmers 문자 반복 출력하기"
date: 2025-05-25 10:00:00 +0900
categories: [til]
tags: [Programmers, 코딩테스트, 입문, 코테]
layout: single
---

Programmers 코딩 테스트 입문 문제 풀이입니다.

---

## 📌 문제 요약

- **문제 이름:** 문자 반복 출력하기  
- **난이도:** 입문  
- **설명:** 문자열 `my_string`과 정수 `n`이 주어질 때, 각 문자를 `n`번 반복한 문자열을 반환

---

## 🧠 내 풀이 (C 언어)

```c
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
```

### 🔎 설명
- `strlen`으로 원래 문자열 길이를 구한 뒤,
- 각 문자를 `n`번 반복하여 `answer`에 차례로 넣음
- 마지막에 널 문자 `\0`을 꼭 붙여서 종료

---

## ✅ 더 나은 풀이 (다른 사람의 코드)

```c
#include <stdio.h>
#include <stdbool.h>
#include <stdlib.h>
#include <string.h>

char* solution(const char* my_string, int n) {
    int len = strlen(my_string);
    char* answer = (char*)malloc(len * n + 1);

    for (int i = 0; i < len * n; i++) {
        answer[i] = my_string[i / n];
    }
    answer[len * n] = '\0';
    return answer;
}
```

### 💬 느낀 점
- 반복문을 하나만 써서 더 깔끔하고 효율적
- `i / n`을 통해 원래 문자를 n번 반복한 것처럼 매핑
- 앞으로 반복 로직을 단순화할 때 참고할 가치 있음

---

## ✍️ 회고

> 처음에는 중첩 루프가 직관적이라고 생각했지만,  
> **i/n 패턴을 이용한 접근이 더 간결하고 효율적**이라고 생각
