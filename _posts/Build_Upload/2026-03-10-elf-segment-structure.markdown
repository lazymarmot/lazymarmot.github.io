---
layout: post
title: "ELF 세그먼트 구조"
date: 2026-03-10 11:00:00 +0900
categories: Build Upload
show_on_home: false
---

# ELF 세그먼트 구조

보드에 올리는 실행파일의 내부 세그먼트 구조는 어떻게 되고, 어떤 메모리에서 어떻게 동작하는걸까??

임베디드 환경에서는 flash 메모리에 elf 또는 elf에서 뽑아낸 bin이 그대로 들어간다. (Flash = ROM)

대부분의 프로그램은 실행 시 RAM으로 로드되서 실행되는데 이때 코드의 성질에 따라서 로드는 세그먼트가 달라지게 된다. 

![image.png](/assets/posts_image/Build_Upload/ELF_segment_structure/image.png)

elf파일은 통째로 ROM에 저장되고, 실행 시작 할 때 startup 코드에서 필요한 섹션 만 RAM으로 복사한다. 

### ROM에 이미지 로드

컴파일 + 링크 과정이 끝나고 만일 elf 포맷으로 실행 파일을 만든다면, 아래와 같은 정보들이 들어있다

| ELF 섹션 | 의미 |
| --- | --- |
| `.text` | 코드 |
| `.rodata` | 문자열, 상수 |
| `.data` | 초기값 있는 전역/정적 변수 |
| `.bss` | 초기값 없는 전역/정적 변수 |
| **Program Header** | “이걸 어디에 로드해라” 정보 |

⇒ ROM에 text/data/bss가 따로 저장되는게 아니라 elf 하나의 파일 형태로 저장

```c
ROM
	.text      ← 코드
	.rodata    ← const
	.data(load)← 초기값만 있음
  .bss 없음

RAM
	.data      ← ROM에서 복사됨
	.bss       ← startup 코드가 0으로 초기화
```

ROM용량 절약 + 부팅 속도 최적화를 위한 설계임임

- `.data`
    
    → “초기값이 중요한 변수”
    
    → 값을 들고 와야 함
    
- `.bss`
    
    → “어차피 전부 0”
    
    → 굳이 ROM에 저장할 필요 없음
    

### 프로그램 시작

전원 on → 프로그램 시작

startup code(_start)에서 초기화 진행

링커 스크립트 + startup 코드 조합으로 ROM → RAM으로 세그먼트 복사 

1. .text, .rodata
    1. ROM(flash)에 그대로 남아있음
    2. ROM에서 실행, RAM으로 옮기지 않음 (XIP)
        
        ex) printf 코드 → ROM, “string” → ROM
        
2. .data (초기값이 있는 전역/정적 변수)
    
    ROM의 데이터 세그먼트를 RAM으로 복사
    
    “ROM에 저장되는 건 변수가 아니라 초기 값이고, 실제 값이 변하는 변수 본체는 처음부터 끝까지 RAM에 만 있다”
    
    ```c
    int a = 10;
    
    ROM → 초기 값 10
    RAM → 실행 중 실제 변수 a
    ```
    
    위 코드 내용 설명 : 
    
    변수는 실행 중 값이 바뀌는 저장 공간이라, ROM의 초기 값은 절대 바꾸면 안되니까 프로그램 시작할 때 RAM으로 한 번 복사하고 그 이후엔 RAM에서  값을 바꾸며 실행한다. 
    
    ```c
    [ROM]
    a_init = 10   ← 초기값
    
            ↓ (프로그램 시작 시 복사)
    
    [RAM]
    a = 10  → 20 → 30 → 99
    ```
    
    startup 코드에서 하는일 :
    
    ```c
    copy ROM(.data 초기값) → RAM(.data 영역)
    ```
    
3. .bss (초기 값 없는 전역/정적 변수)
    
    ```c
    int b;
    
    초기값이 없음
    ```
    
    초기 값이 없으니 ROM에 저장할게 없음(ROM공간을 낭비할 이유도 없음)
    
    ROM에는 .bss 자체가 없고,
    
    RAM에 .bss를 위한 공간만 잡아두고 startup 코드가 RAM에서 0으로 초기화 함
    
    ```c
    memset(_sbss, 0, _ebss - _sbss);
    
    memset(bss_start, 0, bss_size);
    ```
