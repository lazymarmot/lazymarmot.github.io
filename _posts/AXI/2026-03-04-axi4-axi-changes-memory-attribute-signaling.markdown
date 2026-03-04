---
layout: post
title:  "AXI4 – Changes to Memory Attribute Signaling"
date:   2026-03-04 12:00:00 +0900
categories: AXI4 AMBA
show_on_home: false
---

# AXI Changes to memory attribute signaling

AXI3의 AxCACHE 비트 의미:

| 비트 | 이름 |
| --- | --- |
| [0] | bufferable |
| [1] | cacheable |
| [2] | read allocate |
| [3] | write allocate |

### **AXI3 문제점 )**

AMBA AXI3 Spec을 요약하면 

> “AxCACHE provides *hints* about the nature of the transaction.”
> 
> 
> ⇒ AxCACHE 신호는 규칙이 아닌 힌트를 제공 하고 있다고 쓰여있음
> 

cacheable=1 이라고 해서 반드시 merge / reorder를 해야 하는 것도 아니고, 반드시 해도 되는 것도 아님

<aside>
💡

AXI3가 만들어질 때는:

- CPU + 단순 interconnect
- 캐시도 대부분 CPU 내부
- 멀티마스터, coherent fabric이 흔하지 않았음

그래서:

“이 트랜잭션이 캐시를 탈 수도 있다” 정도만 알려주면 충분하다고 생각했음.

메모리 모델(ARM ordering rules)까지 fabric이 깊게 이해해야 할 거라고는 생각 안 한 듯

</aside>

interconnect 설계자들이 항상 이렇게 추측해야 했음:

“cacheable이면 reorder 해도 되겠지?”

“bufferable이면 write buffer에 넣어도 되겠지?”

근데 **Normal Non-cacheable Bufferable** 같은 애들은?

- 캐시는 안 타지만 (non-cacheable)
- bufferable
- reorder는 허용되는 메모리

AXI3로 표현하면 

Cacheable = 0 → “캐시는 안타는 거니까 조심”

Bufferable = 1  → “write buffer는 써도 되나 보네?”

reorder는

ARM 메모리 모델에서는 reorder가 허용됨

근데 AXI3에서는 reorder 허용 여부를 표현하는 비트가 없음

⇒ 그 결과, fabric 벤더 (interconnect..)에서는 이렇게 혼자 판단하게 됨

“bufferable이니까 reorder해도 되나?”

“cacheable이 아니니까 reorder 하면 위험한가?”

<aside>
💡

왜 cacheable이면 reorder가 안전하냐??

normal cacheable memory는

- cpu 캐시에 들어감
- coherence 프로토콜이 있음
- ARM memory model이 “언제 다른 코어가 보이는지”를 규정
</aside>

계속 이렇게 힌트를 기반으로 추측을 해야 했음

: 각 SoC 벤더들이 AXI3 힌트를 자기 나름의 규칙으로 해석해서 interconnect를 만들어서 ARM, Synopsys, Arteris 각자 약간 씩 달랐음

AXI fabric 입장에서는

“이 트랜잭션을 **reorder / merge / buffer** 해도 되나?” 를 판단해야 하는데,

AXI3에서는 “cacheable?”, “bufferable?” 같은 캐시 중심 신호만 제공

---

AXI4에서는,

"캐시가 가능하냐"보다는 "트랜잭션이 중간에서 변경(병합, 재정렬) 되어도 되느냐"가 더 중요 해졌기 때문에

AxCACHE의 의미를 modifiable 중심으로 재정의 했다.

- AxCACHE[1] = 'cacheable'
    
    → AxCACHE[1] = 'modifiable' 
    

트랜잭션의 reorder / merge / buffer / combine 가능 여부를 

cacheable → modifiable로 표현 하기로 한 것!

| AxCACHE 의미 | Fabric 동작 | memory type |
| --- | --- | --- |
| modifiable=1 | reorder / merge / buffer 가능 | Normal |
| modifiable=0 | 절대 손대지 마 | Device |
|  |  |  |

Non-modifiable 트랜잭션에 대해 ordering 요구 사항이 정의 되었다

- Read-allocate와 write-allocate의 의미가 수정되었다.
    
    ![image.png](/assets/posts_image/AXI/AXI%20Changes%20to%20memory%20attribute%20signaling/image.png)
    
    - AXI3 → AXI4에서 RA/WA(Read-allocate / Write-allocate)가 “힌트 성격”으로 재정의 됐다
        
        AXI4에서는 RA/WA가 
        
        “이번 트랜잭션이 캐시 할당을 하느냐”가 아니라 
        
        “이 주소가 캐시에 있을 수도 있으니 캐시를 확인해 봐야 하느냐”를 나타낸다.
        
        | AXI3 | AXI4 |
        | --- | --- |
        | AXI3에서 RA / WA 의미
        Read-allocate (RA) → read miss 시 캐시에 올려도 된다
        
        Write-allocate (WA) → write miss 시 캐시에 올려도 된다
        
        즉, “이번 접근으로 캐시를 만들까?” 에 초점 | AXI4에서는 coherent system이 기본이 됨
          - 나말고도
          - 다른 cpu core, DMA, 가속기 들이 같은 캐시 쓰거나
             같은 주소를 미리 캐시에 올려 놓을 수 있음
        ⇒ “내가 이 주소를 처음 접근 하는게 아닐 수도 있다”
        
        RA/WA가 이렇게 바뀜  :
        “이 주소가 이미 캐시에 들어가 있을 수도 있다”는 가능성 힌트 |
        
        AXI4 Spec
        
        “RA/WA는 캐시를 만들 허가증이 아니라, 이미 캐시에 있을 가능성을 알려주는 힌트”
        
        | 비트 | AXI4에서의 의미 |
        | --- | --- |
        | WA | “이 주소가 write 때문에 캐시에 있을 수도 있음” |
        | RA | “이 주소가 read 때문에 캐시에 있을 수도 있음” |
        
        Read 트랜잭션 WA 비트
        
        Read 요청이 왔다고 해보자.
        
        WA=1이면 fabric은 이렇게 해석한다:
        
        > “이 주소는 과거에 누군가 write해서 캐시에 들어가 있을 수도 있다.”
        > 
        
        그 “누군가”에는:
        
        - 나 자신 (AXI3 의미)
        - **다른 master** (AXI4 새 의미)
        
        가 포함됨.
        
        그래서 fabric은:
        
        > “혹시 캐시에 있나 lookup 해봐야겠다.”
        > 
        
        Write 트랜잭션에서 RA 비트 의미
        
        Write 요청이 왔다고 해보자.
        
        RA=1이면:
        
        > “이 주소는 과거에 누군가 read해서 캐시에 들어가 있을 수도 있다.”
        > 
        
        역시:
        
        - 나 자신
        - 다른 master
        
        둘 다 가능.
        
        그래서:
        
        > “혹시 캐시에 있나 lookup 필요.”
        > 
        
        <aside>
        💡
        
        왜 이런 구조가 필요했나
        
        AXI4는 
        
        multi-core
        
        multi-master
        
        snoop / cache cohernece 환경을 전제로 설계 됐음
        
        1. 다른 master가 이미 캐시를 만들었을 수도 있음:
        2. 멀티코어 + coherent 시스템에서 이런 상황이 있음:
            
            Core0이:
            
            x = *(int*)0x8000
            
            → 캐시에 x를 올림
            
            Core1이:
            
            *(int*)0x8000 =5
            
            이때 Core1의 write는:
            
            “이 주소는 이미 캐시에 있을 수도 있음”
            
            을 fabric이 알아야:
            
            invalidate
            
            update
            
            snoop
            
            같은 coherence 동작을 함.
            
            그래서 RA/WA는:
            
            “이 주소는 cacheable 세계에 있다”는 표식이 된 거야.
            
        </aside>
        
        ![image.png](/assets/posts_image/AXI/AXI%20Changes%20to%20memory%20attribute%20signaling/image%201.png)
        
        ![image.png](/assets/posts_image/AXI/AXI%20Changes%20to%20memory%20attribute%20signaling/image%202.png)
        

ARM Architecture Reference Manual:

> Normal memory accesses may be speculative, reordered, and merged.
> 
> 
> **Device memory accesses must not be speculative, reordered, or merged.**
> 

위 내용을 보자면, 

| AXI3 | AXI4 |
| --- | --- |
| cacheable / bufferable 비트로
“대충 Normal인지 Device인지”를 힌트로 줬음
⇒ 그래서 fabric이 추측해야 했음. | AxCACHE를 memory type 인코딩으로 재 정의해서Normal → modifiable
Device → non-modifiable
를 명확히 알려줌. |

AXI4에서는 

reorder / merge / speculative(prefetch 포함) 기준은 **cacheable이 아니라 Normal vs Device** 이다.

| 메모리 타입 | reorder | merge | prefetch |
| --- | --- | --- | --- |
| Normal cacheable | ✔ | ✔ | ✔ |
| Normal non-cacheable | ✔ | ✔ | ✔ |
| Device | ❌ | ❌ | ❌ |

그 결과 interconnect는 이제 

“이 트랜잭션은 내가 좀 만져도 되나?”를 비트 해석 없이 바로 알 수 있음

ARM 메모리 모델 기준으로 reorder / merge / combine / prefetch 허용 여부는 **“Normal memory”**에서 결정된다.

```c
MMU (Normal / Device)
        ↓
CPU 내부 메모리 속성
        ↓
AXI AxCACHE (modifiable / non-modifiable)
        ↓
Interconnect: reorder / merge / prefetch 여부 결정
```

cacheable은 그 안에 있는 하위 속성으로

“이 normal memory가 캐시에 들어가도 되나?”  만 말해주는 하위 옵션으로 되버림

캐시에 들어간다?

그 메모리 주소의 데이터가 CPU의 L1/L2/L3 캐시에 복사본으로 저장 될 수 있다는 뜻

```c
x = *(int*)0x8000_0000;

cacheable이면 CPU는:
	DDR에서 0x8000_0000 값을 읽어옴
	그 값을 L1/L2 캐시에 저장
	이후에 같은 주소를 다시 읽으면 DDR 안 가고 캐시에서 바로 읽음

non-cacheable이면:
	매번 DDR로 감
```

<aside>
💡

cacheable은:

> “이 Normal memory를 CPU가 로컬에 저장해도 되냐”
> 

reorder, merge, prefetch는:

> “이 Normal memory를 fabric이 만져도 되냐”
> 

이 둘은 **다른 층위**야.

DMA 예로 딱 이해

DMA 버퍼를:

Normal non-cacheable로 설정하면

CPU는:

캐시에 안 저장함

매번 DDR 직접 봄

하지만:

reorder, merge, burst는 여전히 가능

왜? RAM이니까

</aside>

---

AxCACHE[1]

**Modifiable Transactions**

AxCACHE[1]값이 asserting된 경우 (1로 설정된 경우 : modifiable)

Modifiable 트랜잭션은 다음과 같은 방식으로 변경될 수 있다:

- 하나의 트랜잭션을 여러 개의 트랜잭션으로 분해할 수 있다
- 여러 개의 트랜잭션을 하나의 트랜잭션으로 병합할 수 있다
- 읽기(read) 트랜잭션은 필요한 것보다 더 많은 데이터를 가져올 수 있다
- 쓰기(write) 트랜잭션은 필요한 것보다 더 넓은 주소 범위에 접근할 수 있다 (단, WSTRB 신호를 사용해 실제로 갱신되는 바이트만 제한해야 한다)
- 생성된 각 트랜잭션에서 다음 신호들은 변경될 수 있다:
    - 전송 주소 AxADDR
    - 버스트 크기 AxSIZE
    - 버스트 길이 AxLEN
    - 버스트 타입 AxBURST
    

다음 항목들은 절대 변경되면 안 된다:

- 락 타입 AxLOCK
- 보호 속성 AxPROT

**AxCACHE 수정에 대한 제한**

- 메모리 속성 AxCACHE 자체는 수정될 수 있다
    - Interconnect가 내부 최적화 과정에서 새로 만들어낸 트랜잭션의 AxCACHE값을 바꿀 수는 있다.
    - 원본 트랜잭션: AxCACHE = Modifiable
    - 쪼개서 만든 하위 트랜잭션들도
        - 상황에 따라 AxCACHE를 조정할 수 있음
- 그러나 어떤 수정이든 다음 조건을 반드시 만족해야 한다:
    - 다른 컴포넌트가 해당 트랜잭션을 보지 못하게 만들어서는 안 된다
        
        원래 보였어야 할 캐시 / snoop / observer가 갑자기 이 트랜잭션을 못 보게 만들면 안 된다
        
        : 원래 cache lookup이 필요했던 트랜잭션을 interconnect가 마음대로
        
        - “이건 캐시 lookup 안 해도 되겠네” 라고 바꿔버리면 안됨
    - 트랜잭션 전파를 막으면 안된다.
        
        중간에서 “아 이건 내부에서 처리됐어” 하고 더 이상 전달 안 해버리면 안 됨
        
        - write를 받아놓고
        - 실제 메모리나 슬레이브로 안 보내면 안됨
        
        반드시 원래 도달해야 할 지점까지 전파돼야 함
        
    - 캐시 lookup 필요성을 변경해서는 안 된다
        - 원래는
            - cacheable → cache lookup 필요
        - 그런데 interconnect가
            - “어차피 합쳤으니까 non-cacheable로 바꾸자”
        
        이렇게 하면 안됨  => 캐시 일관성(coherency) 깨짐
        
    - 또한 같은 주소 범 위에 대한 모든 트랜잭션에 대해 일관되게 적용되어야 한다
        
        어떤 주소 범위(예: 0x8000_0000 ~ 0x8000_FFFF)에 대해 
        
        한 트랜잭션은 cacheable,
        
        다른 트랜잭션은 non-cacheable
        
        이런 식이면 안됨
        
        ⇒ 주소 범위 단위로 속성은 일관돼야 함
        

**기타 변경 가능 항목**

- Transaction ID
    - 내부 재정렬
    - 여러 트랜잭션을 합치거나 쪼갤 때
    - ID 재부여 필요
    
    ⇒  기능적으로 문제 없음
    
- QoS 값
    - 트래픽 관리용 힌트
    - 성능 정책용
    
    ⇒ 데이터 의미에는 영향 없음
    

**절대 허용되지 않는 변경**

다음과
같은 변경은 어떤 경우에도 허용되지 않는다:

- 원래 트랜잭션이 접근하던 4KB 주소 영역을 벗어나게 만드는 변경
    - 원래: 0x8000_1000 ~ 0x8000_10FF
    - interconnect가: 0x8000_1000 ~ 0x8000_20FF 로 바꾸면 ❌
    
     ⇒ 주소 디코딩 / protection / atomicity 다 깨짐
    
- 단일 접근으로 보장돼야 하는(single-copy atomicity) 영역을 여러 번의 접근으로 쪼개는 것
    
    이건 말 그대로:
    
    “한번에 처리돼야 의미가 유지되는 접근”을 여러 번으로 쪼개면 안 된다
    
    예:
    
    - atomic register
    - lock variable
    - device status bit
    
    이걸:
    
    - read 두 번
    - write 두 번
    
    으로 나누면 안됨
    
    ⇒ 동기화 / 락 / 장치 제어 붕괴
    

**Non-modifiable transaction**

AxCACHE[1] 값이 LOW 로 설정된 경우 (0일 경우)

시스템에 "이 트랜잭션은 절대 건드리지 말고 형태 그대로를 전달해"라고 하는거

그래서 interconnect / NoC / fabric / memory system전부 최적화 금지 모드로 들어감

**절대 금지 : split / merge**

안되는 것

Burst 쪼개기 안됨

여러 write 합치기 안됨

Read prefetch안됨

Write combine 안됨

이 표에 있는 신호들은 “절대 변경 불가”

![image.png](/assets/posts_image/AXI/AXI%20Changes%20to%20memory%20attribute%20signaling/image%203.png)

[](data:image/png;base64,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)

[](data:image/png;base64,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)

**AxCACHE 수정에 대한 "유일한 예외"**

> “AxCACHE는 Bufferable → Non-Bufferable로만 변경 가능
> 

왜 이건 허용될까?

- 원래:
    - Bufferable = “늦게 가도 됨”
- Non-bufferable로 바꾸면:
    - “더 보수적으로, 더 안전하게”

  ⇒ 시스템 가시성/정합성 감소가 없음

반대로:

- Non-bufferable → Bufferable → 위험 증가 → 금지

**ID / QoS는 왜 바꿔도 될까?**

Transaction ID

- interconnect 내부 재정렬용
- 기능적 의미 없음

QoS

- 성능 힌트
- 데이터 의미와 무관

⇒ 데이터 의미를 바꾸지 않기 때문에 허용

---

**Ordering Requirements for Non-modifiable transactions**

![image.png](/assets/posts_image/AXI/AXI%20Changes%20to%20memory%20attribute%20signaling/image%204.png)

이 문단은 AXI4에서 

“Non-modifiable 트랜잭션의 순서(ordering)를 누가, 어디까지 보장해야 하는가” 를 정확히 규정한 아주 중요한 규칙이야.

결론부터 말하면 **MMIO/디바이스 접근**의 정합성을 보장하는 핵심 규칙이다.

Normal Memory는 reorder OK이기 때문에 트랜잭션 순서랑 관련 없음

이 규칙이 말하는 한 줄 요약

> “Non-modifiable 트랜잭션은  ‘같은 ID’ + ‘같은 slave’로 가면 어떤 interconnect를 거쳐도 반드시 순서가 지켜져야 한다(보낸 순서 = 도착/완료 순서)”
> 

언제 “순서 보장”이 의무인가?

AXI4가 반드시 순서를 보장하라고 강제하는 조건은 이 3가지가 모두 만족 될 때야:

1. Non-modifiable (AxCACHE[1] = 0)
2. 같은 AXI ID
3. 같은 slave 디바이스

이 세 가지가 동시에 맞으면 주소가 달라도 순서 보장 필수.

문서에 있는 이 문장이 핵심

> *“The ordering must be preserved, irrespective of the address”*
> 

“주소가 달라도 순서 보장해야 된다” 예시

UART같은 MMIO 디바이스는 이렇게 생겼음

```c
UART base = 0x4000_0000

UART_CTRL = 0x4000_0000
UART_DATA = 0x4000_0004
UART_STATUS = 0x4000_0008
```

CPU입장에서는 

```c
*(UART_CTRL) = START;
*(UART_DATA) = 'A';
```

→ 서로 다른 주소에 write 2개

하지만 하드웨어 입장에서는:

→ 둘 다 같은 UART 장치(slave) 안의 레지스터

AXI에서 주소가 다르면 원래 reorder가 가능하다

AXI의 기본 철학은:

“다른 주소면 서로 독립적인 트랜잭션일 수도 있다”

예를 들어:

```c
0x8000_0000 (DDR)
0xA000_0000 (GPU buffer)
```

이건 순서 바꿔도 상관없다.

그래서 AXI는 기본적으로:

> 주소가 다르면 reorder 허용
> 

근데 MMIO는 주소가 달라도 순서가 의미가 있다 (중요)

UART 예

```c
UART_CTRL = ENABLE;
UART_DATA = 'A';
```

이게 바뀌면

```c
UART_DATA 먼저
UART_CTRL 나중
```

→ UART는:

“아직 enable 안 됐는데 data?”  → 데이터 날아감, 버그

그래서 ARM + AXI는:

“Device / Non-modifiable memory는 주소가 달라도 순서를 반드시 지켜라”

라고 규정함.

이 규칙이 위에서 얘기한 조건이 만족 했을 때 적용됨

세 가지가 동시에 맞으면:

- Non-modifiable (Device memory)
- 같은 Slave (같은 UART)
- 같은 ID

→ **주소가 달라도 절대 reorder 금지**

**만일 다른 slave로 가면?**

다른 slave 간에는 순서 보장 안 함

write UART

write GPIO

- slave가 다르면
- Non-modifiable라도

⇒ AXI는 순서 보장 안 함

이유:

- UART, GPIO는 물리적으로 완전히 다른 장치
- 순서를 묶을 이유가 없음

**slave 경계가 애매하면? (중요한 현실 규칙)**

이 문단이 SoC 현실을 반영한 부분이야:

> *“address map boundary is IMPLEMENTATION DEFINED”*
> 

즉,

- interconnect 입장에서
- “이 주소가 어느 slave인지” 확실히 모를 수도 있음

⇒ 그 경우엔 보수적으로 처리:

같은 AXI ID로 같은 path를 타는 모든 Non-modifiable 트랜잭션의 순서를 보장하라 (안전 우선 규칙)

**Bufferable / Non-bufferable 섞여도?**

이 문장도 중요해:

> *“applies between all Non-modifiable transactions, including between Non-bufferable and Bufferable”*
> 

즉,

- AxCACHE[1]=0 (Non-modifiable) 이면
- AxCACHE[0]=0/1 (bufferable 여부)와 상관없이
    
    ⇒  순서 보장 대상
    

**누가 책임지나?** 

중간 컴포넌트(interconnect)가 응답을 생성한다면

그 컴포넌트가 순서 보장 책임자가 된다. 

즉,

- interconnect
- NoC
- bridge

같은 애들이:

- 내부에서 reorder / buffer / split 했다면 ⇒ 최종 응답 순서가 맞도록 책임져야 함

**Read ↔ Write 사이의 순서**

읽기 채널과 쓰기 채널은 독립되어 있음 

그래서 AXI는 이렇게 말해:

⇒ Read/Write 간 순서는 “자동 보장 아님”

한 방향(read/write)의 트랜잭션은 다른 방향의 이전 트랜잭션에 대한 응답을 받은 뒤에

다음 트랜잭션을 발행해야만 순서가 보장된다.

예시

write A

read B

- write 응답(BVALID/BREADY) 받고 나서
- read를 발행해야

write → read 순서가 보장됨

응답 기다리지 않고 read를 먼저 쏘면

→ 순서 보장 없음, 이건 소프트웨어(드라이버) 책임이야.

이 규칙이 없으면 생기는 버그

- MMIO write 순서 뒤집힘
- device state machine 깨짐
- interrupt ack → enable 순서 꼬임
- secure register race
    
    ⇒ 디버깅 지옥
    

그래서 AXI4는:

- Modifiable / Non-modifiable을 나누고
- Non-modifiable에는 강제 ordering 규칙을 둔 거야.

[Memory Type](AXI%20Changes%20to%20memory%20attribute%20signaling/Memory%20Type%202e96feb16a3e80709f58cb8760def393.md)