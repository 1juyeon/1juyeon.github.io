---
title: "250610 Programmers 2차원"
date: 2025-06-10 21:00:00 +0900
categories: [til]
tags: [Programmers, 코딩테스트, 입문, 코테]
layout: single
---

Programmers 코딩 테스트 입문 문제 풀이입니다.

---

## 📌 문제 요약


---

## * 내 풀이 (C 언어)

```c
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
    int ans[2][2] = \{\{3,2\}\, \{\4,1\}\};
    return ans[dot[0] > 0][dot[1] > 0];
}
```


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
