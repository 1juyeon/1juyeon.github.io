---
title: "250610 Programmers 2차원으로 만들기"
date: 2025-06-10 21:00:00 +0900
categories: [til]
tags: [Programmers, 코딩테스트, 입문, C]
layout: single
---

Programmers 코딩 테스트 입문 문제 풀이입니다.

- [문제 링크](https://school.programmers.co.kr/learn/courses/30/lessons/120842)

## 문제 요약

1차원 정수 배열을 앞에서부터 `n`개씩 나눠 2차원 배열로 반환하는 문제다. 입력 배열의 길이는 `n`의 배수이므로 행의 수는 `num_list_len / n`이다.

## C 풀이

Programmers C 함수 시그니처에서는 2차원 배열을 `int**`로 반환하고, 각 행의 길이를 `answer_cols`에 기록해야 한다.

```c
#include <stdlib.h>

int** solution(int num_list[], size_t num_list_len, int n,
               size_t* answer_rows, size_t** answer_cols) {
    *answer_rows = num_list_len / n;

    int** answer = malloc(sizeof(int*) * (*answer_rows));
    *answer_cols = malloc(sizeof(size_t) * (*answer_rows));

    for (size_t row = 0; row < *answer_rows; row++) {
        answer[row] = malloc(sizeof(int) * n);
        (*answer_cols)[row] = n;

        for (int col = 0; col < n; col++) {
            answer[row][col] = num_list[row * n + col];
        }
    }

    return answer;
}
```

## 인덱스 계산

1차원 배열에서 `row`번째 묶음의 `col`번째 값은 다음 위치에 있다.

```text
원본 인덱스 = row * n + col
```

예를 들어 `n = 3`이고 `row = 2`, `col = 1`이면 원본 인덱스는 `2 * 3 + 1 = 7`이다.

## 회고

처음 게시한 내용에는 이전 문제인 사분면 판별 코드가 잘못 들어가 있었다. 제목과 본문이 일치하는지 다시 확인하면서 2차원 동적 배열의 반환 규칙도 함께 정리했다.

이 문제에서 중요한 것은 값 복사보다 메모리 구조였다.

- 행 포인터 배열을 먼저 할당한다.
- 각 행마다 `n`개의 `int` 공간을 할당한다.
- `answer_rows`와 각 `answer_cols`를 정확히 채운다.
- 호출자가 반환된 각 행과 행 포인터 배열을 해제한다.
