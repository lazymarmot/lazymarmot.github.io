---
layout: post
title: "변수 타입 사용"
date: 2026-03-05 12:00:00 +0900
categories: Code Optimization
show_on_home: false
---

## ※ 적절한 데이터 타입 선택

### 네이티브 데이터 타입

CPU가 가장 효율적으로 처리하는 타입 

보통, word 크기와 같은 정수형이 네이티브 타입이다. 

32bit CPU → int = 32bit

64bit CPU → long 또는 long long = 64bit (아키텍처에 따라 다름)

⇒ 레지스터 크기랑 같아서 쪼개지지 않고 한번에 처리 됨 (빠름)

```smalltalk
💡

word : CPU가 한 번의 load/store 명령으로 자연스럽게 처리하는 기본 단위

CPU 레지스터 크기 = word 크기

- 32bit CPU → 레지스터 32bit → word = 32bit (4byte)
- 64bit CPU → 레지스터 64bit → word = 64bit (8byte)
```
<br>
```smalltalk
💡

Bus Width

버스 폭은 CPU ↔ 메모리 사이의 도로 폭 이라고 보면된다.

- 32비트 데이터 버스 → 한 번에 32bit 전송
- 64비트 데이터 버스 → 한 번에 64bit 전송
```

만일 32bit 컴퓨터에서 사용되는 버스 폭이 32bit이면 레지스터나 버스 폭 크기에 맞는 데이터 타입을 사용하면 그만큼 연산도 줄어들기 때문에 성능이 좋아진다.

***예시***

| 예시 1 | 예시 2 |
| --- | --- |
| CPU: 32비트<br>데이터 버스: 64비트<br><br>이 경우:<br>CPU는 32bit 연산 가능<br>메모리에서는 64bit씩 읽어올 수 있음<br>내부에서 나눠서 사용<br><br>int a,b;<br>연속된 메모리에 있다면<br>메모리에서 한번에 64비트를 읽을 수 있어서 a(32bit) + b(64bit)를 한번에 가져올 수 있음<br><br>CPU는 먼저 a 사용 → 다음에 b 사용<br>(이미 캐시나 버퍼 안에 있어서 추가 메모리 접근이 없음)<br><br>⇒ 메모리 접근 횟수 줄고, 캐시 효율 좋아서 성능 빠름 | CPU: 64비트<br>버스: 32비트<br><br>이 경우:<br>CPU는 64bit 연산 가능<br>버스가 32bit라 두번 전송해야 함<br><br>⇒ 메모리에서 32bit씩 두 번 읽어야 함<br>⇒ 성능 저하 발생 |

<br>

| --- |
| 32bit CPU, 데이터 버스폭 32bit (4byte)<br><br>double c; (8byte)<br>변수 c를 메모리에서 읽어올 때 두 번의 메모리 접근 필요<br>char d; (1byte)<br><br><strong>CPU가 byte load 지원 시 (ARM, x86)</strong><br>`LDRB r0, [addr]`<br>1byte만 읽음<br>버스가 4byte 전송해도 CPU는 내부적으로 1byte만 사용해서 마스킹 추가 비용 거의 없음<br>⇒ 효율성 크게 떨어지지 않음<br><br><strong>CPU가 byte load 지원 안 함</strong><br>`LDR r0, [addr]`<br>`AND r0, #0xFF`<br>버스에서 4byte 전송하면 CPU는 읽어온 4byte 중 1byte 데이터를 추려내는 마스킹 연산 추가 → 효율성 떨어짐<br>하지만 현대 CPU에선 거의 해당 안 됨. |

성능은 마스킹보다 “메모리 접근 횟수”가 훨씬 더 중요하다.

그래서 임베디드에서 진짜 조심해야 할 건

- 큰 데이터 타입
- misaligned 접근
- 캐시 미스

<br>
### 변수 데이터 타입의 사용 규칙

가능하면 네이티브 데이터 타입 쓰기 

변수의 범위 안에 포함되는 가장 작은 데이터 타입을 쓰기

부동 소수점을 피해라 → 되도록 정수 사용, 소수점이 필요한 경우에는 고정 소수점을 사용하기

부동 소수점은 최악의 경우 정수 연산에 비해 수백 배까지 성능을 떨어트릴 수 있다 

고속 부동 소수점 연산을 지원하는 FPU(Floating Point Unit)를 사용하기도 하는데 프로세서에 따라서 이 유닛의 지원 여부가 달라진다. 

만일 FPU가 지원되지 않는 프로세서라면고정소수점을 사용하는 것이 좋고, 되도록 float, double과 같은 부동소수형을사용하지 않도록 한다. 

<br>

## ※ 전역 변수 최적화

■ 전역 변수 + 초기 값 있는 경우

```c
int a = 10;
```

빌드 결과(ELF기준)

- a는 .data 섹션에 들어간다.
- 초기 값 10은 ROM(Flash)에 저장되고, 실행 시 RAM에 공간이 따로 할당된다.

<br>

■ 부팅 순간에 무슨 일 벌어지나?

스타트 업 코드에서 아래 같은 작업을 한다.

```c
ROM(.data 초기값 영역)  →  RAM(.data 실행 영역) 복사
```

Flash 안에 있는 10이 RAM에 복사되며, CPU는 RAM에 있는 a를 사용하게 된다.

<br>

■ 실행 중에 값이 바뀌면?

```c
a = 20;
```

RAM에 있는 a 값만 바뀐다.

ROM은 절대 안바뀐다. (ROM은 읽기 전용)

***정리***

```c
[Flash / ROM]
  a 초기값 = 10  ← 여기 저장되어 있음

[RAM]
  a 실행 영역  ← 부팅 시 10이 복사됨
  이후 20, 30 등으로 변경됨
```

<br>

---

레지스터에 캐싱한다?

RAM에 있는 값을 CPU 내부 레지스터에 복사해두고 이후에는 RAM을 다시 안 읽고, 

그 레지스터 값을 계속 쓰는 것

<br>

CPU는 

- 연산 → 레지스터에서만 가능

- RMA은 느리고, 레지스터는 빠름 (CPU안에 있어서)

    그래서 RAM → 레지스터로 한번 복사해서, 그 다음부터는 레지스터 끼리 연산하는게 제일 빠름

<br>

지역 변수는 레지스터에 배정될 가능성이 높다 (컴파일러가 결정하는 부분) 

```c
int g_cnt = 100;

void foo(void)
{
    int local = g_cnt;
    for (int i = 0; i < local; i++)
    {
        ;
    }
}
```

위 코드를 컴파일러에서 컴파일 할 때

```nasm
ldr r0, =g_cnt
ldr r1, [r0]     ; g_cnt 값을 r1에 복사 (RAM → 레지스터)

mov r2, #0       ; i = 0

loop:
cmp r2, r1       ; r1(local)과 비교 (레지스터끼리 비교)
bge done
add r2, r2, #1
b loop
```

r1 = local

r2 = i

local은 RAM에 따로 저장 안 할 수도 있음

r1 레지스터가 local 역할을 하게 되는 것이다. 

```c
int local;
```

C에서 local은 RAM에 반드시 공간을 만들겠다가 아니라, 

컴파일러가 

- 레지스터에 둘지

- 스택에 둘지

- 최적화로 없앨지 결정한다.

<br>

전역 변수는 레지스터에 할당할 수 없으므로 함수나 루프에서 사용하지 않는 것이 좋다.

- 전역 변수는

    주소가 고정되어 있음

    다른 파일에서도 접근 가능

    ISR이 바꿀 수 도 있음

- 그래서 컴파일러가 함부로 판단하지 못한다.

    “이 값은 절대 안바뀌네?” → “레지스터에 계속 둬야 겠다” (이런 판단을 못함)

<br>

컴파일러 옵션이 -O2 이상이면 컴파일러가 똑똑해져서 전역 변수도 레지스터에 캐싱 해 줄 수 있음

⇒ 항상 전역변수를 지역변수로 복사해야 하는 건 아님

<br>

 volatile 전역변수 사용, ISR 루틴에서 전역 변수 사용 , 멀티코어 공유 변수의 경우는 또 다르게 봐야 함

<br>

***코드 예시 확인***

1. 전역 변수 그대로 Loop에서 쓰는 경우 (매번 메모리 load)
    
    ```c
    int g_cnt = 1000;
    
    void foo(void) {
        for (int i = 0; i < g_cnt; i++) {
            work();
        }
    }
    ```
    
    어셈블리 패턴 (최적화 약함/보수적으로 잡을 때)
    
    ```nasm
        movs    r4, #0          ; i = 0
    loop:
        ldr     r3, =g_cnt      ; g_cnt의 주소
        ldr     r2, [r3]        ; ★ 매 반복마다 RAM에서 g_cnt 읽음
        cmp     r4, r2
        bge     done
    
        bl      work            ; work()
    
        adds    r4, r4, #1
        b       loop
    done:
        bx      lr
    ```
    
    포인트: `ldr r2, [r3]` 가 루프 안에 있음 → **루프마다 RAM 읽음**
    
2. 전역 변수를 지역변수로 한 번만 받아서 쓰는 경우
    
    전역변수를 레지스터 캐싱이 가능한 지역변수로 복사해서 레지스터 연산할 수 있도록 함
    
    ```c
    void foo(void)
    {
        int local_cnt = g_cnt;   // 한 번만 RAM에서 읽음
    
        for (int i = 0; i < local_cnt; i++)
        {
            do_something();
        }
    }
    ```
    
    어셈블리 패턴 
    
    ```nasm
        ldr     r3, =g_cnt
        ldr     r5, [r3]        ; ★ 딱 한 번만 RAM에서 읽어서 r5에 저장(local_cnt)
    
        movs    r4, #0          ; i=0
    loop:
        cmp     r4, r5          ; ★ 비교는 레지스터끼리
        bge     done
    
        bl      work
    
        adds    r4, r4, #1
        b       loop
    done:
        bx      lr
    ```
    
    g_cnt를 한번 RAM에서 읽어와서 레지스터에 올려두고, 
    
    루프를 도는 동안 계속 레지스터 값을 사용하게 한다. 
    
    ⇒ RAM load가 루프 밖에서 1번 → 루프는 레지스터 연산만 함 (빠름)
    

1. 전역 변수인데 레지스터 캐싱되는 경우
    
    ```c
    int g_cnt = 100;
    
    void foo(void)
    {
        for (int i = 0; i < g_cnt; i++)
        {
            do_nothing();
        }
    }
    ```
    
    만약 do_nothing이 
    
    inline되거나
    
    g_cnt를 수정하지 않는다고 확실하게 분석되면
    
    -O2 이상의 옵션에서 아래와 같이 레지스터에 캐싱되기도 함
    
    ```nasm
    ldr r1, =g_cnt
    ldr r2, [r1]   ; 한번만 읽음
    ...
    cmp r3, r2     ; 계속 r2 사용
    
    ```
    

1. volatile 전역 변수 (무조건 매번 메모리 읽어야 함) → **최적화 하지마!**
    
    ```c
    volatile int g_flag;
    
    void foo(void) {
        while (g_flag == 0) {
            // wait
        }
    }
    ```
    
    어셈블리 패턴
    
    ```nasm
    loop:
        ldr     r3, =g_flag
        ldr     r2, [r3]        ; ★ volatile이라 매번 RAM 읽기 강제
        cmp     r2, #0
        beq     loop
        bx      lr
    ```
    
    volatile은 “레지스터에 캐싱하지마!” 가 규칙이라서, 
    
    지역변수로 복사해도 의미가 달라질 수 있음
    

1. ISR이 전역 변수 건드리는 경우
    
    ```c
    int g_flag;
    
    void ISR(void)
    {
        g_flag = 1;
    }
    
    void main_loop(void)
    {
        while (g_flag == 0)
            ;
    }
    ```
    
    컴파일러가 ISR을 분석하지 못하면
    
    - ISR은 하드웨어가 비동기적으로 호출 → 컴파일러는 “언제 실행되는지” 알 수 없음
    
        ```c
        int flag = 0;
    
        void main_loop(void)
        {
            while (flag == 0)
                ;
        }
        ```
    
    - 컴파일러 입장 :
    
        이 함수 안에서 flag가 안 바뀜, 다른 함수도 호출 안됨, volatile도 아님
    
        ⇒ 그럼 flag는 항상 0이네? (결론 내림)
    
        그래서 컴파일러가 최적화 해버림
    
        ```c
        ldr r0, =flag
        ldr r1, [r0]     ; 한번 읽음
        cmp r1, #0
        bne done
        b .              ; 무한루프
        ```
        flag를 다시 안 읽음
    
    근데 ISR 함수가 존재한다면?
    
    ```c
    int flag = 0;
    
    void ISR(void)
    {
        flag = 1;
    }
    
    void main_loop(void)
    {
        while (flag == 0)
            ;
    }
    ```
    
    하드웨어가 인터럽트가 발생시키면 → ISR이 실행되고, flag가 1이 됨
    
    근데 컴파일러는 :
    
    - ISR이 언제 실행될지 몰라 → while문 안에서 ISR이 개입 할 지 추론을 못하게 됨
    
    - 특히 ISR이 다른 파일에 있거나, 링커 단계에서 연결 되거나, 벡터 테이블에 의해 연결되면
    
        → main_loop와 ISR의 관계를 직접적으로 알지 못함
    
    **결과적으로**
    
    - **컴파일러는 :**
    
        **“flag가 안 바뀌네?” 라고 가정하고 레지스터에 캐싱해버림**
    
    - **그럼**
    
        **flag를 RAM에서 한번만 읽고 이후엔 레지스터 값만 보게 됨**
    
    - **그러면**
    
        **ISR에서 flag=1로 바꾸게 되도, main_loop는 RAM을 안읽고 레지스터 값만 봐서 → 무한루프**
    
    그래서 이런 경우 반드시 volatile 붙여주는게 좋음
    
    ```c
    volatile int g_flag;
    ```
    
     이 변수는 최적화 하지 말고 매번 메모리에서 읽어라!
    
    ```nasm
    loop:
    ldr r0, =flag
    ldr r1, [r0]    ; 매번 RAM에서 읽음
    cmp r1, #0
    beq loop
    ```
    
    이제 ISR이 flag=1하면 다음 루프에서 RAM이 다시 읽어서 빠져나감
    

< 정리 >

| 상황 | 레지스터 캐싱 가능? |
| --- | --- |
| 단순 읽기 + 함수가 안 건드림 | 가능 |
| 함수 호출 있고 분석 불가 | 보통 안 함 |
| volatile | 절대 안 함 |
| ISR 공유 변수 | volatile 없으면 위험 |
| 멀티코어 공유 | barrier 필요 |

<br>

## ※ 지역 변수 최적화

지역 변수는 스택이나 레지스터에 저장된다. 

함수 인자는 레지스터에 저장되기도 하며, 보통 스택에 저장된다. 

ARM 프로세서

- ARM Calling Convention (AAPCS)

- r0 ~ r3 → 함수 인자용 레지스터

    : 컴파일러에서 

    레지스터 남으면 인자 저장으로 사용하고, 부족하면 스택 사용한다. 

<br>

ARM 32bit CPU

- 레지스터 = 32bit 

- ALU 연산 단위 = 32bit

- ⇒ CPU는 기본적으로 32bit 단위로 연산

    ```c
    char a = 5;
    char b = 3;
    char c = a + b;
    ```

    CPU는 8bit 연산

    - 8bit 값을 32bit로 확장 (sign/zero extend)

    - 32bit 연산 수행

    - 다시 8bit로 줄임 (masking)

    ```nasm
    ldrb r0, [a]     ; 8bit load → 자동 zero extend
    ldrb r1, [b]
    add  r2, r0, r1  ; 32bit add
    strb r2, [c]     ; 하위 8bit만 저장
    ```

    대부분의 경우 int가 기본 연산 단위로 효율적이긴 한데, char/short 써도 요즘엔 컴파일러가 잘 최적화 한다.  (특히 -O2 이상이면 의미 없음)

<br>

< 지역변수 최적화를 위해 알아둬야 할 것들 >

1. 연산용 변수는 기본적으로 int 사용
    
    CPU 레지스터 = 32bit
    
    ALU 기본 연산 단위 = 32bit
    
    확장/마스킹 최소화
    
    ```c
    for (int i = 0; i < n; i++)
    ```
    
     루프 카운터는 그냥 int가 제일 자연스러움
    

1. 루프 안에서 메모리 접근 최소화
    
    ```c
    이런 코드는 안됨
    
    for (int i = 0; i < g_cnt; i++)
    ```
    
    ```c
    int cnt = g_cnt;
    for (int i = 0; i < cnt; i++)
    ```
    
    <br>

1. 구조체 멤버 반복 접근 피하기
    
    ```c
    for (...) {
        sum += obj->value;
    }
    ```
    
    위 코드는 아래처럼 바꾸는게 나음
    
    ```c
    int v = obj->value;
    for (...) {
        sum += v;
    }
    ```
    
    포인터 역참조는 메모리 접근임
    
    <br>

2. 함수 안에서 쓰는 작은 함수는 inline 사용
    
    ```smalltalk
    💡
    
    inline
    
    “이 함수를 호출하지 말고 함수 내용을 호출한 자리로 그냥 복붙해라”
    
    라는 컴파일러를 위한 힌트임
    
    ```
    
    함수 호출은 :
    
    - 스택 사용, 레지스터 저장/복구를 하기 때문에
    
    <br>

    작은 함수는 inline 사용해서 호출 오버헤드 제거
    
    ```c
    static inline int add(int a, int b)
    {
        return a + b;
    }
    
    void foo()
    {
        int x = add(1, 2);
    }
    ```
    
    컴파일러가 이렇게 바꿈
    
    ```c
    void foo()
    {
        int x = 1 + 2;
    }
    ```
    
    ```nasm
    mov r0, #3
    ```
    
    함수 호출 없음, 점프 없음, 스택 사용 없음
    
    ```smalltalk
    💡
    
    작은 함수
    
    getter
    
    단순 계산
    
    짧은 조건 검사
    
    ```
    <br>
    
    ```c
    for (int i = 0; i < 1000; i++)
    {
        sum += add(i, 1);
    }
    ```
    
    add가 inline 안 되면 → 1000번 함수 호출
    
    inline 되면 → 그냥 덧셈 1000번
    
    **⇒ 요즘에는 inline 안 써도 컴파일러가 알아서 작은 함수는 inline 함**
    
    코드 크기 커질 수 있으니까 자중해서 사용해야 함
    
3. 큰 지역 배열은 스택에 두지 말기
    
    ```c
    이렇게 쓰지 말기
    void foo() {
        int buf[1000];   **// 스택 터질 수 있음**
    }
    
    ```
    
    큰 배열은 보통  static/global, heap으로 생성
    
    <br>

1. volatile 남발하지 말기
    
    volatile 붙이면 :
    
    - 레지스터 캐싱이 안되고, 매번 메모리에 접근해서 성능 확 떨어짐
    
        **⇒ ISR 공유 변수, HW 레지스터에서만 사용하면 될 듯**
    

1. 포인터 aliasing 주의
    
    ```smalltalk
    💡
    
    alias
    
    “ 같은 대상을 가리킴”
    
    서로 다른 포인터가 같은 메모리 주소를 가리키는 상황
    
    ```
    
    alias 예시
    
    ```c
    int x = 10;
    
    int *a = &x;
    int *b = &x;
    ```
    
    a, b는 서로 다른 포인터 변수지만 같은 메모리 (x)를 가리킴 → 이게 aliasing
    
    ```c
    void foo(int *a, int *b)
    {
        *a = 10;
        *b = 20;
    }
    ```
    
    컴파일러 입장 :
    
    - a, b가 같은 주소일 수도 있음
    
    실제 호출이 이렇게 되면
    
    ```c
    int x;
    foo(&x, &x);
    ```
    
    실행 순서에 따라 결과 달라짐
    
    그래서 컴파일러에서 최적화가 제한됨
    
    C99 이상이면
    
    ```c
    void foo(int *restrict a, int *restrict b)
    ```
    
    alias가 아닌 것을 restrict로 알려줌
    
<br>

## ※ 타입 한정자 const

const는 일반 변수와 같지만, 초기 값 이외의 값으로 변경 될 수 없다.

- 전역 상수로 정의 된다면, 데이터 세그먼트에 저장

- 지역 상수로 정의되면, 스택에 저장

const는 데이터 타입을 갖는 상수로 이해 할 수 있으며, 한 가지 유의할 점은 배열의 크기 값으로 사용할 수 없다.  

***< const 키워드는 위치에 따라 역할이 조금씩 달라짐 >***

const는 앞의 단어를 수식

```c
1. const int *p_a;
2. int const *p_b;
3. int * const p_c;
4. int const * const p_d;
```

1번 : const가 맨 앞에 있을 경우는 기본 데이터 타입 여기서는 int를수식해서 값을 변경하지 못함  

2번 : const 앞에 int가 있으므로 값을 변경할 수 없음

3번 : const 앞에 *(포인터)가 있으므로 주소 값을 변경할 수 없음

4번 : const 두 번, 값도 변경하지 말고 주소도 변경하지 말라는의미