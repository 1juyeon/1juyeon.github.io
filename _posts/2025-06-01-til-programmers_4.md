---
title: "[TIL] 250601 Programmers 외계행성의 나이"
date: 2025-06-01 10:00:00 +0900
categories: [til]
tags: [Programmers, 코딩테스트, 입문, 코테]
layout: single
---

Programmers 코딩 테스트 입문 문제 풀이입니다.

---

## 📌 문제 요약

- **문제 이름:** 외계행성의 나이
- **난이도:** 입문  
- **설명:**
우주여행을 하던 머쓱이는 엔진 고장으로 PROGRAMMERS-962 행성에 불시착하게 됐습니다. 입국심사에서 나이를 말해야 하는데, PROGRAMMERS-962 행성에서는 나이를 알파벳으로 말하고 있습니다. a는 0, b는 1, c는 2, ..., j는 9입니다. 예를 들어 23살은 cd, 51살은 fb로 표현합니다. 나이 age가 매개변수로 주어질 때 PROGRAMMER-962식 나이를 return하도록 solution 함수를 완성해주세요.

---

## 🧠 내 풀이 (C 언어)

```c
#include <stdio.h>
#include <stdbool.h>
#include <stdlib.h>

char* solution(int age) {
    // return 값은 malloc 등 동적 할당을 사용해주세요. 할당 길이는 상황에 맞게 변경해주세요.
    char* answer = (char*)malloc(sizeof(char)*4);

    sprintf(answer, "%d" , age);
     
    for(int i=0; i<strlen(answer); i++)
        answer[i] += 49;
    
    return answer;

}
```

## ✍️ 회고

> sprintf 함수를 잘 사용해보지 않아 배열에 한 자리씩 담는 작업을 위해 코드가 좀 길어지고 효율이 떨어졌었는데, 다른 풀이를 참고해보니 엄청 간단한 방법을 참고하게 되었다.

> a는 0, b는 1, c는 2라는 이야기가 나왔을 때 아스키 코드를 떠올려야하는 점
(a는 10진수로 나타냈을 때 97이고, 0은 48이다. 즉, 0에 49를 더해주면 a의 아스키코드 10진수가 나오게 됨)

> 내가 오래 걸렸던 자릿수별로 배열에 담아내는 작업은 sprintf 함수를 이용할 수 있었다. sprintf 함수를 이용하면 정수를 문자열로 바꿀 수 있다. sprintf('넣어줄 변수',"%d",'정수')와 같은 형태로 해주면 된다.

> 매일 사용하던 함수만 사용하고 하나하나 계산하려는 습관이 들다보니 효율적이지 못한 코드를 짜게 되는 것 같다. sprintf를 꼭 기억해야겠다.