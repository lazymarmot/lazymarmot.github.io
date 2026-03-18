---
layout: post
title:  "AXI4 – Changes to Memory Attribute Signaling"
date:   2026-03-04 12:00:00 +0900
categories: AXI4 AMBA
show_on_home: false
---

<br>

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

```smalltalk
💡

AXI3가 만들어질 때는:

- CPU + 단순 interconnect
- 캐시도 대부분 CPU 내부
- 멀티마스터, coherent fabric이 흔하지 않았음

그래서:

“이 트랜잭션이 캐시를 탈 수도 있다” 정도만 알려주면 충분하다고 생각했음.

메모리 모델(ARM ordering rules)까지 fabric이 깊게 이해해야 할 거라고는 생각 안 한 듯

```

<br>

interconnect 설계자들이 항상 이렇게 추측해야 했음:

“cacheable이면 reorder 해도 되겠지?”

“bufferable이면 write buffer에 넣어도 되겠지?”

<br>

근데 **Normal Non-cacheable Bufferable** 같은 애들은?

- 캐시는 안 타지만 (non-cacheable)
- bufferable
- reorder는 허용되는 메모리

AXI3로 표현하면 

- Cacheable = 0 → “캐시는 안타는 거니까 조심”

- Bufferable = 1  → “write buffer는 써도 되나 보네?”

    reorder는,

    ARM 메모리 모델에서는 reorder가 허용됨

    근데 AXI3에서는 reorder 허용 여부를 표현하는 비트가 없음

⇒ 그 결과, fabric 벤더 (interconnect..)에서는 이렇게 혼자 판단하게 됨

- “bufferable이니까 reorder해도 되나?”

- “cacheable이 아니니까 reorder 하면 위험한가?”

```smalltalk
💡

왜 cacheable이면 reorder가 안전하냐??

normal cacheable memory는

- cpu 캐시에 들어감
- coherence 프로토콜이 있음
- ARM memory model이 “언제 다른 코어가 보이는지”를 규정
```

<span style="color:red">
계속 이렇게 힌트를 기반으로 추측을 해야 했음

: 각 SoC 벤더들이 AXI3 힌트를 자기 나름의 규칙으로 해석해서 interconnect를 만들어서 ARM, Synopsys, Arteris 각자 약간 씩 달랐음
</span>

<br>

AXI fabric 입장에서는

“이 트랜잭션을 **reorder / merge / buffer** 해도 되나?” 를 판단해야 하는데,

AXI3에서는 “cacheable?”, “bufferable?” 같은 캐시 중심 신호만 제공

<br>

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

<br>

Non-modifiable 트랜잭션에 대해 ordering 요구 사항이 정의 되었다

- Read-allocate와 write-allocate의 의미가 수정되었다.
    
    ![image.png](/assets/posts_image/AXI/AXI%20Changes%20to%20memory%20attribute%20signaling/image.png)
    
    - AXI3 → AXI4에서 RA/WA(Read-allocate / Write-allocate)가 “힌트 성격”으로 재정의 됐다
        
        <span style="color:#00aa00;">AXI4에서는 RA/WA가 “이번 트랜잭션이 캐시 할당을 하느냐”가 아니라 “이 주소가 캐시에 있을 수도 있으니 캐시를 확인해 봐야 하느냐”를 나타낸다.</span>
        
        | AXI3 | AXI4 |
        | --- | --- |
        | AXI3에서 RA / WA 의미<br>Read-allocate (RA) → read miss 시 캐시에 올려도 된다<br><br>Write-allocate (WA) → write miss 시 캐시에 올려도 된다<br><br>즉, “이번 접근으로 캐시를 만들까?” 에 초점 | AXI4에서는 coherent system이 기본이 됨<br>- 나말고도<br>- 다른 cpu core, DMA, 가속기 들이 같은 캐시 쓰거나<br>&nbsp;&nbsp;같은 주소를 미리 캐시에 올려 놓을 수 있음<br>⇒ “내가 이 주소를 처음 접근 하는게 아닐 수도 있다”<br><br>RA/WA가 이렇게 바뀜  :<br>“이 주소가 이미 캐시에 들어가 있을 수도 있다”는 가능성 힌트 |
        
        AXI4 Spec
        
        “RA/WA는 캐시를 만들 허가증이 아니라, 이미 캐시에 있을 가능성을 알려주는 힌트”
        
        | 비트 | AXI4에서의 의미 |
        | --- | --- |
        | WA | “이 주소가 write 때문에 캐시에 있을 수도 있음” |
        | RA | “이 주소가 read 때문에 캐시에 있을 수도 있음” |
        
        <br>

        Read 트랜잭션 WA 비트
        
        - Read 요청이 왔다고 해보자.
        
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
        
        <br>

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
        
        ```smalltalk
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
            
        ```
        
        ![image.png](/assets/posts_image/AXI/AXI%20Changes%20to%20memory%20attribute%20signaling/image%201.png)
        
        ![image.png](/assets/posts_image/AXI/AXI%20Changes%20to%20memory%20attribute%20signaling/image%202.png)
        
<br>

ARM Architecture Reference Manual:

> Normal memory accesses may be speculative, reordered, and merged.
> 
> 
> **Device memory accesses must not be speculative, reordered, or merged.**
> 

<br>

위 내용을 보자면, 

| AXI3 | AXI4 |
| --- | --- |
| cacheable / bufferable 비트로 “대충 Normal인지 Device인지”를 힌트로 줬음<br>⇒ 그래서 fabric이 추측해야 했음. | AxCACHE를 memory type 인코딩으로 재정의해서<br>Normal → modifiable<br>Device → non-modifiable<br>를 명확히 알려줌. |

<br>

AXI4에서는 

reorder / merge / speculative(prefetch 포함) 기준은 **cacheable이 아니라 Normal vs Device** 이다.

| 메모리 타입 | reorder | merge | prefetch |
| --- | --- | --- | --- |
| Normal cacheable | ✔ | ✔ | ✔ |
| Normal non-cacheable | ✔ | ✔ | ✔ |
| Device | ❌ | ❌ | ❌ |

그 결과 interconnect는 이제 

“이 트랜잭션은 내가 좀 만져도 되나?”를 비트 해석 없이 바로 알 수 있음

<br>

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

- “이 normal memory가 캐시에 들어가도 되나?”  만 말해주는 하위 옵션으로 되버림

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

<br>

```smalltalk
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

```

---

**AxCACHE[1]**

- ***Modifiable Transactions***

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
    
    <br>
    다음 항목들은 절대 변경되면 안 된다:

    - 락 타입 AxLOCK
    - 보호 속성 AxPROT

    <br>

    ***AxCACHE 수정에 대한 제한***

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
        
    <br>

    ***기타 변경 가능 항목***

    - Transaction ID
        - 내부 재정렬
        - 여러 트랜잭션을 합치거나 쪼갤 때
        - ID 재부여 필요
    
        ⇒  기능적으로 문제 없음
    
    - QoS 값
        - 트래픽 관리용 힌트
        - 성능 정책용
    
        ⇒ 데이터 의미에는 영향 없음
    
    <br>

    ***절대 허용되지 않는 변경***

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
    
<br>

***Non-modifiable transaction***

AxCACHE[1] 값이 LOW 로 설정된 경우 (0일 경우)

시스템에 "이 트랜잭션은 절대 건드리지 말고 형태 그대로를 전달해"라고 하는거

그래서 interconnect / NoC / fabric / memory system전부 최적화 금지 모드로 들어감

<br>

- **절대 금지 : split / merge**

    안되는 것

    - Burst 쪼개기 안됨

    - 여러 write 합치기 안됨

    - Read prefetch안됨

    - Write combine 안됨

    이 표에 있는 신호들은 “절대 변경 불가”

    ![image.png](/assets/posts_image/AXI/AXI%20Changes%20to%20memory%20attribute%20signaling/image%203.png)


    <br>

    ***AxCACHE 수정에 대한 "유일한 예외"***

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

<br>

***Ordering Requirements for Non-modifiable transactions***

![image.png](/assets/posts_image/AXI/AXI%20Changes%20to%20memory%20attribute%20signaling/image%204.png)

이 문단은 AXI4에서 

> “Non-modifiable 트랜잭션의 순서(ordering)를 누가, 어디까지 보장해야 하는가” 를 정확히 규정한 아주 중요한 규칙이야.

결론부터 말하면 **MMIO/디바이스 접근**의 정합성을 보장하는 핵심 규칙이다.

Normal Memory는 reorder OK이기 때문에 트랜잭션 순서랑 관련 없음

<br>

이 규칙이 말하는 한 줄 요약

> “Non-modifiable 트랜잭션은  ‘같은 ID’ + ‘같은 slave’로 가면 어떤 interconnect를 거쳐도 반드시 순서가 지켜져야 한다(보낸 순서 = 도착/완료 순서)”
> 

<br>

언제 “순서 보장”이 의무인가?

AXI4가 반드시 순서를 보장하라고 강제하는 조건은 이 3가지가 모두 만족 될 때야:

1. Non-modifiable (AxCACHE[1] = 0)
2. 같은 AXI ID
3. 같은 slave 디바이스

<span style="color:red">이 세 가지가 동시에 맞으면 주소가 달라도 순서 보장 필수.</span>

<br>

문서에 있는 이 문장이 핵심

> *“The ordering must be preserved, irrespective of the address”*
> 

<br>

“주소가 달라도 순서 보장해야 된다” 예시

- UART같은 MMIO 디바이스는 이렇게 생겼음

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

<br>

하지만 하드웨어 입장에서는:

→ 둘 다 같은 UART 장치(slave) 안의 레지스터

<br>

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

<br>

근데 MMIO는 주소가 달라도 순서가 의미가 있다 (중요)

- UART 예

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

<br>

그래서 ARM + AXI는:

“Device / Non-modifiable memory는 주소가 달라도 순서를 반드시 지켜라”

라고 규정함.

이 규칙이 위에서 얘기한 조건이 만족 했을 때 적용됨

세 가지가 동시에 맞으면:

- Non-modifiable (Device memory)
- 같은 Slave (같은 UART)
- 같은 ID

→ **주소가 달라도 절대 reorder 금지**

<br>

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

<br>

**slave 경계가 애매하면? (중요한 현실 규칙)**

이 문단이 SoC 현실을 반영한 부분이야:

> *“address map boundary is IMPLEMENTATION DEFINED”*
> 

즉,

- interconnect 입장에서
- “이 주소가 어느 slave인지” 확실히 모를 수도 있음

⇒ 그 경우엔 보수적으로 처리:

같은 AXI ID로 같은 path를 타는 모든 Non-modifiable 트랜잭션의 순서를 보장하라 (안전 우선 규칙)

<br>

**Bufferable / Non-bufferable 섞여도?**

- 이 문장도 중요해:

    > *“applies between all Non-modifiable transactions, including between Non-bufferable and Bufferable”*
    > 

    즉,

    - AxCACHE[1]=0 (Non-modifiable) 이면
    - AxCACHE[0]=0/1 (bufferable 여부)와 상관없이
    
    ⇒  순서 보장 대상

<br>    

**누가 책임지나?** 

중간 컴포넌트(interconnect)가 응답을 생성한다면

그 컴포넌트가 순서 보장 책임자가 된다. 

즉,

- interconnect
- NoC
- bridge

같은 애들이:

- 내부에서 reorder / buffer / split 했다면 ⇒ 최종 응답 순서가 맞도록 책임져야 함

<br>

**Read ↔ Write 사이의 순서**

읽기 채널과 쓰기 채널은 독립되어 있음 

그래서 AXI는 이렇게 말해:

⇒ Read/Write 간 순서는 “자동 보장 아님”

한 방향(read/write)의 트랜잭션은 다른 방향의 이전 트랜잭션에 대한 응답을 받은 뒤에

다음 트랜잭션을 발행해야만 순서가 보장된다.

예시

- write A

- read B

    - write 응답(BVALID/BREADY) 받고 나서
    - read를 발행해야

    write → read 순서가 보장됨

    응답 기다리지 않고 read를 먼저 쏘면

    → 순서 보장 없음, 이건 소프트웨어(드라이버) 책임이야.

<br>

이 규칙이 없으면 생기는 버그

- MMIO write 순서 뒤집힘
- device state machine 깨짐
- interrupt ack → enable 순서 꼬임
- secure register race
    
    ⇒ 디버깅 지옥
    
<br>

그래서 AXI4는:

- Modifiable / Non-modifiable을 나누고
- Non-modifiable에는 강제 ordering 규칙을 둔 거야.

<br>
---
<br>
[AXI4 – Memory Type (AxCACHE)]({% post_url AXI/2026-03-04-axi4-memory-type-axcache %})