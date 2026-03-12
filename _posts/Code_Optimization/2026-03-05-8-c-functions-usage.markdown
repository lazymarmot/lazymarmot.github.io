---
layout: post
title: "함수는 어떻게 사용할까?"
date: 2026-03-05 12:30:00 +0900
categories: Code Optimization
show_on_home: false
---

# 함수는 어떻게 사용할까?

함수의 호출 깊이를 깊게 하면 성능에 그닥 좋은 영향을 주진 않음

근데 또 너무 모듈화해서 자잘하게 함수에서 다른 함수 호출하고 그러면 또 비효율적인 코드가 됨

어떻게 함수의 오버헤드를 줄이고 인자와 같은 함수 요소를 적합한 방법으로 다룰까

### **※ 매크로 함수**

매크로 함수는 보통 번거로운 "비트 연산"과 같은 작업들을 매크로 함수로 만들어서 사용   

특히 레지스터 제어할 때 주로 사용

```c
**특정 비트 SET

#define SET_BIT(reg, bit)   ((reg) |= (1U << (bit)))

SET_BIT(GPIO_CTRL, 3);**
```

```c
특정 비트 CLEAR

#define CLEAR_BIT(reg, bit) ((reg) &= ~(1U << (bit)))

CLEAR_BIT(GPIO_CTRL, 3);
```

```c
특정 비트 READ

#define READ_BIT(reg, bit)  (((reg) >> (bit)) & 0x1)

if (READ_BIT(GPIO_STATUS, 2))
{
    // 버튼 눌림
}
```

```c
특정 필드 값 넣기 (비트 필드 설정)

#define SET_FIELD(reg, mask, pos, val) \
    ((reg) = ((reg) & ~(mask)) | ((val) << (pos)))

#define MODE_MASK  (0xF << 8)

SET_FIELD(GPIO_CTRL, MODE_MASK, 8, 5);

8~11번 비트에 5 설정
```

매크로 함수는 개발중인 시스템 메모리에 여유가 있고 서비스 속도가 중요한 경우 자주 사용하면 유용할 것 같다. 

### **※ 인라인 함수**

 인라인은 함수를 호출하는 대신 함수 코드를 호출 위치에 삽입하는 방법

높은 레벨의 최적화에서는 작은 크기의 함수를 자동 인라인 처리하기도 한다. 

인라인은 분기를 하지 않기 때문에 속도를 높일 수 있지만, 코드를 직접 삽입하기 때문에 프로그램 크기가 커진다. 

inline은 

> "이 함수 호출하지 말고 코드 위치에 그냥 펼쳐 넣어라" 라고 컴파일러에게 요청하는 것
> 

```c
비트 SET

static inline void set_bit(volatile unsigned int *reg, unsigned int bit)
{
    *reg |= (1U << bit);
}

set_bit(&GPIO_CTRL, 3);

함수는 값 복사라서 
void set_bit(unsigned int reg, ...)
이렇게 하면 원본이 안바뀜, 그래서 주소를 넘겨야 됨
```

```c
static inline void clear_bit(volatile unsigned int *reg, unsigned int bit)
{
    *reg &= ~(1U << bit);
}
```

```c
비트 필드 설정

static inline void set_field(volatile unsigned int *reg,
                             unsigned int mask,
                             unsigned int pos,
                             unsigned int val)
{
    *reg = (*reg & ~mask) | (val << pos);
}

```

<매크로와 inline의 차이>

| 항목 | 매크로 | inline |
| --- | --- | --- |
| 타입 체크 | 없음 | 있음 |
| 디버깅 | 힘듦 | 쉬움 |
| 부작용 위험 | 있음 | 없음 |
| 안전성 | 낮음 | 높음 |

매크로보다 static inline을 더 선호함

### **※  함수에서 인자를 잘 활용하는 방법?**

함수의 리턴 값은 보통 레지스터에 저장되지만, 메모리의 특정 위치에 저장되기도 한다.

- int → r0 (또는 x0)
- 64bit 값 → r0+r1 (32bit ARM)
- 큰 구조체 → 호출자가 넘긴 메모리 주소에 저장

 

**32비트 ARM (AArch32, ARMv7 기준)**

표준 ABI인 **AAPCS (ARM Procedure Call Standard)** 기준으로

- **r0 ~ r3** → 첫 4개 인자
- 5번째부터 → 스택
- **리턴값 → r0**

```c
int add(int a, int b, int c, int d, int e);

배치
a → r0
b → r1
c → r2
d → r3
e → stack

리턴값
r0
```

**64비트 ARM (AArch64)**
64비트는 다름

AArch64 ABI 기준 

- **x0 ~ x7** → 첫 8개 인자
- 리턴값 → x0

즉 요즘 ARM64에서는 **8개까지 레지스터로 전달**

함수 호출에서 스택 접근은 느리고 레지스터가 훨씬 빠르다.

인터럽트 핸들러 만들때

```nasm
push {r0-r3}
```

이런 코드가 있는 이유가 함수 인자 레지스터라 보존해야 되기 때문에 스택에 push하는것

인자를 더 쓴다고 성능이 급락하거나 하는 건 아닌데

인자가 많으면 함수 책임이 많고 API가 복잡해지니까 구조체로 묶는게 코드 관리에 유리하다.

```c
void config(int mode, int speed, int size, int type, int enable);
```

이거보단

```c
typedef struct {
    int mode;
    int speed;
    int size;
    int type;
    int enable;
} config_t;

void config(config_t *cfg);
```

이게 유지 보수에 좋음

현실적인 결론

| 상황 | 전략 |
| --- | --- |
| 일반 코드 | 인자 개수 크게 신경 안 써도 됨 |
| 5개 이상 | 구조체로 묶는 게 좋음 |
| 극한 최적화 | 레지스터 개수 고려 |

함수 인자의 타입은

레지스터 타입에 맞추려고 일부러 타입을 바꿀 필요는 없고, 의미 기준으로 정하는게 맞다.

RM 레지스터는 그냥 '그릇'일 뿐

예를 들어 32bit ARM이면:

- r0~r3 → 32비트 레지스터

그래서 이런 것들은 전부 32비트로 실려서 감:

```c
char
short
int
pointer
```

컴파일러가 자동으로 **32비트로 확장**해서 전달

```c
void f(char a);

실제로는
a → r0 (32bit로 zero/sign extend 됨)
```

int로 통일하는게 빠를까?

```c
void set_flag(uint8_t flag);
```

이걸 굳이

```c
void set_flag(int flag);
```

이거로 바꿀 필요는 없음  (어차피 레지스터는 32비트라서 차이가 거의 없음)

**중요한 건**

1. 타입 의미를 유지하는 것

```c
uint8_t → 0~255 의미
int → 부호 있음
```

의미가 다름

1. 구조체 정렬

인자 타입보다, 구조체 내부 정렬이 더 중요

1. 64비트 타입은 다름

```c
uint64_t
double
```

32비트 ARM에서는

```c
r0 + r1 두 개 사용
```

그래서 레지스터 더 소비함 ⇒ 고려 대상

| 타입 | 신경 써야 하나? |
| --- | --- |
| char / short | ❌ 거의 영향 없음 |
| int / pointer | 기본 단위 |
| 64bit 타입 | ⭕ 레지스터 2개 사용 |
| 구조체 값 전달 | ⭕ 스택 사용 가능성 높음 |

### **※  리프 함수사용**

함수 깊이를 적게 사용하기 위해 리프 함수 사용
