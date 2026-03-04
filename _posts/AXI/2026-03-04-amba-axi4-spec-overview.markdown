---
layout: post
title:  "AMBA&AXI4 Overview"
date:   2026-03-02 12:00:00 +0900
categories: AXI4 AMBA
show_on_home: false
---

AXI4 is a high‑performance AMBA on‑chip bus protocol that separates address/control and data phases, allows multiple outstanding and out‑of‑order transactions, and is optimized for high‑bandwidth, low‑latency access to shared memories in modern SoCs.

# Overview

## 1. 왜 만들어졌냐?

기존에 쓰던 **AMBA AHB/APB**의 기본 개념은 아래와 같다

```smalltalk
한 트랜잭션 = [주소 + 제어] -> [데이터] -> [응답]
// in-order, single outstanding transaction 이라고 한다.
```

이렇게 하나의 트랜잭션이 끝나야지 다음 트랜잭션을 진행하는 방식으로 전송한다.

⇒ 앞 요청이 끝나기 전까지 다음 요청을 못 보내도록 구조적으로 고정되어 있다.

예) AHB로 메모리 읽기

CPU가 메모리에서 4개의 데이터를 읽고 싶다면 아래와 같이 진행된다. 

```smalltalk
Cycle 1: 주소 A
Cycle 2: 데이터 A + 응답
Cycle 3: 주소 B
Cycle 4: 데이터 B + 응답
Cycle 5: 주소 C
Cycle 6: 데이터 C + 응답
...
```

그러나 SoC의 모듈들은 대부분 아래와 같은 구조로 연결 되어 있기 때문에  APB/AHB 방식을 계속 사용하게 되면 문제가 발생한다.

CPU  ─┐
DMA  ─┼─> Memory
GPU  ─┘

만일 이런 구조에서 AHB는,

“CPU 에러 read 시작 → 버스에서는 지금 트랜잭션 처리 중 → DMA 대기, GPU 대기 “

이런 현상들이 지속되면 버스가 병목(Bottleneck)이 걸리고 만다. 

<aside>
💡

**파이프라인**

“줄 서서 한 명씩” vs “컨베이어 벨트”

1. 파이프라인 없는 방식 (AHB)
    
    사람 A: 접수 → 처리 → 끝
    사람 B: 접수 → 처리 → 끝
    사람 C: 접수 → 처리 → 끝
    
    A가 끝나야 B 시작
    
    B가 끝나야 C 시작
    
    이게 AHB, APB 방식 ⇒ 한 트랜잭션이 끝나야 다음 가능
    
2. 파이프라인 방식 (AXI)
    
    컨베이어 벨트가 있다고 해보자
    
    [접수] → [처리] → [완료]
    
    시간 흐름
    
    시간1: A 접수
    시간2: A 처리 | B 접수
    시간3: A 완료 | B 처리 | C 접수
    시간4: B 완료 | C 처리
    
    A가 아직 처리 중인데, B는 이미 접수 중이고, C도 들어오고 있음 
    
</aside>

AXI 구조 (진짜 채널 분리)

AXI는 주소, 데이터, 응답을 분리해서 여러 거래를 동시에, 순서에 얽매이지 않고 처리하는 고성능 버스

AXI는 처음부터 채널을 독립적으로 사용

> “AXI allows multiple outstanding transactions.”
> 

```smalltalk
주소 A 전송
주소 B 전송
주소 C 전송
(아직 A의 응답이 안와도 괜찮음)
```

```smalltalk
데이터 B 먼저
데이터 A 나중
(A 데이터가 나중에 나가도 허용한다)
```

```smalltalk
CPU: Read A  → AR channel
DMA: Read B  → AR channel
GPU: Write C → AW channel

Memory controller:
"아, B가 캐시라서 빠르네 → B 먼저 처리"

이게 동시에 돌아간다.
```

IoT, 모바일, DSP, 네트워크 SoC, GPU등 아래와 같은 환경에서 AXI가 필요한다. 

메모리 많이 씀

동시에 여러 엔진이 접근

지연 1번이 전체 성능에 영향을 줌

AXI 내부는 이렇게 겹쳐짐

[주소 전송][주소 전송][주소 전송]

             [데이터 전송][데이터 전송][데이터 전송]

⇒ 이 결과 파이프라인 효율을 올릴 수 있음 (버스 놀지 않음, 클럭 여유 생김, 고주파 설계가능)

---

예 2) 레지스터 초기화 예제 상황

CPU또는 DMAC (AXI Master)

GPIO (AXI Slave)

초기화 해야 할 레지스터  

```
	REG_A : 0x4000_0000 ← 0x0000_0001
	REG_B : 0x4000_0004 ← 0x0000_00FF
	REG_C : 0x4000_0008 ← 0x1234_5678
```

AHB 방식

1. 주소 0x4000_0000 + 데이터 0x1
    
    → 완료 될 때까지 대기
    
2. 주소 0x4000_0004 + 데이터 0xFF
    
    → 완료 될 때까지 대기
    
3. 주소 0x4000_0008 + 데이터 0x1234_5678
    
    → 완료 될 때까지 대기
    

⇒ 매 write 마다 slave 반응 기다림 

파이프라인 안됨

Clock을 올릴 수 없음

AXI 방식

1단계 : 주소만 연속으로 던짐 (AW 채널)

```smalltalk
t0 : AWADDR = 0x4000_0000 AWVALID = 1
t1 : AWADDR = 0x4000_0004 AWVALID = 1
t2 : AWADDR = 0x4000_0008 AWVALID = 1
```

⇒ 이 시점에 데이터는 하나도 안보냄

GPIO(Slave)는 :

“아, 곧 이 주소들에 write가 들어오겠구나” 라고 내부 큐에 주소 저장

2단계 : 데이터는 준비되는 대로 보냄 (W 채널)

```smalltalk
t3 : WDATA = 0x0000_0001
t4 : WDATA = 0x0000_00FF
t5 : WDATA = 0x1234_5678
```

GPIO(Slave)는:

앞에서 받은 주소 큐에서 하나 꺼내고, 들어온 데이터랑 매칭해서 write 처리 수행

3단계 : 응답은 나중에 몰아서 와도 됨 (B 채널)

```smalltalk
t6 : BRESP OKAY
t7 : BRESP OKAY
t8 : BRESP OKAY
```

⇒ 요청된 순서가 완료된 순서와 같지 않을 수 있음

“데이터는 준비되는 대로”의 진짜 의미

 ⇒ 주소 전송이 끝난 시점에 데이터가 아직 준비 안 돼 있어도 된다

왜 이런 상황이 생기냐면

- CPU:
    - 레지스터 값 계산 중
    - ISR에서 값 만들어야 함
- DMA:
    - 메모리에서 데이터 읽어오는 중
- 버스:
    - arbitration 중

주소는 이미 확정되어 있으니, 데이터는 뒤에서 천천히 와도 OK

AXI 프로토콜은 다음과 같은 설계에 적합하다

- 높은 대역폭(high-bandwidth) 과 낮은 지연(latency) 이 필요한 설계
- 복잡한 브리지 없이도 고주파 동작이 가능한 구조
- 매우 다양한 컴포넌트들의 인터페이스 요구사항을 만족
- 초기 접근 지연(latency)이 큰 메모리 컨트롤러에 적합
- 인터커넥트 아키텍처 구현에 유연성을 제공
- 기존 AHB, APB 인터페이스와 하위 호환(backward-compatible)

🔹 AXI 프로토콜의 핵심 기능 (Key Features)

AXI 프로토콜의 주요 특징은 다음과 같다.

- 주소/제어 단계와 데이터 단계를 분리
- 바이트 스트로브(byte strobe) 를 사용해 비정렬(unaligned) 데이터 전송 지원
- 버스트 기반 트랜잭션 사용 → 시작 주소만 전달
- 읽기/쓰기 데이터 채널 분리 → 저비용 DMA 구현 가능
- 여러 개의 주소 트랜잭션을 동시에 발행 가능
- 트랜잭션 완료 순서가 요청 순서와 달라도 허용
- 타이밍 클로저를 위해 레지스터 단을 쉽게 추가 가능

[AXI Architecture](AMBA%20AXI4%20Spec/AXI%20Architecture%202e66feb16a3e80dbb036c1cb2d1ca736.md)

[Channel Signaling Requirements](AMBA%20AXI4%20Spec/Channel%20Signaling%20Requirements%202e66feb16a3e806bb15ae33d0834bc39.md)

[[SIGNAL] Global Signals](AMBA%20AXI4%20Spec/%5BSIGNAL%5D%20Global%20Signals%202e66feb16a3e800f9d2fd4d74e9ff556.md)

[[SIGNAL] AVALID, AREADY  & Handshake](AMBA%20AXI4%20Spec/%5BSIGNAL%5D%20AVALID,%20AREADY%20&%20Handshake%202e66feb16a3e80f49f76df106fd4274d.md)

[[SIGNAL] BURST & LEN & SIZE](AMBA%20AXI4%20Spec/%5BSIGNAL%5D%20BURST%20&%20LEN%20&%20SIZE%202e66feb16a3e809eb261f31d610e094d.md)

[[SIGNAL] PROT](AMBA%20AXI4%20Spec/%5BSIGNAL%5D%20PROT%202e66feb16a3e808bab0febc95a6fb980.md)

[[SIGNAL] CACHE](AMBA%20AXI4%20Spec/%5BSIGNAL%5D%20CACHE%202e66feb16a3e8084b3b4e27bfed1cbd0.md)