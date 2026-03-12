---
layout: post
title: "분기문은 어떻게 쓰면 좋나"
date: 2026-03-05 12:20:00 +0900
categories: Code Optimization
show_on_home: false
---

# 분기문은 어떻게 쓰면 좋나

```c
void main()
{
		int a,b,c,d,x = 0;
		a = b = 3;
		c = 5;
		d = 6;
		
		if(((a*c)+b)/d == 0)
				x = ((a*c)+b)/d + 5;
		else if(((a*c)+b)/d == 1)
				x = ((a*c)+b)/d + 10;
		else if(((a*c)+b)/d == 2)
				x = ((a*c)+b)/d + 15;
		else if(((a*c)+b)/d == 3)
				x = ((a*c)+b)/d + 20;
		else if(((a*c)+b)/d == 4)
				x = ((a*c)+b)/d + 25;
		else
				x = 0;
		
		printf("%d\n", x);
}
```

```c
**ARM32, -O0 느낌 (최적화 없음)**
((a*c)+b)/d를 매번 다시 계산할 가능성이 큼
나눗셈은 보통 __aeabi_idiv 같은 런타임 함수 호출 (하드웨어 DIV 없으면)

main:
    push    {r4, r5, r6, r7, lr}
    sub     sp, sp, #24          ; 지역변수 공간 (a,b,c,d,x 등)

    mov     r0, #0
    str     r0, [sp, #20]        ; x = 0

    mov     r0, #3
    str     r0, [sp, #0]         ; a = 3
    str     r0, [sp, #4]         ; b = 3
    mov     r0, #5
    str     r0, [sp, #8]         ; c = 5
    mov     r0, #6
    str     r0, [sp, #12]        ; d = 6

    ; --- if ( ((a*c)+b)/d == 0 ) ---
L_if0_calc:
    ldr     r0, [sp, #0]         ; r0 = a
    ldr     r1, [sp, #8]         ; r1 = c
    mul     r0, r0, r1           ; r0 = a*c
    ldr     r1, [sp, #4]         ; r1 = b
    add     r0, r0, r1           ; r0 = (a*c)+b
    ldr     r1, [sp, #12]        ; r1 = d
    bl      __aeabi_idiv         ; r0 = r0 / r1   (t)
    cmp     r0, #0
    bne     L_elif1

    ; x = t + 5  (또 계산할 수도 있지만 여기선 t(r0) 재사용 가정)
    add     r0, r0, #5
    str     r0, [sp, #20]
    b       L_done

L_elif1:
    ; --- else if (t == 1) ---
    ; (대부분 -O0이면 여기서도 t를 다시 계산하는 코드 블록이 반복됨)
    ; ... t 계산 ...
    cmp     r0, #1
    bne     L_elif2
    add     r0, r0, #10
    str     r0, [sp, #20]
    b       L_done

L_elif2:
    ; ... t 계산 ...
    cmp     r0, #2
    bne     L_elif3
    add     r0, r0, #15
    str     r0, [sp, #20]
    b       L_done

L_elif3:
    ; ... t 계산 ...
    cmp     r0, #3
    bne     L_elif4
    add     r0, r0, #20
    str     r0, [sp, #20]
    b       L_done

L_elif4:
    ; ... t 계산 ...
    cmp     r0, #4
    bne     L_else
    add     r0, r0, #25
    str     r0, [sp, #20]
    b       L_done

L_else:
    mov     r0, #0
    str     r0, [sp, #20]

L_done:
    ldr     r1, [sp, #20]        ; printf 인자 x
    ldr     r0, =.LC0            ; "%d\n"
    bl      printf

    mov     r0, #0
    add     sp, sp, #24
    pop     {r4, r5, r6, r7, pc}

.LC0:
    .asciz "%d\n"

```

수식을 계산하는 부분이 비교문 수행할 때, 변수 x에 값 대입할 때마다 반복

이런 반복은 불필요하고 코드 크기 증가시키고, 속도도 떨어지게 함

아래처럼 값을 미리 계산해서 변수에 저장하고 필요할 때마다 수식 대신 변수 사용하는게 좋음

 

```c
void main()
{
		int a,b,c,d,x = 0;
		a = b = 3;
		c = 5;
		d = 6;
		t = ((a*c)+b)/d == 0
		
		if(t==0)
			x = t+5;
		else if(t==1)
			x = t+10;
		else if(t==2)
			x = t+15;
		else if(t==3)
			x = t+20;
		else if(t==4)
			x = t+25;
		else 
			x = 0;
}
```

```c
**ARM32, -O0 느낌 (최적화 없음)**
main:
    push    {fp, lr}
    add     fp, sp, #4
    sub     sp, sp, #24        ; 지역변수 공간 확보

    mov     r3, #0
    str     r3, [fp, #-20]     ; x = 0

    mov     r3, #3
    str     r3, [fp, #-4]      ; a = 3
    str     r3, [fp, #-8]      ; b = 3

    mov     r3, #5
    str     r3, [fp, #-12]     ; c = 5

    mov     r3, #6
    str     r3, [fp, #-16]     ; d = 6

    ; ----- t = ((a*c)+b)/d -----

    ldr     r2, [fp, #-4]      ; r2 = a
    ldr     r3, [fp, #-12]     ; r3 = c
    mul     r3, r2, r3         ; r3 = a*c

    ldr     r2, [fp, #-8]      ; r2 = b
    add     r3, r3, r2         ; r3 = (a*c)+b

    ldr     r2, [fp, #-16]     ; r2 = d
    mov     r0, r3
    mov     r1, r2
    bl      __aeabi_idiv       ; r0 = r0 / r1

    str     r0, [fp, #-24]     ; t 저장

    ; ----- if chain -----

    ldr     r3, [fp, #-24]
    cmp     r3, #0
    bne     L1
    ldr     r3, [fp, #-24]
    add     r3, r3, #5
    str     r3, [fp, #-20]
    b       L_end

L1:
    ldr     r3, [fp, #-24]
    cmp     r3, #1
    bne     L2
    ldr     r3, [fp, #-24]
    add     r3, r3, #10
    str     r3, [fp, #-20]

```

```c
-O0 특징 분석
1️⃣ 모든 지역변수 → 스택에 저장

a,b,c,d,t,x 전부 메모리에서 load/store

2️⃣ t를 매번 메모리에서 다시 읽음

ldr r3, [fp, #-24] 반복

3️⃣ 분기 많음 (branch chain)
4️⃣ 나눗셈은 함수 호출

bl __aeabi_idiv
```

수식 계산 같은 중복된 표현은 제거해버리면 코드 크기 줄고, 속도도 향상됨

책에는 if문은 비교랑 분기를 참일 때까지 하는데 switch는 한번 비교하고 지정된 레이블로 분기해서,

switch문이 낫다고 하지만…

> O0에서는 switch도 그냥 if-else 체인이랑 거의 똑같이 동작한다.
> 
>
> jump table은 최적화(-O2 이상)에서나 보통 생성된다.
> 

---

내 GPT가 말하길…

switch도 결국 컴파일러가 최적화 할 때 jump table을 만드는 경우에나, if문 보다 더 효율적이라고 한다. 

  

너가 생각하는 switch는 이런 구조지?

```
t → 바로 점프 테이블 인덱싱 → 해당case로 점프
```

이게 맞는 형태는 **jump table 최적화가 적용된 경우**야.

하지만 -O0에서는 컴파일러가:

> “디버깅 쉬운 코드”를 만드는 게 목적이지
> 
> 
> “빠른 코드”가 목적이 아님
> 

그래서 그냥 순차 비교로 만들어버림.

🔥 -O0 switch 실제 형태

switch(t) {

case 0:

case 1:

case 2:

case 3:

case 4:

이렇게 있으면 -O0에서는 거의 이렇게 나와:

```nasm
cmp r0, #0
beq L0
cmp r0, #1
beq L1
cmp r0, #2
beq L2
cmp r0, #3
beq L3
cmp r0, #4
beq L4
b   Ldefault
```

👉 완전히 if-else 체인과 동일 구조

🔥 그럼 언제 네 말처럼 되냐?

- O2 이상에서

그리고 조건이 만족해야 함:

1. case 값이 연속적이어야 함 (0,1,2,3,4)
2. 범위가 너무 크지 않아야 함
3. case 개수가 충분히 많아야 함

그때는 이렇게 바뀜:

```nasm
cmp r0, #4
bhi default
adr r1, jump_table
ldr pc, [r1, r0, lsl #2]
```

👉 O(1) 점프

이때는 switch가 훨씬 유리함.

🔥 근데 branch prediction도 변수임

if 체인이라도

```
if (t==3)
```

이 조건이 대부분 참이면

branch predictor가 잘 맞춰서

실제로는 큰 차이 안 날 수도 있음.

🔥 그래서 정리하면

| 상황 | 더 빠른 것 |
| --- | --- |
| -O0 | 거의 동일 |
| -O2 + 연속 case | switch (jump table) |
| case 적고 드물면 | 차이 거의 없음 |
| 값 분포 한쪽 치우침 | if가 더 예측 잘 될 수도 |

🔥 한 줄 요약

> switch가 항상 한 번 비교하고 끝나는 건 아니다.
> 
> 
> 그건 “최적화된 switch”일 때만 그렇다.
> 

지금 질문 수준 보니까

이거 궁금하지 않냐 😎

👉 jump table이 실제 메모리에서 어떻게 생겼는지

👉 왜 sparse case(0,100,1000)는 jump table 안 쓰는지

여기까지 가면 분기 최적화 거의 끝판왕이다.

근데 무조건 switch가 jump table 되냐?

아니.

컴파일러는 이런 걸 봄:

1️⃣ case 값이 연속적인가?

```
0,1,2,3,4 → 좋음
```

2️⃣ 범위가 너무 크진 않은가?

```
0,1000,100000 → jumptable 만들면 메모리 낭비
```

3️⃣ case 개수가 충분히 많은가?

2~3개면 그냥 비교가 더 나음.

🔥 오히려 if가 더 유리한 경우도 있음

예:

```c
if (t <3)
```

이건 switch로 표현 못 함.

또는

값 분포가 편향되어 있다면

branch predictor가 잘 맞추는 if가 더 빠를 수도 있음.

---

## 이진 분기 사용

값이 균등하게 분포한다면, 이진 분기가 평균적으로 더 효율적이다. 

하지만 현대 CPU에서는 branch prediction이 더 중요 할 때가 많다. 

```c
if (a==1){
} else if(a==2){
} else if (a==3){
} else if (a==4){
} else if (a==5){
} else if (a==6){
} else if (a==7){
} else if (a==8){
}
```

이건 최악의 경우 8번 비교, 평균 약 4~5번 비교를 하게 됨 O(n) 구조

만일 이렇게 이진 분기 한다면?

```c
if (a <= 4) {
    if (a <= 2) {
        if (a == 1)
        else if (a == 2)
    } else {
        if (a == 3)
        else if (a == 4)
    }
} else {
    if (a <= 6) {
        if (a == 5)
        else if (a == 6)
    } else {
        if (a == 7)
        else if (a == 8)
    }
}

```

최대 3~4번 비교 **O(log N)** 구조임

현실 CPU에서는 branch prediction이 중요한데

1. 값이 특정 값에 몰려 있으면?
    
    a==1이 90%일 확률 ⇒ 선형 구조가 오히려 빠름 (첫번째 분기에서 바로 끝남)
    
2. 값이 랜덤하게 균등 분포라면?
    
    그 때는 이진 분기가 평균적으로 유리
    

이 경우는 사실 switch가 제일 좋음

```c
switch(a) {
  case 1:
  case 2:
  ...
  case 8:
}

```

-O2 이상이면 보통 jump table로 변환됨:

```c
cmp r0, #8
bhi default
adr r1, jump_table
ldr pc, [r1, r0, lsl #2]

```

비교 1번 + 인덱스 점프

<정리 하자면>

| 상황 | 가장 좋은 방식 |
| --- | --- |
| 값 균등, case 많음 | switch (jump table) |
| 값 균등, if만 써야 함 | 이진 분기 |
| 특정 값이 자주 발생 | 선형 if (자주 나오는 걸 위에) |
| -O0 | 차이 거의 없음 |

---

### branch prediction?? (내 GPT가 정리)

🔥 한 줄 정의

> **Branch Prediction = CPU가 “분기 결과를 미리 예측”하는 기능**
> 

🔹 왜 필요하냐?

CPU는 **파이프라인**으로 동작해.

예:

```
1. 명령어 읽기2. 해석3. 실행4. 메모리 접근5. 결과 저장
```

이걸 겹쳐서 동시에 처리함.

🔥 문제는 “if문”

```c
if (a ==0)
```

CPU 입장에서:

> “다음에 실행할 명령어가 어디냐?”
> 
> 
> - 참이면 A로 점프
> - 거짓이면 B로 진행
> 

근데 비교 결과가 아직 안 나왔어 😬

🔹 그래서 CPU가 하는 짓

“음… 아마 이쪽일 거야”

→ 일단 그쪽 명령어를 미리 가져와서 실행 준비

이게 **Branch Prediction**이야.

🔥 예시로 이해해보자

상황 1️⃣ 버튼 체크

```c
if (button_pressed)
```

버튼은 거의 안 눌림.

CPU는 이렇게 학습함:

> 대부분 false네 → 계속 false로 예측
> 

그래서 거의 100% 맞춤.

상황 2️⃣ 랜덤 값

```c
if (rand() %2)
```

참/거짓 50:50

CPU가 예측하기 어려움 → 틀릴 확률 높음.

🔥 예측이 틀리면?

이게 핵심이다.

예측이 틀리면:

1. 이미 실행 준비한 명령어들 전부 버림
2. 파이프라인 초기화
3. 다시 가져오기

👉 10~20 사이클 날아감 (CPU마다 다름)

이걸 **pipeline flush**라고 해.

🔥 그래서 중요한 거

| 경우 | 성능 |
| --- | --- |
| 예측 잘 맞음 | 빠름 |
| 예측 자주 틀림 | 느림 |

🔥 그래서 이런 코드가 빠름

```c
if (likely_condition)
```

리눅스 커널에 이런 매크로 있음:

```c
#define likely(x)   __builtin_expect((x),1)
```

CPU에게 힌트 줌:

> “이 조건은 대부분 참이야”
> 

🔥 너 코드에 적용해보면

```c
if(a==1)elseif(a==2)
...
```

만약 a가 대부분 1이라면?

👉 첫 분기만 비교하고 끝

👉 branch prediction 거의 완벽

👉 매우 빠름

근데 a가 1~8 랜덤이면?

👉 예측 실패 많음

👉 이진 분기나 switch가 더 유리

🔥 한 문장으로 끝내자

> Branch Prediction은
> 
> 
> CPU가 “분기 방향을 미리 맞추려고 하는 똑똑한 추측 시스템”이다.
> 

---

컴파일러 옵션은 물론 넣어야겠지만

만일 -O0일지 -O2일지 모르는 상황에서 

switch문을 사용할 때 컴파일러를 믿을지, 내가 구조를 강제할 건지? 이걸 고민 하게 된다면

값이 아래 예제같이 

0 ~ 113 처럼 연속적이고 범위가 명확하면

```c
int f1(int a)
{
		switch(a) {
				case 0 : return 0x3f;
				case 1 : return 0x12;
				case 2 : return 0x31;
				...
				case 113 : return 0x61;
				default : return 0x00
		}
}

void main(void)
{
		f1(3);
}
```

아래와 같이 테이블 방식(b[a])이 구조적으로 명확하고 안정적이다

(범위 체크는 반드시 넣어야 함)

```c
int f1(int a)
{
		int b[] = {0x3f, 0x12, 0x31...., 0x61};
		return b[a];
}

void main(void)
{
		f1(3);
}
```

-O2옵션을 쓰면 어차피 컴파일러는 jump table을만들 가능성이 높아

```c
return table[a];
```

결과는 동일해짐

-O0 옵션을 쓰는 경우

switch는  

```c
cmp a, #0
beq L0
cmp a, #1
beq L1
...
cmp a, #113
beq L113
```

이렇게 어셈이 되고, 최악은 114번의 비교가 발생 됨

근데 만약 테이블 방식을 강제로 작성했다면?

```c
static const int b[114] = {
   0x3f, 0x12, 0x31, ...
};

int f1(int a)
{
    if (a < 0 || a >= 114)
        return 0;
    return b[a];
}
```

-O0 옵션이어도 어셈은 거의

```c
cmp r0, #113
bhi default
ldr r1, =b
ldr r0, [r1, r0, lsl #2]
bx lr
```

비교 1번 + 인덱스 접근 해서 O(n)의 복잡도를 갖게 됨

<aside>
💡

근데 Case 값이 연속이 아닐 때는 

테이블을 쓰면 괜히 공간 낭비니까

 메모리가 극도로 제한된 MCU에서는 메모리가 부담되면 switch가 나을수 있음

</aside>

  

아래 두 경우는 성능 차이 별로 없으니 걍 의미에 맞게 쓰는게 좋음

```c
char c;
c = getch();

switch(c)
{
		case 'A':
		{
				break;
		}
		case 'B':
		{
				break;
		}
		case 'C':
		{
				break;
		}
}
```

```c
char c;
c = getch();

switch(c)
{
		case '0':
		{
				break;
		}
		case '1':
		{
				break;
		}
		case '2':
		{
				break;
		}
}
```

---

내 GPT가 추가로 알려준 내용

char가 signed냐 unsigned냐에 따라 switch 동작 달라질 수 있냐?

결론부터 말하면:

> ✅ **switch 동작 자체는 같지만, 비교 결과가 달라질 수 있다.**
> 
> 
> 즉, **case에 도달하는 값이 달라질 수 있음.**
> 

🔥 핵심 원인

`char`는 C에서 **signed인지 unsigned인지 구현체에 따라 다름.**

- 어떤 컴파일러는 `char` = signed char
- 어떤 컴파일러는 `char` = unsigned char

ARM GCC는 보통 **signed char** 기본.

🔹 예시 1️⃣ (문제 안 되는 경우)

```c
char c ='A';switch(c) {case'A': ...
}
```

'A' = 65

65는 양수라서

signed/unsigned 상관없음.

👉 동일하게 동작.

🔹 예시 2️⃣ (문제 발생 가능)

```c
char c =0xFF;switch(c) {case0xFF:
      ...
}
```

여기서 갈림.

🔥 만약 char가 signed라면

0xFF = 255 (unsigned 해석)

하지만 signed char에서 0xFF는:

```
0xFF → -1
```

그래서 내부 비교는:

```
switch(-1)case255
```

👉 절대 매칭 안 됨.

🔥 만약 char가 unsigned라면

0xFF = 255 그대로

```
switch(255)case255
```

👉 정상 매칭

🔥 실제 어셈에서 뭐가 다르냐?

signed char면 컴파일러가 이런 코드 넣음:

```nasm
ldrsb r0, [addr]   ; sign extend
```

unsigned면:

```nasm
ldrb r0, [addr]    ; zero extend
```

이 차이 때문에 값이 달라짐.

🔥 그래서 임베디드에서 중요한 습관

절대 이렇게 쓰지 마:

```c
char c;
```

이렇게 써:

```c
int8_t  c;// 명확히 signeduint8_t c;// 명확히 unsigned
```

특히 UART, getch(), 키보드 입력 같은 건

보통 `uint8_t`가 안전함.

🔥 한 줄 요약

| 상황 | 결과 |
| --- | --- |
| ASCII 문자 (0~127) | 차이 없음 |
| 128~255 값 | signed면 음수로 변함 |
| switch 비교 | 매칭 달라질 수 있음 |

---
