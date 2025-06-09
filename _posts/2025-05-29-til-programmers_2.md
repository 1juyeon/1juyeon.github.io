---
title: "250529 Programmers 양꼬치"
date: 2025-05-25 10:00:00 +0900
categories: [til]
tags: [Programmers, 코딩테스트, 입문, 코테]
layout: single
---

Programmers 코딩 테스트 입문 문제 풀이입니다.

---

## 📌 문제 요약

- **문제 이름:** 양꼬치  
- **난이도:** 입문  
- **설명:**
머쓱이네 양꼬치 가게는 10인분을 먹으면 음료수 하나를 서비스로 줍니다. 양꼬치는 1인분에 12,000원, 음료수는 2,000원입니다. 정수 n과 k가 매개변수로 주어졌을 때, 양꼬치 n인분과 음료수 k개를 먹었다면 총얼마를 지불해야 하는지 return 하도록 solution 함수를 완성해보세요.

---

## 🧠 내 풀이 (C 언어)

```c
#include <stdio.h>
#include <stdbool.h>
#include <stdlib.h>

int solution(int n, int k) {
    int answer = 0;
    
    int cnt = n/10;
    
    if(n%10==0||((n>10)&&cnt>1)){
        answer = (n*12000)+(k-cnt) * 2000;
    } else{
        answer =  (n*12000)+(k*2000);
    }
       
    return answer;
}
```

### 🔎 설명
- 10, 20, 30 이거나 11 이상을 굳이 나눠서 조건을 구하고 나머지 일의자리 숫자는 그냥 계산하는 방식으로 구현했다

---

## ✅다른 사람의 코드

```c

#include <stdio.h>
#include <stdbool.h>
#include <stdlib.h>

int solution(int n, int k) {
    int answer = 0;

    answer = (n*12000) + ((k-(n/10))*2000);

    return answer;
}
```


## ✍️ 회고

> 다른 사람의 풀이 방식이 더 효율적이고 간결하며 가독성도 높음.
앞으로는 문제 조건을 수식으로 단순화할 수 있는지 먼저 고민하고, 불필요한 조건 분기는 줄이려고 노력해야겠다
