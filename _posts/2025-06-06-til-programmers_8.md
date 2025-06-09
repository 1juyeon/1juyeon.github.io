---
title: "[TIL] 250606 Programmers 모스부호(1)"
date: 2025-06-06 21:00:00 +0900
categories: [til]
tags: [Programmers, 코딩테스트, 입문, 코테]
layout: single
---

Programmers 코딩 테스트 입문 문제 풀이입니다.

---

## 📌 문제 요약

# [level 0] 모스부호 (1) - 120838 

[문제 링크](https://school.programmers.co.kr/learn/courses/30/lessons/120838) 

### 성능 요약

메모리: 4.44 MB, 시간: 0.01 ms

### 구분

코딩테스트 연습 > 코딩테스트 입문

### 채점결과

정확성: 100.0<br/>합계: 100.0 / 100.0

### 제출 일자

2025년 06월 06일 21:13:43

### 문제 설명

<p>머쓱이는 친구에게 모스부호를 이용한 편지를 받았습니다. 그냥은 읽을 수 없어 이를 해독하는 프로그램을 만들려고 합니다. 문자열 <code>letter</code>가 매개변수로 주어질 때, <code>letter</code>를 영어 소문자로 바꾼 문자열을 return 하도록 solution 함수를 완성해보세요.<br>
모스부호는 다음과 같습니다.</p>
<div class="highlight"><pre class="codehilite"><code>morse = { 
    '.-':'a','-...':'b','-.-.':'c','-..':'d','.':'e','..-.':'f',
    '--.':'g','....':'h','..':'i','.---':'j','-.-':'k','.-..':'l',
    '--':'m','-.':'n','---':'o','.--.':'p','--.-':'q','.-.':'r',
    '...':'s','-':'t','..-':'u','...-':'v','.--':'w','-..-':'x',
    '-.--':'y','--..':'z'
}
</code></pre></div>
<hr>

<h5>제한사항</h5>

<ul>
<li>1 ≤ <code>letter</code>의 길이 ≤ 1,000</li>
<li>return값은 소문자입니다.</li>
<li><code>letter</code>의 모스부호는 공백으로 나누어져 있습니다.</li>
<li><code>letter</code>에 공백은 연속으로 두 개 이상 존재하지 않습니다.</li>
<li>해독할 수 없는 편지는 주어지지 않습니다.</li>
<li>편지의 시작과 끝에는 공백이 없습니다.</li>
</ul>

<hr>

<h5>입출력 예</h5>
<table class="table">
        <thead><tr>
<th>letter</th>
<th>result</th>
</tr>
</thead>
        <tbody><tr>
<td>".... . .-.. .-.. ---"</td>
<td>"hello"</td>
</tr>
<tr>
<td>".--. -.-- - .... --- -."</td>
<td>"python"</td>
</tr>
</tbody>
      </table>
<hr>

<h5>입출력 예 설명</h5>

<p>입출력 예 #1</p>

<ul>
<li>.... = h</li>
<li>. = e</li>
<li>.-.. = l</li>
<li>.-.. = l</li>
<li>--- = o</li>
<li>따라서 "hello"를 return 합니다.</li>
</ul>

<p>입출력 예 #2</p>

<ul>
<li>.--. = p</li>
<li>-.-- = y</li>
<li>- = t</li>
<li>.... = h</li>
<li>--- = o</li>
<li>-. = n</li>
<li>따라서 "python"을 return 합니다.</li>
</ul>

<hr>

<ul>
<li>a ~ z에 해당하는 모스부호가 순서대로 담긴 배열입니다.</li>
<li><code>{".-","-...","-.-.","-..",".","..-.","--.","....","..",".---","-.-",".-..","--","-.","---",".--.","--.-",".-.","...","-","..-","...-",".--","-..-","-.--","--.."}</code></li>
</ul>


> 출처: 프로그래머스 코딩 테스트 연습, https://school.programmers.co.kr/learn/challenges
---


## * 다른 사람의 풀이

```c
#include <stdio.h>
#include <stdbool.h>
#include <stdlib.h>
#include <string.h>


// 파라미터로 주어지는 문자열은 const로 주어집니다. 변경하려면 문자열을 복사해서 사용하세요.
char* solution(const char* letter) {
 // return 값은 malloc 등 동적 할당을 사용해주세요. 할당 길이는 상황에 맞게 변경해주세요.

     char *morse[] = {
     ".-","-...","-.-.","-..",".","..-.",
     "--.","....","..",".---","-.-",".-..",
     "--","-.","---",".--.","--.-",".-.",
     "...","-","..-","...-",".--","-..-",
     "-.--","--.."};
    
     int idx=0;
     char* answer = (char*)malloc(1000);
     char* cut_morse = strtok(letter, " "); //공백일때 문자열 자르기
     memset(answer, 0, 1000);

     while (cut_morse != NULL)
     {
         for (int i = 0; i < 26; i++)
             {
                 if (strcmp(cut_morse, morse[i])==0)
                 {
                 answer[idx++] = 97 + i;
                 break;
                 }
         }
     cut_morse = strtok(NULL, " ");
     }

     return answer;
} return h/5 + (h%5)/3 + ((h%5)%3);
}
```

## ✍️ 회고

> 스스로 풀이하지 못해 다른 코드를 참고했다. strtok 라는 함수를 이용하는 방법과 포인터 배열을 선언해서 사용하는 방법에 대해 복습하면 좋을 것 같은 문제. 아스키코드의 대표적인 값들은 외우고 있어야 된다는걸 느꼈다.