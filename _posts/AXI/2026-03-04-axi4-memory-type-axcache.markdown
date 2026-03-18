---
layout: post
title:  "AXI4 – Memory Type (AxCACHE)"
date:   2026-03-04 12:00:00 +0900
categories: AXI4 AMBA
show_on_home: false
---

**AXI3**

→ AxCACHE = 비트 의미 위주 (cacheable, bufferable 등)

<br>

AXI4 AxCACHE 비트 정의

| 비트 | 의미 |
| --- | --- |
| [0] | Bufferable |
| [1] | Modifiable |
| [2] | Read-allocate |
| [3] | Write-allocate |

<br>

AxCACHE[1](modifiable) :  “이 접근은 어떤 메모리 성격이냐”를 나타냄 (Normal, Device, Non-cacheable 등)

- 같은 “Normal memory”라도 read/write 채널에서 AxCACHE 값이 다를 수 있음
    
    
    | 타입 | 캐시 |
    | --- | --- |
    | Device | ❌ |
    | Normal non-cacheable | ❌ |
    | Normal Write-through | ✔ |
    | Normal Write-back | ✔ |

AXI4는 

![image.png](/assets/posts_image/AXI/Memory%20Type/image.png)

“AxCACHE 비트 4개를 따로 해석하지 말고 이 4개를 조합해서 ‘메모리 타입’으로 해석” 하도록 바꿨다. 

> “modifiable은 “Normal memory냐 Device냐”를 결정하는 핵심 비트”
> 

> “memory type encoding을 통해 사람이 이해하기 쉽게 이름 붙인 것”
> 

⇒ AXI4에서는 AxCACHE비트 조합을 "의미 있는 메모리 타입 이름"으로 정리했고 

⇒ AXI3와의 호환을 위해 같은 메모리 타입에 여러 AxCACHE 인코딩을 허용한다. (AXI3 값도 여전히 허용됨)

![image.png](/assets/posts_image/AXI/Memory%20Type/image%201.png)

```c
AxCACHE = 0011
→ Normal Non-cacheable Bufferable
→ modifiable = 1

AxCACHE = 0000
→ Device Non-bufferable
→ modifiable = 0
```

AxCACHE신호를 fabric이 즉시 판단

: AxCACHE → memory type → rules

|  | **Normal memory** | **Device memory** | **Normal Non-cacheable memory** |
| --- | --- | --- | --- |
| 성격 | 일반적인 RAM (DDR, SRAM, L2/L3 뒤의 메모리, heap/stack/array)<br>“write-through, write-back은 100% cacheable memory이다” | 레지스터 / 장치 제어용 메모리 (MMIO) | RAM이지만 캐시는 쓰면 안 되는 메모리<br>(Normal과 Device의 중간) |
| 시스템이 해도 되는 것 / 안되는 것 | - CPU L1/L2/L3 캐시 사용<br>◦ ✅ read prefetch<br>◦ ✅ write combine<br>◦ ✅ reordering<br>◦ ✅ speculative access<br>◦ ✅ burst / merge / split | 시스템이 하면 안 되는 것<br>• ❌ 캐시<br>• ❌ prefetch<br>• ❌ write combine<br>• ❌ reordering<br>• ❌ speculative access<br><br>반드시 지켜야 할 것<br>• ✅ 순서 보장<br>• ✅ 접근 횟수 정확<br>• ✅ 즉시 반영 | 시스템이 해도 되는 것 / 안 되는 것<br>• ❌ CPU 캐시 (가장 중요)<br>• ❌ read prefetch (보통)<br>• ⭕ write buffer (상황에 따라)<br>• ⭕ 순차 burst<br>• ⭕ ordering은 보장되는 쪽 |
| AXI 관점 | ◦ Modifiable<br>◦ Cacheable<br>◦ Bufferable<br>◦ RA/WA 가능 | ◦ Non-modifiable<br>◦ Non-cacheable<br>◦ 보통 Non-bufferable | ◦ Cacheable ❌<br>◦ Modifiable ❌ 또는 제한적<br>◦ Bufferable은 경우에 따라 ⭕/❌ |
| 예시 | ◦ heap / stack<br>◦ 전역 변수<br>◦ 배열 (arr[i])<br>◦ 코드 영역 | ◦ UART / SPI / I2C 레지스터<br>◦ GPIO<br>◦ DMA control register<br>◦ interrupt controller | ◦ DMA buffer (non-coherent 시스템)<br>◦ frame buffer<br>◦ shared memory (cache flush 귀찮을 때)<br>◦ HW가 직접 읽는 메모리 |
|  | 성능 최적화 최우선 메모리 | “한 번 쓰고, 한 번 읽는 게 의미인” 메모리 | “RAM이지만 캐시 일관성 문제를 피하려고 캐시를 끈 메모리” |


**최적화가 일어나는 레벨은 두 군데가 있다.**

[CPU 내부]

- store buffer
- write combining
- speculative read
- cache line fill

[AXI / Interconnect]

- write merge
- reordering
- burst forming
- path scheduling

Device Memory

| 위치 | 허용 |
| --- | --- |
| CPU 내부 | ❌ |
| Interconnect | ❌ |

Normal Memory

| 위치 | 허용 |
| --- | --- |
| CPU 내부 | ✔ |
| Interconnect | ✔ |

---

### **메모리 타입 요구 사항**

이 절은 각 메모리 타입에 대해 필수적으로 지켜야 하는 동작을 규정한다.

<br>

**Device Non-bufferable 메모리 (0000)**

- "Device Non-bufferable 메모리는 중간에 아무 최적화도 하지 말고 실제 디바이스와 1:1로 순서대로 통신해"라는 규칙

    | 위치 | 허용 |
    | --- | --- |
    | CPU 내부 | ❌ cache, ❌ store buffer, ❌ prefetch |
    | Interconnect | ❌ reorder, ❌ merge, ❌ burst 재구성, ❌ speculative |

    Device Non-bufferable 메모리에 대해 요구되는 동작은 다음과 같다:

    - 쓰기 응답(write response)은 반드시 최종 목적지(slave 디바이스)로 부터 받아야 한다.
    - 읽기 데이터(read data)는 반드시 최종 목적지로부터 가져와야 한다.
    - 모든 트랜잭션은 Non-modifiable이어야 한다. 
    (자세한 내용은 *Non-modifiable transactions* 절 참조)
    - 읽기(read)는 prefetch되면 안 되며, 쓰기(write)는 merge되면 안 된다.
    - 같은 AXI ID를 사용하고 같은 slave로 향하는 모든 Non-modifiable read/write 트랜잭션 (즉, AxCACHE[1] = 0)은 반드시 순서가 유지되어야 한다.
    - 디바이스 메모리 중에서도 가장 보수적인 규칙 세트라고 보면 될 듯

용도

    순서/타이밍이 곧 의미인 레지스터 (주로 명령, 제어 레지스터 설정 시)

    UART, GPIO, Timer, Interrupt Controller  레지스터 접근

    ```c
    UART_CTRL = ENABLE;
    SPI_CTRL = START;
    DMA_START = 1;
    GPIO_DIR = OUTPUT;
    ```

<br>

**Device Bufferable 메모리 (0001)**

- "Device 메모리 인데, 쓰기 응답만 중간에서 받아도 되는 타입"

    “CPU는 write를 잠깐 미뤄도 되지만, interconnect는 절대 재배열, 병합, 지연 하면 안됨”

    ⇒ Device는 레지스터 쓰기 / 순서가 의미가 있는데, write 응답을 기다리느라고 CPU가 멈추는 건 싫음

    | 위치 | 허용 |
    | --- | --- |
    | CPU 내부 | ❌ cache, ✔ store buffer, ❌ prefetch |
    | Interconnect | ❌ reorder, ❌ merge, ❌ burst 재구성, ❌ speculative |

    Device Bufferable 메모리 타입에 대해 요구되는 동작은 다음과 같다:

    - 쓰기 응답(write response)은 중간 지점(intermediate point)에서 받아도 된다.
    - 쓰기 트랜잭션은 *Transaction buffering* 절(A4-72)에 정의된 바에 따라, 최종 목적지에 적절한 시점 내에 반드시 반영되어야 한다.
    - 읽기 데이터(read data)는 반드시 최종 목적지로부터 얻어야 한다.
    - 모든 트랜잭션은 Non-modifiable이어야 한다. (자세한 내용은 *Non-modifiable transactions* 절 A4-62 참조)
    - 읽기는 prefetch되면 안 되며, 쓰기는 merge되면 안 된다.
    - 같은 AXI ID를 사용하고 같은 슬레이브를 대상으로 하는 모든 Non-modifiable 읽기 및 쓰기 트랜잭션(AxCACHE[1] = 0)은 반드시 순서가 유지되어야 한다.

    CPU 내부 write buffer(store buffer)

    ```c
    UART_TX = 'A';
    ```

    - CPU는 이 write를
    
        → **store buffer에 잠깐 넣고**
    
        → 다음 명령 실행 가능
    
    - 실제 AXI write는
    
        → 조금 뒤에 장치로 나감
    

    ⇒ CPU가 MMIO write 때문에 매번 멈추지 않아도 됨.

    Interconnect

    - interconnect는 이 트래픽에 대해 아무것도 바꾸면 안됨, 받는 순서, 주소 그대로 장치로 전달

    | 기능 | 허용 |
    | --- | --- |
    | reorder | ❌ |
    | merge | ❌ |
    | burst 재구성 | ❌ |
    | speculative | ❌ |

    용도

    주로 데이터 밀어 넣는 레지스터 (write여러 개를 FIFO에 밀어 넣을 때)

    write 응답을 기다리느라 CPU가 멈추는 걸 피하고 싶을 때

    ```c
    UART_TX = 'A';
    SPI_DATA = 0x55;
    I2C_TXFIFO = data;
    ```

<br>

AXI4문서 에서는 Device Memory = Non-modifiable memory를 같은 의미로 사용

- ⇒ Write latency를 좀 줄이고 싶을 때 사용 (값 자체보다는 써졌다는 사실이 중요한 경우)

| 타입 | CPU write buffer | Fabric reorder | 의미 |
| --- | --- | --- | --- |
| Device NB (0000) | ❌ | ❌ | 완전 직결 |
| Device B (0001) | ✔ | ❌ | 순서 유지 + 비동기 write |

**Normal Non-cacheable Non-bufferable (0010)**

“CPU가 RAM(DDR/SRAM)을 접근하되, 캐시(L1/L2/L3)랑 write buffer는 쓰지 말고 그냥 직접 메모리 처럼 다뤄라”

“그래도 RAM이니까 interconnect쪽 reorder / merge / burst 최적화는 그대로 허용한다 (RAM의미를 깨지 않는 범위의 최적화)”

| 위치 | 허용 |
| --- | --- |
| CPU 내부 | ❌ cache, ❌ store buffer, ❌ prefetch |
| Interconnect | ✔ reorder, ✔ merge,  ✔burst 재구성, ✔path scheduling, ✔bank interleave |

Normal Non-cacheable Non-bufferable 메모리 타입에 대해 요구되는 동작은 다음과 같다:

- 쓰기 응답(write response)은 반드시 최종 목적지로부터 받아야 한다.
- 읽기 데이터(read data)는 반드시 최종 목적지로부터 획득되어야 한다.
- 트랜잭션은 Modifiable이어야 한다.(A4-63절 *Modifiable transactions* 참조)
- 쓰기는 merge될 수 있다.
- 같은 AXI ID를 사용하고 주소가 서로 겹치는(overlap) 읽기 및 쓰기 트랜잭션은 반드시 순서가 유지되어야 한다.

CPU 내부

- CPU는 이 영역을 진짜 DDR에 직접 접근하는 것 처럼 다룸

| 기능 | 허용 |
| --- | --- |
| L1/L2/L3 캐시 | ❌ |
| store buffer | ❌ |
| prefetch / speculative | ❌ |
| 캐시 lookup | ❌ |

<br>

Interconnect 

- AXI4 + ARM memory model 기준 normal memory이면

    | 기능 | 허용 |
    | --- | --- |
    | reorder | ✔ |
    | merge | ✔ |
    | burst 재구성 | ✔ |
    | path scheduling | ✔ |
    | bank interleave | ✔ |

    조건 이 있음 (RAM 의미를 깨는 reorder는 금지)

    - 같은 ID + 겹치는 주소(overlapping address)는 순서 보장해야 함
    - barrier(DMB/DSB) 걸린 트랜잭션은 순서 보장
    - exclusive, atomic은 순서 보장

    ⇒ “RAM 의미만 지키면 어떻게 보내든 상관 없다” 

    용도 :

    - DMA buffer, shared RAM:

        - CPU는 캐시 끄고 직접 접근
        - fabric은 burst, merge로 대역폭 최대화
        - DDR bank 병렬화로 성능 유지

→ **“coherence만 끈 RAM”**

<br>

**Normal Non-cacheable Bufferable (0011)**

"일반 RAM인데 CPU캐시는 쓰지 말고, 

CPU write buffer + 시스템(interconnect 등)의 최적화는 전부 쓰자"

“coherence 문제는 피하고, 성능은 최대한 살리는 RAM”

| 위치 | 허용 |
| --- | --- |
| CPU 내부 | ❌ cache, ✔ store buffer, ✔ write combining, ❌ speculative load(prefetch) |
| Interconnect | ✔ reorder, ✔ merge,  ✔burst 재구성, ✔path scheduling, ✔bank interleave |

Normal Non-cacheable Bufferable 메모리 타입에 대해 요구되는 동작은 다음과 같다:

- 쓰기 응답(write response)은 중간 지점(intermediate point)에서 받을 수 있다.
- 쓰기 트랜잭션은 A4-72절 *Transaction buffering*에 정의된 바와 같이, 최종 목적지에서 적절한 시점 내에 반드시 가시화되어야 한다. 단, 쓰기 트랜잭션이 최종 목적지에서 언제 가시화되었는지를 판별할 수 있는 메커니즘은 없다.
- 읽기 데이터(read data)는 다음 중 하나로부터 획득되어야 한다:
    - 최종 목적지
    - 최종 목적지로 진행 중인(write가 아직 완료되지 않은) 쓰기 트랜잭션
- 읽기 데이터가 쓰기 트랜잭션으로부터 획득되는 경우:
    - 가장 최신의 쓰기 버전으로부터 획득되어야 하며
    - 이후의 읽기 트랜잭션을 처리하기 위해 캐시에 저장되어서는 안 된다.
- 트랜잭션은 Modifiable이어야 한다. (A4-63절 *Modifiable transactions* 참조)
- 쓰기는 merge될 수 있다.
- 같은 AXI ID를 사용하고 주소가 서로 겹치는(overlap) 읽기 및 쓰기 트랜잭션은 반드시 순서가 유지되어야 한다.

CPU 내부

| 기능 |  |
| --- | --- |
| 캐시 | L1/L2/L3 캐시에 절대 안들어 감
    x = *(0x8000);
 → 매번 DDR로 감
 → 캐시 lookup도 안함 |
| store (write)  | store (write) 명령
     *(0x8000) = 5;
CPU는 
   → 이 값을 store buffer에 먼저 넣음
   → 즉시 다음 명령 실행

store buffer 
     [ addr=0x8000, data=5 ]

즉, CPU는 write를 비동기로 보냄
 |
| Read after Write | *(0x8000) = 5;
r = *(0x8000);

이 때 CPU는 
    → DDR 가기전에 store buffer에 5가 있는지 먼저 확인
    → 있으면 그걸 읽음

⇒ 그래서 캐시는 없지만 자기 write는 자기가 본다 |

Interconnect

AXI4 관점:

> Normal + modifiable
> 

그래서 시스템 (interconnect 등..)dms 

| 기능 | 동작 |
| --- | --- |
| reorder | ✔ |
| merge | ✔ |
| burst | ✔ |
| path scheduling | ✔ |
| bank interleave | ✔ |
| write buffer | ✔ |
| intermediate write response | ✔ |

CPU가 이렇게 보냄

```c
store 0x8000 = 1
store 0x8004 = 2
store 0x9000 = 3
```

fabric (interconnect)는

- 주소, bank, slave 상태 보고
- 순서 바꾸고
- burst로 묶고
- 빈 경로 먼저 사용

해서 DDR에 보냄

Read가 Write보다 먼저 오면?

```c
*(0x8000) = 5;   // 아직 DDR 안 감
r = *(0x8000);
```

fabric (interconnect)는

- write buffer에 5가 있는 걸 보고
- **DDR 안 가고 그 5를 바로 반환**

→ 이게 spec의 “read may come from a pending write” 의미.

⇒ “CPU 캐시 때문에 coherence 꼬이지 않게 하면서, DDR 대역폭은 풀로 쓰고 싶을 때 사용”

**write response를 중간에서 받아도 된다**

CPU write → interconnect buffer → (나중에) DDR

- CPU는 쓰기 완료 응답을 빨리 받음
- 실제 메모리 반영은 약간 뒤

⇒ 성능 ↑, 👉 하지만 정확한 완료 시점은 CPU가 모름

그래서 문서에 

“언제 final destination에 보였는지 알 방법이 없다”

라고 명시됨

**read는 두 군데서 올 수 있다 (이게 핵심)**

읽기 데이터는:

1. 최종 메모리(DDR)에서 오거나
2. 아직 메모리에 쓰이기 전인 write buffer에서 바로 올 수도 있음

이걸 보통:

read-after-write forwarding

(store-to-load forwarding)

이라고 부름.

엄격한 조건 있음:

- 반드시 가장 최신 write 값
- 이 read 결과를 캐시에 저장하면 안 됨

⇒ 일회성 전달만 허용

**왜 이런 read 허용이 필요하냐?**

```c
buf[0] = 1;
x = buf[0];

- buf가 non-cacheable bufferable이면
- write가 아직 DDR에 안 갔어도
- 바로 x = 1 이 나와야 함

=> 프로그램 의미 보존 + 성능
```

**Writes can be merged**

```c
write A
write A+4
write A+8

→ 하나의 burst로 묶어서 DDR로

=> 메모리 대역폭 효율 ↑
```

**Ordering 규칙 (주소 겹칠 때만)**

- 같은 ID
- 주소가 겹칠 때만 순서 보장

용도

- DMA ring buffer
- 네트워크 RX/TX 버퍼
- GPU 공유 메모리

**Write-through No-allocate**

Write-through No-allocate 메모리 타입에 대해 요구되는 동작은 다음과 같다:

- 쓰기 응답(write response)은 중간 지점(intermediate point)에서 받을 수 있다.
- 쓰기 트랜잭션은 A4-72절 *Transaction buffering*에 정의된 바와 같이, 최종 목적지에 적절한 시점 내에 반드시 가시화되어야 한다. 단, 쓰기 트랜잭션이 최종 목적지에서 언제 가시화되었는지를 알 수 있는 메커니즘은 없다.
- 읽기 데이터(read data)는 중간에 존재하는 캐시된 복사본(intermediate cached copy)으로부터 얻을 수 있다.
- 트랜잭션은 Modifiable이다. (A4-63절 *Modifiable transactions* 참조)
- 읽기는 prefetch될 수 있다.
- 쓰기는 merge될 수 있다.
- 읽기 및 쓰기 트랜잭션 모두에 대해 캐시 lookup이 필요하다.
- 같은 AXI ID를 사용하고 주소가 겹치는(overlap) 읽기 및 쓰기 트랜잭션은 반드시 순서가 유지되어야 한다.
- No-allocate 속성은 “할당 힌트(allocation hint)”이다. 즉, 성능상의 이유로 이 트랜잭션을 캐시에 할당하지 않는 것이 바람직하다는 권고일 뿐이며, 읽기 및 쓰기 트랜잭션의 캐시 할당 자체가 금지되는 것은 아니다.

<br>

Write-through

- write 시:
    - 캐시에만 쓰고 끝 ❌
    - 메모리(DDR)까지 반드시 같이 반영 ⭕
- 그래서:
    - cache와 memory가 항상 같은 값 유지
    - write-back처럼 dirty line 관리 안 함

⇒  정합성 단순 / 성능은 중간

<br>

No-allocate

- read / write miss 시:
    - “굳이 캐시 라인을 만들지 마라”는 힌트
- 하지만:
    - ❌ 금지는 아님
    - 상황에 따라 캐시가 만들어질 수도 있음

⇒ “가능하면 캐시 오염 줄이자” 정도의 의미

항목별로 현실적으로 풀어보면

write response를 중간에서 받아도 된다

- interconnect / cache가 응답 먼저 줌
- 실제 메모리 반영은 뒤에서 진행

 ⇒ write latency 감소

read는 캐시에서 바로 나올 수 있다

- read hit 가능
- 그래서:
    - cache lookup이 반드시 필요
    
    ⇒ “cacheable한 Normal memory”라는 뜻
    
<br>

Reads can be prefetched

- 연속 접근 예상되면
- 다음 cache line 미리 가져옴

⇒ 성능 ↑ (전형적인 Normal memory 특징)

<br>

Writes can be merged

- 작은 write 여러 개 → burst로 합침
- AXI Modifiable이기 때문에 가능

⇒ 메모리 대역폭 효율 ↑

No-allocate는 “힌트”일 뿐

이 문장이 핵심이야:

*allocation is not prohibited*

즉:

- write-through + no-allocate는
    - “가급적 캐시 만들지 마”이지
    - “절대 캐시 쓰지 마”가 아님

⇒ 그래서 cache lookup은 항상 필요

언제 이런 타입을 쓰냐? (실전 예시)

잘 맞는 경우

- 스트리밍 데이터
- 한번 쓰고 다시 안 읽는 데이터
- large buffer sequential write
- cache pollution을 줄이고 싶은 경우

안 맞는 경우

- MMIO (Device 타입 써야 함)
- DMA buffer (보통 non-cacheable)

 

![image.png](/assets/posts_image/AXI/Memory%20Type/image%202.png)

![image.png](/assets/posts_image/AXI/Memory%20Type/image%203.png)