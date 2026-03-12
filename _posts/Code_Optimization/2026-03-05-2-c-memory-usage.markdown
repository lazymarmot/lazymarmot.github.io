---
layout: post
title: "C의 메모리 사용"
date: 2026-03-05 12:00:00 +0900
categories: Code Optimization
show_on_home: false
---

# C의 메모리 사용

[ELF 세그먼트 구조](https://www.notion.so/ELF-2fb6feb16a3e8079b7d6e690c90a31ac?pvs=21) 

위 페이지 내용을 참고하게 되면 C 프로그램이 메모리 레이아웃을 어떻게 사용하는지 알 수 있다. 

| .text  | 코드, 문자열과 상수 |
| --- | --- |
| .data | 초기화되는 전역 변수 / 정적(static) 변수 |
| .bss | 초기화되지 않은 전역 변수 /정적(static) 변수 |
| stack | 지역 변수 / 함수 인자 |
| register  | 레지스터 변수 |

C에서 메모리를 지정하는 지정자들은 아래와 같이 나뉜다

| 기억 부류 지정자 | extern, static, auto, register | 데이터가 저장될 메모리 영역 결정
결정된 메모리의 영역에 따라 데이터 초기화, 유효범위, 유효 시간등 달라짐 |
| --- | --- | --- |
| 타입 지정자 | void, char, short, int, long, float, double, signed, unsigned | 데이터 크기 결정지음 |
| 타입 한정자 | const, volatile | 메모리의 성질 변화를 일으킴 |

### **※ static**

static으로 선언된 변수는 데이터 세그먼트에 저장된다.

프로그램이 실행될 때 생성되고, 프로그램이 수명을 다할 때까지 유지된다.

static 전역 변수는 외부 파일에서는 보이지 않기 때문에 외부에서 선언된 같은 이름의 전역 변수와 충돌하지 않는다. 그래서 전역 변수가 외부의 영향을 받지 않게 하고 싶을 때 사용하면 편리하다. (다른 파일에서 선언된 전역 변수와 이름이 같아도 문제되지 않는다는 의미)

```c
int a;   
int b = 20;
static int c = 30;
static int t;

void t(int a)
{
    static int d = 10;
    char e = 0x1;
}

```c
int a;          // 전역, 초기화 X
static int t;   // 정적(static), 초기화 X
```

섹션 : .bss

프로그램 시작부터 끝까지 존재

```c
int b = 20;
static int c = 30;
```

섹션 : .data

static 여부는 링커 심볼 가시성 차이만 있음, 메모리 수명은 동일

```c
static int d = 10;
```

섹션 : .data (전역/정적 변수 영역)

프로그램 시작 시 한번 초기화

함수가 끝나도 값 유지

```c
char e = 0x1;
```

섹션 : stack

함수 들어올 때 생성, 함수 나가면 사라짐

<c, d 공통 / 차이점>

| 구분 | `static int c = 30;` | `static int d = 10;` |
| --- | --- | --- |
| 선언 위치 | 파일 전역 | 함수 내부 |
| 메모리 영역 | `.data` | `.data` |
| 수명 | 프로그램 시작 ~ 종료 | 프로그램 시작 ~ 종료 |
| 같은 파일 다른 함수 접근 | 가능 | 불가 |
| 같은 함수 접근 | 가능 | 가능 |
| 외부 파일 접근(extern 접근) | 불가 | 불가 |
| 용도 | 파일 전용 전역 상태 | 함수 전용 내부 상태 |

< 함수 내부 static 변수 특징>

| 구분 | 전역 변수 | 함수 내부 static  |
| --- | --- | --- |
| 메모리 위치 | .data / .bss | **.data / .bss** |
| 생성 시점 | 프로그램 시작 | **프로그램 시작** |
| 소멸 시점 | 프로그램 종료 | **프로그램 종료** |
| 접근 범위 | 전체 파일 / 외부 | **함수 내부만** |
| 이름 충돌 | 가능 | **불가능** |

<aside>
💡

함수 내부 static 변수

- 메모리(.data/.bss)에 실제로 존재하지만,
- 그 함수 안에서만 이름으로 접근 가능
- 같은 파일 다른 함수에서도 직접 접근 불가

함수 하나를 만들고 그 함수 안에서만 변수를 설정할 수 있도록 하기 위해 사용(static 변수)

주로 상태 제어나 count 설정, device driver에서는 initialized 체크를 위해서 사용

</aside>

```c
<초기화 한 번만 해야 할 때>
void driver_init(void)
{
    static int initialized = 0;

    if (initialized)
        return;

    hw_setup();
    initialized = 1;
}
```

```c
<카운터/누적값>
int read_error(void)
{
    static int err_cnt = 0;

    if (hw_error())
        err_cnt++;

    return err_cnt;
}

```

### ※ extern

전역 변수는 함수 외부에서 선언되고 변수 중 유효 범위가 가장 넓은 변수로써 프로그램 전체에서 사용 할 수 있다. 

보통 다른 모듈의 전역 변수를 끌어다 쓰려고 할 때 사용한다.

외부에 있는 전역 변수를 현재 모듈에서도 사용할 수 있도록 컴파일러랑 링커에게 알려줌

- 컴파일러는 이 외부 변수가 어떤 데이터 타입인지 알지만, 어디 있는지는 모름
- 링커에 의해서 링킹 할 때 오브젝트 파일 합쳐지면서 연결
    
    ```c
    <ext_test.c>
    #include <stdio.h>
    
    extern int k;
    void main(void)
    {
    		k = 5000;
    }
    ```
    
    ```c
    <ext.c>
    
    int k;
    ```
    

### ※ auto

자동 변수, 지역 변수라고 하며 스택에 저장된다. 

지역 변수가 선언된 블록이 실행될 때 동적으로 메모리 할당 받고, 블록이 끝나면 같이 생명을 다한다. 

유효 범위에 가장 제약을 받는 변수

```c
#include <stdio.h>

void func4(void);
int a;

void main(void)
{
		int a = 10;
		{
				int a = 500, b = 5;
				printf("main의 블록 내에서 출력 : a(%d), b(%d)\n", a,b);
		}
		func4();
		printf("main의 블록 외부에서 출력 : a(%d)\n", a);
}

void func4(void)
{
		int c = 100;
		printf("func4에서 출력 : a(%d) c(%d)\n", a,c);
}
```

```c
출력 결과
main의 블록 내에서 출력 : a(500), b(5)
func4에서 출력 : a(0) c(100)
main의 블록 외부에서 출력 : a(10)
```

- 블록 안 `a` → **500** (가장 안쪽 스코프)
- `func4()`의 `a` → **전역 변수 a = 0** (초기화 안 된 전역 → 0)
- 블록 밖 `a` → **main의 지역 변수 a = 10**