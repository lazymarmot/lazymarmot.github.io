---
layout: post
title: "루프는 어떻게 쓰면 좋을까"
date: 2026-03-05 12:10:00 +0900
categories: Code Optimization
show_on_home: false
---

```c
에제 1)

int i, j = 10;
for(i=0; i<20; i++){
		function(i);
}
```
<br>

```c
예제 2)

int i, j = 10;
for(i=0; i<20; i+=4){
		function(j);
		function(j);
		function(j);
		function(j);		
}
```

예제 1보다 예제 2의 실행 시간이 좀 더 줄어든다. 

i가 20보다 작은지 검사하는 연산, i를 증가 시키는 연산을 20번 수행해야 하고, 반복할 때마다 분기가 일어난다.

분기는 이미 준비된 파이프라인을 파괴하고 다시 생성해야 하는 문제를 발생 시키므로 속도의 효과 면에서 좋은 방법이 아니다. 

 <br>

### ※ 루프 풀기 (loop unrolling)

```c
에제 1)

int i, a[100];
for(i=0; i<100; i++)
{
		a[i] = i + 1;
}

```
<br>

```c
예제 2)

int i, a[100];
for(i=0; i<100; i+=4)
{
		a[i] = i + 1;
		a[i+1] = i + 2;
		a[i+2] = i + 3;
		a[i+3] = i + 4;		
}
```

루프 언롤링은 루프를 펼치는 것을 말하는데, 성능 면에서 빨라질 수 있다. 

예제1을 보면 특정 문구를 100회 반복하는데 “비교 연산 101회”, “반복을 위한 계산 100회”를 수행하기 때문에 반복을 위한 추가 연산과 관련된 비용이 반복 횟수가 늘어날수록 커진다.

예제2 처럼 루프 횟수를 좀 줄이면 반복에 대한 비용이 1/4로 줄어든다. 

<br>

왜 좀 빨라질까?

1. loop 제어 오버헤드 제거
    
    원래는 매 반복마다 
    
    ```c
    i 증가
    비교
    분기
    ```
    
    위 단계를 실행해야 하는데 unrolling하면 반복 횟수가 줄어든다.
    
2. ILP (Instruction Level Parallelism) 증가
    
    CPU가 동시에 여러 연산을 실행할 수 있음
    
    ```c
    a[i] = i + 1;
    a[i+1] = i + 2;
    a[i+2] = i + 3;
    a[i+3] = i + 4;		
    ```
    
    이렇게 펼쳐져 있으면 파이프라인 활용이 더 잘됨
    
3. branch prediction 부담 감소
    
    loop 는 매번 branch가 있음
    
    ```c
    cmp
    bne loop
    ```
    
    unrolling 하면 branch 횟수 줄어 듬
    

보통 루프 언롤링은 

- 수행 횟수가 소수가 아니고
    
    (소수는 1과 자기 자신 이외 약수를 가질 수 없는 수라서 반복을 나누어 줄이는 작업이 어려움)
    
- 반복 제어에 사용되는 변수가 한번만 초기화 되는

로직에서 가장 효과적이다.

<br>

그렇다고 루프 언롤링이 항상 좋은 건 아니다. 

1. 코드 크기 증가 (루프 블록 크기 커짐)
    
    펼치면 코드가 커짐 → “I-cache(Instruction cache)“miss 증가 가능
    
    메모리 낭비가 생겨서 적정선을 좀 찾아야 한다. 
    
    메모리가 넉넉하고, 속도가 중요한 시스템이면 하드 코딩하는 횟수 늘리고, 반복 횟수를 줄이는게 좋지만, 
    
    메모리가 넉넉하면 반복을 하는게 좋을 듯? 하다.
    
2. 현대 CPU는 이미 똑똑함
    
    branch prediction 잘함
    
    out-of-order 실행
    
    자동 vectorization 
    
    자동 unrolling(-O3)
    

```smalltalk
💡

Cortex-M 과 Cortex-A의 차이

Cortex-M (MCU)

- branch prediction 거의 없음
- 파이프라인 단순
- loop unrolling이 효과 있음

Cortex-A (리눅스, 고성능)

- branch prediction 있음
- superscalar
- out-of-order
- prefetch 있음
- 효과 줄어들고 오히려 i-cache 압박이 문제임
```

| 상황 | 효과 |
| --- | --- |
| 아주 작은 반복 (4~8회) | 거의 의미 없음 |
| 아주 긴 반복 + 단순 연산 | 도움 될 수 있음 |
| 메모리 병목 | 효과 적음 |
| MCU | 의미 있음 |
| 이미 -O3 | 컴파일러가 알아서 함 |

**루프 언롤링 하면 루프 블록이 커지고 명령어 캐시(I-cache) 사용 못 한다는게 뭔 말일까?**

- 명령어 캐시(I-cache)란?

    CPU는 프로그램 코드를 메모리에서 바로 매번 가져오지 않음 (느리니까)

    대신 작은 명령어 캐시라는 공간에 “자주 쓰는 코드”를 미리 담아두고 거기서 읽음

    ```smalltalk
    💡

    I-cache 크기 = 32KB

    한 번에 64바이트 단위로 가져옴(cache line)

    ```

    작은 루프 일 때

    ```c
    for (i=0; i<1000; i++)
        sum += a[i];
    ```
<br>

    ```nasm
    L:
    load
    add
    inc
    cmp
    branch
    ```

    코드 크기가 아주 작아서(20~30바이트) 이 작은 코드가 통째로 캐시에 올라간다.

    → 1000번 반복해도 캐시 안에서 계속 돎 (그래서 빠름)
    
    <br>

    루프 언롤링 하면

    4배 언롤링

    ```nasm
    for (i=0; i<1000; i+=4) {
        sum += a[i];
        sum += a[i+1];
        sum += a[i+2];
        sum += a[i+3];
    }
    ```

    이러면 어셈 블록이 4배 커짐

    이제 루프 본문이 예를들어 120바이트가 됐다고 치자

    만일 I-cache line이 64바이트라면

    - 첫 64바이트

    - 다음 64바이트

    두 줄을 계속 오가게 된다.

    그리고 만일 다른 코드도 같이 실행 중이면 캐시에서 밀려 날 수도 있다.

즉, 

- 코드가 커지고
- I-cache miss 증가 가능 → miss 나면 메모리에서 다시 가져와야 하고
- 수십 사이클이 손해임

⇒ 고성능 CPU(cortex-A, x86)에서 더 민감해서 과도한 언롤링은 오히려 느려지게 함

<br>

### ※ 루프 병합 (loop fusion)

루프 병합은 보통 아래와 같이 예1 에서 루프가 따로 구현 되어 있을 때 이걸 예2 처럼 합치는게 낫다는 말이다.

```c
예 1) 

int a[10], i, sum = 0;
for(i=0; i<10; i++)
	a[i] = i + 1;
for(i=0; i<10; i++)
	sum += a[i];
```
<br>

```c
예 2)

int a[10], i, sum = 0;
for(i=0; i<10; i++){
	a[i] = i + 1;
	sum += a[i];
}
```

<br>

### ※ 루프랑 상관 없는 계산은 루프 밖에서 해라

 루프 밖에서 계산 가능 한 것들은 밖에서 계산하고 루프를 타는게 성능에 좋다.

```c
예 1)

for(i=0; i<30; i++){
		sum[i] = group1->classA->person[i]->kor + 
						group1->classA->person[i]->math +
						group1->classA->person[i]->eng;
}
```
<br>

```c
예 예 2)

struct student *tmp = group1->classA;
for(i=0; i<30; i++){
		sum[i] = tmp->person[i]->kor + 
						tmp->person[i]->math +
						tmp->person[i]->eng;
}
```

중첩된 구조체로 이루어져 있는 코드에서

멤버에 접근하려면 몇 개의 포인터 변수를 이용해야 한다.  

근데, 멤버에 접근하는 변수들은 매번 같게 사용되므로 루프 밖에서 미리 계산하여 루프 안에서 사용하면 반복 계산을 피할 수 있다.  

<br>

위 코드를 보며 포인터 계산  “group1→classA”가 반복 수행 되는데, 이걸 예 2에서 포인터 계산을 루프 밖으로 빼내서 반복 계산을 피했다.   

```c
예 1)

void function1(int *p)
{
		int i;
		for(i=0; i<10; i++)
			subfunction(*p, i);
}
```
<br>

```c
예 2)

void function1(int *p)
{
		int i;
		int buf = *p;
		for(i=0; i<10; i++)
			subfunction(buf, i);
}
```

예1의 경우 *p의 값은 반복 내내 값이 변하지 않지만, 매번 메모리에 접근해서 값을 가져온다. 

이런 동작은 불필요하기 때문에 예2처럼 메모리에 한 번만 접근해서 읽어온 데이터를 buf에 저장하고 이걸 루프 안에서 사용한다. 

<br>

### ※ 필요 없는 루프를 끝까지 돌리지 마라

```c
flag = FALSE;
search = 55;

for(i=0; i<10000; i++){
		if(a[i] == search)
				flag = TRUE;
				break;        //요렇게 루프가 완료됐으면 break걸어서 나가야 한다. 
}

if(flag)
		printf("found\n");
else
		printf("not found\n");
```

<br>

### ※ 함수 루프

루프에서 함수를 호출할 때가 많은데 이렇게 되면 반복적인 분기가 이중으로 발생해서 성능에 좋지가 않다. 

그래서 거꾸로 함수 안에 루프를 넣어 사용하면 루프 안의 함수를 반복적으로 호출하는 비용을 줄일 수 있다. 

```c
for(i=0; i<1000; i+){
		function1(x, i);
}

void function1(int m)
{
		// do something		
}
```

```c
function1(x);

void function1(int m)
{
		int i;
		for(i=0; i<1000; i++){
				// do something
		}
}
```

