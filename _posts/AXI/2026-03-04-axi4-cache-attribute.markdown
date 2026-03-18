---
layout: post
title:  "AXI4 – Cache Attribute"
date:   2026-03-04 12:00:00 +0900
categories: AXI4 AMBA
show_on_home: false
---

<br>

### **Cacheable**

"Cacheable = 시스템이 이 트랜잭션을 저장, 재사용, 합치기, 재정렬 같은 최적화 대상으로 취급해도 된다”

- CPU의 L2/L3 캐시 만을 의미 하는게 아니라 “시스템 레벨의 모든 캐시/버퍼” 최적화를 통칭한다. 

    > “AxCACHE controls how a transaction progresses through the system and how any system-level caches handle the transaction
    > 

    AXI 관점에서 **cacheable=1”의 의미**

    **“** 이 트랜잭션은 시스템 ( CPU 캐시 + interconnect + memory controller)가 캐시/버퍼/병합/재정렬을 해도 된다”

<br>

**CPU L2/L3 Cache에서**

cacheable = 1 설정/동작 되는 것 : 

- 캐시 라인 할당 허용
- Read miss 시 캐시 line fill
- Write miss 시 write-allocate 가능
- Prefetch 가능
- 같은 주소 read 재사용 가능

Cacheable = 0

- 캐시 line 할당 금지
- Speculative read 금지
- 반드시 버스로 나감

<br>

**AXI Interconnect 에서**

Cacheable = 1 설정/동작 되는 것 :

- Write merge 허용
- Write combining buffer 사용
- 일부 reordering 허용
- Burst 재구성 허용
    
    ![image.png](/assets/posts_image/AXI/Cache%20Attribute/image.png)
    

Cacheable = 0

- Merge 금지
- Reordering 금지
- 그대로 통과(pass-throhgh)

<br>

**Memory Controller(DDR)에서**

Cacheable = 1 설정/동작 되는 것 :

- Write queue 사용
- Command reordering
- Open-page 최적화
- Read prefetch

Cacheable = 0

- Write queue 최소화
- 순서 보장
- 즉시 커밋 성격

<br>

**Snoop / Coherency Logic 에서**

Cacheable = 1 설정/동작 되는 것 :

- 캐시 간 데이터 공유
- Snoop 응답 가능
- Read sharing

Cacheable = 0

- Snoop 대상 아님

---

### Buffering

- Bufferable = 1

    “ 이 트랜잭션을 중간에 잠시 보관 했다가 나중에 보내도 된다”

    - Bufferable은 “저장”을 의미

    - Cacheable은 “재사용”을 의미

    **Interconnect 에서**

    - bufferable=1 설정/동작 되는 것 :

        - write buffer에 쌓아도 됨
        - 나중에 슬레이브로 전달 가능
        - 반드시 merge를 의미하진 않음

    - bufferable=0 이면

        - write buffer 사용 금지
        - 즉시 전달

    **Memory Controller 에서**

    - bufferable=1

        - write queue 적재 허용
        - DRAM 타이밍 맞춰 나중에 write

    - bufferable=0이면

        - 바로 issue
        - flush 성격

    **CPU 캐시에서는 bufferable 개념이 거의(?) 없음**

        - CPU 캐시는 cacheable 속성만 중요
        - bufferable은 주로 interconnect / memory controller용

<br>

Cacheable / Bufferable 차이 요약

| 속성 | 의미 |
| --- | --- |
| cacheable | 저장 + 재사용 + 병합 + 재정렬 허용 |
| bufferable  | 중간에 잠시 저장(지연) 허용 |
| cacheable = 0 | 절대 건드리지 마라 |
| bufferable = 0 | 뭐 하지 말고 바로 보내라 |

---

### write/Read - allocate

Read-allocate / Write-allocate :

"cacheable 접근 중에서도 미스(miss)가 났을 때 이 트랜잭션을 계기로 캐시 라인을 만들 가치가 있느냐” 를 CPU캐시, interconnect, 메모리 컨트롤러에게 동시에 알려주는 힌트 같은 것

- RA=1 : read miss 시 캐시 라인 할당 *권장*
- WA=1 : write miss 시 캐시 라인 할당 *권장*
- 전제 : Cacheable(C)=1일 때만 의미 있음

```smalltalk
💡

"캐시 라인을 만든다" 

:  miss가 발생했을 때 메인 메모리에서 캐시 라인 크기만큼 데이터를 가져와 캐시 메모리에 저장해 둔다는 뜻

캐시 miss 났을 때 실제로 벌어지는 일 (read 기준)

상황 : 

캐시 라인 크기 : 64Byte

CPU가 주소 0x1008의 내용을 읽음

캐시에 없음 → read miss 발생

이 때 **Read-allocate = 1 인 경우**

1. 메모리에서 :
    
    0x1000 ~ 0x103F (64Byte)를 통째로 가져옴
    
2. 이 64Byte를 캐시 메모리에 저장
3. 필요한 값(0x1008)을 CPU에 전달
4. 이후 같은 라인 접근은 캐시 hit

이걸 “ 캐시 라인 할당(allocate)했다” or “캐시 라인 만들었다” 라고 함

**Read-allocate = 0 인 경우**

1. 메모리에서 필요한 데이터만 가져옴
2. CPU에 전달 
3. 캐시에 저장하지 않음

다음에 또 읽으면 또 miss됨

streaming / MMIO / 일회성 접근 일 때 사용

⇒ “캐시 라인을 만든다” == “주변 데이터까지 포함해서 한 덩어리로 저장한다” 

```

<br>

**시스템 관점에서 Read-allocate / Write-allocate가 의미하는 것?**

cacheable(C) = 1 이면,

- “이 접근은 시스템이 캐시/버퍼/병합을 해도 된다”

C = 1인 상태에서 RA / WA는 

- “그 중에서도 캐시 라인을 실제로 만들어서 들고 있어도 되냐”를 구체화 

- C = 캐시 사용 *가능*
- RA/WA = 캐시 라인 *할당 전략 힌트*

<br>

**CPU L2/L3 캐시 관점 (가장 직접적)**

- Read-allocate (RA)

    RA = 1

    - read miss 발생
    - 캐시 라인 할당
    - 메모리에서 한 줄(예: 64B) 가져와 캐시에 저장
    - 이후 read는 캐시 히트

    ⇒ 일반 데이터, 코드 fetch

    RA = 0

    - read miss 발생
    - 캐시에 저장 안 함
    - 값만 가져와 CPU로 전달

    ⇒ 스트리밍 read, MMIO 같은 경우

- Write-allocate (WA)

    WA = 1

    - write miss 발생
    - 캐시 라인 먼저 할당
    - 그 라인에 write 수행
    - 이후 write-back

    ⇒ write-back 캐시의 일반 동작

    WA = 0

    - write miss 발생
    - 캐시에 안 넣고 바로 메모리로 write
    - (write-through / no-allocate)

    ⇒ streaming write, DMA buffer

<br>

**AXI Interconnect 관점 (캐시 대신 fabric 최적화)**

- Interconnect에는 CPU 캐시처럼 라인 개념은 없지만,

    RA/WA는 “이 트랜잭션이 캐시 친화적이냐”를 알려주는 힌트로 쓰임

    RA / WA = 1 일 때

    - interconnect는:
        - read prefetch 가능
        - write combine을 더 공격적으로 해도 됨
        - burst 재구성 허용
    

    RA / WA = 0 일 때

    - interconnect는:
        - 요청을 단발성으로 취급
        - 불필요한 재시도/프리페치 안 함

    ⇒ fabric 수준에서 캐시 친화성 힌트

<br>

**MemoryController (DDR controller) 관점**

DDR 컨트롤러도 “캐시 라인”은 없지만, RA/WA는 접근 패턴 힌트로 쓰임.

RA = 1

- read miss는:
    - 연속 접근 가능성 높음
    - read ahead / prefetch 가능
    - open-page 유지 전략 유리

WA = 1

- write miss는:
    - 이후 read 가능성 있음
    - write buffer 유지
    - write combine 적극 활용

RA/WA = 0

- 단발성 접근
- 빠른 커밋, 순서 보장 쪽으로 동작

```smalltalk
💡

**실제로 자주 쓰는 조합 (이게 실무 핵심)**

일반 DDR 메모리

C=1, RA=1, WA=1

AxCACHE = 1111

- CPU 캐시 라인 할당
- interconnect/DDR 모두 최적화

Read-only 데이터 / 코드

C=1, RA=1, WA=0

- 읽을 때만 캐시
- write allocate 안 함

Streaming write buffer (DMA TX) 

C=1, RA=0, WA=0

- 캐시 저장 안 함
- 하지만 merge/buffer는 허용

MMIO

C=0 → RA/WA 무의미

AxCACHE = 0000

- 캐시/버퍼/프리페치 전부 금지
```