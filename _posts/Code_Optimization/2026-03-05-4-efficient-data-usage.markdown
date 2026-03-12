---
layout: post
title: "데이터를 효율적으로 사용"
date: 2026-03-05 12:00:00 +0900
categories: Code Optimization
show_on_home: false
---

# 데이터를 효율적으로 사용

## ※ 배열 인덱스 활용

배열의 인덱스를 활용하면 분기를 제거할 수 있다.

```c
<예 1>
switch(choose) 
{
		case 1:
				text = 'a';
				break;
		case 2:
				text = 'b';
				break;
		case 3:
				text = 'c';
				break;
}
```

```c
<예 2>
static char *buf = "abc";
text = buf[choose];
```

1은 switch문을 사용해 분기하고 있지만,

2는 choose 값을 바로 배열 인덱스로 활용해서 분기, 비교가 일어나지 않고, 코드 크기도 줄었다.

이 방법은 조건에 따라 다른 함수를 호출하거나 다른 값을 대입 할 때 유용하게 사용된다. 

switch가 항상 느린건 아닌데, 컴파일러가 코드를 어떻게 바꾸느냐에 따라 다르다.

**case 1 : 값이 연속 적일 때**

```c
switch (choose) {
case 1: A();
case 2: B();
case 3: C();
}
```

이런 경우 컴파일러는 아래 처럼 바꾼다

```c
점프 주소 배열 = {A, B, C}
-> x 값으로 바로 점프
비교 여러번 안하고 한번에 점프
```

이런 경우 컴파일러는 보통 jump table로 바꿔버린다. 

```nasm
cmp r0, #3        ; (r0 (==choose값)이 3보다 큰지 비교)
bhi default       ; bhi : branch if higher (r0 > 3 이면 default로 점프) 배열 범위 넘어가면 안전하게 처리
ldr r1, =jump_table  ; r1에 jump_table의 주소를 넣어라
; jump_table은 이런 메모리임
;  jump_table:
;        .word case 0
;        .word case 1
;        .word case 2
;        .word case 3
ldr pc, [r1, r0, LSL #2] ; => jump_table[r0] 주소 읽어서 그 주소로 점프해라
; pc = *(r1 + r0 * 4)
; r0 = case 번호
; LSL #2 = 왼쪽 shift 2 -> *4
; 4는 주소 하나 크기 (32bit)
; pc에 값 넣으면 그 주소로 점프함
```

분기 1번 + 테이블 점프

⇒ 이 경우는 배열 인덱스랑 성능 거의 비슷

**case 2 : 값이 듬성듬성일 때**

```c
case 1:
case 100:
case 5000:
```

이러면  jump table을 못 씀

컴파일러가 아래와 같이 바꿈

 

```nasm
cmp r0, #1
beq case1
cmp r0, #100
beq case100
cmp r0, #5000
beq case5000
```

이러면 분기가 많아지고, 케이스 수가 많으면 느려질 수 있음 

2번의 배열 인덱스 방식은

```c
static char buf[] = {'a','b','c'};
text = buf[choose];
```

```nasm
ldr r1, =buf
ldrb r2, [r1, r0]
```

분기가 없고 단순 주소 계산 + load를 해서 빠르다.

즉,

switch가 jump table을 사용하면 배열 인덱스를 쓰나, switch를 쓰나 성능은 거의 동일하다.

branch predictor가 잘 맞으면 분기도 거의 공짜임

진짜 성능 차이가 나는 경우는

- case 개수 많고
- 값이 불규칙하고
- 자주 실행되는 hot path

이럴 때는 배열 방식이 더 일정하고 빠르다.

또한,

배열 방식은 choose 범위 검사를 안 한다 → 잘못된 값이면 메모리 침범 위험

switch는 범위 검사 후 범위를 벗어나면 default로 간다 → 안전

함수 호출 예제

```c
switch(choose)
{
		case 1:
				f1();
				break;
		case 2:
				f2();
				break;
		case 3:
				f3();
				break;
}
```

```c
void (*p[3])() = {f1, f2, f3};
p[choose]();
```

## ※ 구조체 패딩비트 줄여라

구조체 패딩 비트?

> CPU가 메모리를 빠르게 접근하려고 구조체 안에  “빈 공간”을 자동으로 넣는 것, 이 빈 공간을 padding이라고 한다.
> 

패딩 비트보다는 “패딩 바이트”가 정확한 표현 ⇒ 보통  byte단위로 정렬

  

왜 패딩 비트를 만들까?

CPU는정렬(alignment)를 좋아한다. 

32bit ARM이면, 

데이터 버스 폭 = 32bit (4byte)

한 번에 4byte 읽는게 가장 효율적임 

⇒ CPU는 기본적으로 주소가 4의 배수인 위치에서 4byte 읽는걸 좋아함

```c
0x1000  
0x1004  
0x1008  
```

1. 정렬이 맞는 경우
    
    ```c
    int x;  // 4byte
    ```
    
    주소가 0x1000에 있다면
    
    ```c
    0x1000 ~ 0x1003
    ```
    
    CPU는 4byte를 한번에 읽을 수 있음
    
    ```c
    lrd r0, [0x1000]  ; 1번 메모리 접근
    ```
    
2. 정렬이 안 맞으면?
    
    int가 0x1001에 있다고 가정하자..
    
    ```c
    0x1001 ~ 0x1004
    ```
    
    이 주소는 4byte 경계에 걸쳐있음
    
    보통 메모리 배치는 아래와 같이 생겼음
    
    ```c
    0x1000 0x1001 0x1002 0x1003
    0x1004 0x1005 0x1006 0x1007
    ```
    
    0x1001에서부터 4byte를 읽으려면 
    
    - 0x1000 블록에서 일부
    - 0x1004 블록에서 일부
    
    ⇒ 2번을 읽어야 한다..
    

**그래서 CPU 내부에서는 정렬이 안 맞으면**

- **첫 번째  aligned 블록 읽기**
- **두 번째 aligned 블록 읽기**
- **shift**
- **OR 연산으로 합침**

**이런 추가적인 작업들이 발생해서 느려지는 것임** 

1. 정렬을 좋아하는 진짜 이유
    1. 버스 폭 때문
        
        CPU는 4byte/8byte 단위로 움직인다.
        
    2. 캐시 라인 구조 때문
        
        캐시는 보통 32byte/64byte 단위로, 정렬이 맞으면 한번에 캐시 라인에 깔끔하게 들어감
        
    3. 내부 데이터 경로 때문
        
        ALU, 레지스터 폭이 32bit/64bit 임 
        

결국, 

정렬이 안 맞으면 접근이 느려지고, 어떤 MCU는 fault발생도 한다.

그래서

⇒ 컴파일러가 자동으로 빈칸을 추가한다. 

```c
struct A {
    char a;   // 1 byte
    int  b;   // 4 byte
};
```

이걸 메모리에 배치하면?

```c
offset 0: a (1byte)
offset 1: padding
offset 2: padding
offset 3: padding
offset 4~7: b (4byte)
```

총 크기 = 8byte

- int는 4byte 정렬이 필요
- a 다음에 바로 못둠
- 4의 배수 위치(4번)으로 밀림

순서 바꾸면?

```c
struct A {
    int  b;
    char a;
};
```

```c
0~3: b
4:   a
5~7: padding
```

여전히 8byte

구조체 전체도 가장 큰 멤버 정렬 단위에 맞춰서 정렬된다. 

패딩 비트 줄이기

```c
before

struct Test1
{
		char a;
		int b;
		char c;
		int d;
};
```

```c
after 

struct Test2
{
		char a;
		char c;
		int b;
		int d;
};
```

![image.png](/assets/posts_image/Code_Optimization/efficient_data_usage/image.png)

구조체 Test1의 멤버 a는 1byte 크기이지만, 

int 형의 크기는 4byte 이므로 b는 주소가 104인 메모리에 저장된다.

이 때 건너 띈 메모리 주소 101 ~ 103은 0으로 채워지는데 이를 패딩 비트라고 한다.

```smalltalk
a (1)
패딩 3
b (4)
c (1)
패딩 3
d (4)

=> 총 16byte
```

⇒ 이런 패딩 비트를 줄이면 메모리를 절약할 수 있다.

구조체 Test2는 멤버 b와 c의 순서를 교체했다.

char 는 1byte이기 때문에 어느 주소에나 저장 할 수 있다.

그래서 멤버 a,c가 한 워드에 저장되고, 나머지 두 개의 int 멤버가 저장되어 총 크기가 12byte이다. 

```smalltalk
a (1)
c (1)
패딩 2   ← int 맞추려고
b (4)
d (4)

=> 총 12byte
```

하지만 Test1의 크기는 모든 멤버가 한 워드 씩 차지하여 16byte이다. 

이처럼 구조체의 멤버 순서만 변경해도 패딩 비트를 줄일 수 있다.   

그럼 어떤 타입 순서에 따라 변수 만들어야 할까?

큰 타입 → 작은 타입 순서가 일반적인 최적화 전략이다

```smalltalk
int
int
char
char
```

**큰 타입 부터 내림차순 정렬이 기본 전략임**

CPU는 보통 int (4byte)를 4의 배수 주소(0x100, 0x104. 0x108 …)에 두려고 한다

그래서 구조체 안에 

```c
char (1)
int  (4)
```

이렇게 오면

```smalltalk
[char][패딩 3바이트][int]
```

이렇게 강제로 맞춰버린다.

추가로 중요한 것

마지막에도 패딩이 붙을 수 있다.

구조체 전체 크기는가장 큰 멤버의 정렬 단위 배수로 맞춰진다

```c
struct {
    int a;
    char b;
}
```

5byte처럼 보이지만 실제는 8byte일 수 있다. (왜냐면 다음 구조체 배열에서 정렬 맞추기 위함)

구조체 많이 쓰는 통신 패킷, DMA버퍼, 대량의 배열 이런 데서는 멤버 순서 정렬이 정말 중요한데, 

```c
__attribute__((packed))
```

이렇게 강제로 패딩 제거하면 CPU가 misaligned access해서 느려질 수 있기 때문에 무조건 packed가 답은 아니다.