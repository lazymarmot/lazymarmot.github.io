---
layout: post
title:  "AXI4 Architecture"
date:   2026-03-04 12:00:00 +0900
categories: AXI4 AMBA
show_on_home: false
---

AXI ArchitectureAXI4 defines a burst‑based on‑chip interconnect where read and write traffic is split into independent address, data, and response channels, each using a VALID/READY handshake so masters and slaves can transfer data concurrently with high bandwidth and predictable control over transaction ordering.

# AXI Architecture

AXI 프로토콜은 버스트 기반(burst-based)이며, 아래와 같이 서로 독립적인 트랜잭션 채널들을 정의하고

각 채널마다 필요한 신호들이 M ↔ S사이에 연결되어 있다.

- 읽기 주소 채널 (read address)
- 읽기 데이터 채널 (read data)
- 쓰기 주소 채널 (write address)
- 쓰기 데이터 채널 (write data)
- 쓰기 응답 채널 (write response)

**데이터 전송 방식**

데이터는 master 와 slave 사이에서 다음 두 방식 중 하나로 전달된다.

![image.png](/assets/posts_image/AXI/AXI%20Architecture/image.png)

**Figure A1-1: 읽기 트랜잭션의 채널 아키텍처**

- 읽기 데이터 채널(read data channel)
    
     : slave → master 방향으로 데이터 전달
    
- Master interface와 Slave interface가 두 개의 채널로 연결
    - **Read address channel** (읽기 주소 채널)
        - Master → Slave 방향
        - Address and control 정보 전송
    - **Read data channel** (읽기 데이터 채널)
        - Slave → Master 방향
        - 여러 개의 Read data 전송 (burst)

**Figure A1-2: 쓰기 트랜잭션의 채널 아키텍처**

- 쓰기 데이터 채널(write data channel)
    
     :  master → slave 방향으로 데이터 전달
    
- 쓰기 응답 채널(write response channel)
    
     : slave 가 master 에게 “쓰기 끝났다” 라고 완료 신호를 보냄
    
- Master interface와 Slave interface가 세 개의 채널로 연결
    - **Write address channel** (쓰기 주소 채널)
        - Master → Slave 방향
        - Address and control 정보 전송
    - **Write data channel** (쓰기 데이터 채널)
        - Master → Slave 방향
        - 여러 개의 Write data 전송 (burst)
    - **Write response channel** (쓰기 응답 채널)
        - Slave → Master 방향
        - Write response 전송

**Channel Definition**

채널 정의 

각각의 독립적인 채널들은 “여러 정보 신호(information signals)”와 VALID, READY 신호로 구성되며

이들은 “양방향 핸드셰이트 매커니즘(two-way handshake)”을 제공한다.

기본 읽기/쓰기 트랜잭션은 AXI spec 데이터 시트의 A3-39페이지 참고

정보를 보내는 쪽(source)는 채널에 유효한 주소, 데이터, 제어 정보가 존재함을 나타내기 위해 VALID 신호를 사용한다

정보를 받는 쪽(destination)은 READY 신호를 사용해서 정보를 받을 준비가 되었음을 표시한다. 

읽기 데이터 채널, 쓰기 데이터 채널 모두에는 트랜잭션에 마지막 데이터 전송임을 나타내는 LAST 신호가 포함된다. 

1. Read / Write Address Channel
    
    읽기 / 쓰기 “주소 채널”
    
    읽기 / 쓰기 트랜잭션은 각각 자신만의 주소 채널을 갖게 되며, 전송될 데이터의 성격을 설명하는 제어 정보를 주로 전달한다. 
    
    예:
    
    - 어디 주소인지
    - 몇 바이트인지
    - burst 길이
    - burst 타입 등
    1. 읽기 주소 채널(Read Address Channel, AR)
        
        “이 주소를 읽겠다” 라고 요청하는 채널
        
        방향 : Master → Slave
        
        | ARADDR | 읽을 주소 |
        | --- | --- |
        | ARVALID | 주소/제어 정보 유효 |
        | ARREADY | Slave가 받을 준비 됨 |
        | ARLEN | burst 길이 (AXI만, Lite는 없음) |
        | ARSIZE | beat 크기 (AXI만, Lite는 없음) |
        | ARBURST | burst 타입 (AXI만, Lite는 없음) |
        | ARLOCK | exclusive/normal (AXI만, Lite는 없음) |
        | ARCHACHE | cache 속성 (AXI만, Lite는 없음) |
        | ARPORT | 보호 속성 (권한/보안) |
        | ARID | 트랜잭션 ID (AXI만, Lite는 없음) |
    2. 쓰기 주소 채널(Write Address Channel, AW)
        
        “이 주소에 쓰겠다” 라고 예약하는 채널
        
        방향 : Master → Slave
        
        | AWADDR | 쓸 주소 |
        | --- | --- |
        | AWVALID | 주소/제어 정보 유효 |
        | AWREADY | Slave가 받을 준비 됨 |
        | AWLEN | burst 길이 (AXI만, Lite는 없음) |
        | AWSIZE | beat 크기 (AXI만, Lite는 없음) |
        | AWBURST | burst 타입 (AXI만, Lite는 없음) |
        | AWLOCK | exclusive/normal (AXI만, Lite는 없음) |
        | AWCHACHE | cache 속성 (AXI만, Lite는 없음) |
        | AWPORT | 보호 속성 (권한/보안) |
        | AWID | 트랜잭션 ID (AXI만, Lite는 없음) |
        
        AW 채널에는 데이터가 없고, 데이터는 나중에 W 채널에서 따로 씀
        
2. Read Data Channel
    
    읽기 “데이터 채널”
    
    읽기 데이터 채널은 Slave → Master 방향으로 읽은 데이터와 읽기 응답 정보를 함께 전달하며, 아래 내용을 포함한다. 
    
    - 데이터 버스
        - 폭: 8, 16, 32, 64, 128, 256, 512, 1024 비트
    - 읽기 응답 신호
        - 읽기 트랜잭션의 완료 상태를 나타냄
    
    | RDATA | 읽은 데이터 값 |
    | --- | --- |
    | RRESP | 읽기 응답(OKAY / SLVERR / DECERR) |
    | RVALID | 데이터 유효 |
    | RREADY | Master가 받을 준비됨 |
    | RLAST | burst의 마지막 beat |
    | RID | 트랜잭션 ID (AXI4만) |
3. Write Data Channel
    
    쓰기 “데이터 채널”
    
    쓰기 데이터 채널에는 이런 신호들이 있다. 
    
    | WDATA | 실제 쓰려는 데이터 값 |
    | --- | --- |
    | WSTRB | 어떤 바이트가 유효한지 표시 |
    | WVALID | 데이터 유효 |
    | WREADY | Slave가 받을 준비 됨 |
    | WLAST | burst의 마지막 beat |
    
    이렇게 쓰기 데이터 채널 한 묶음 신호들이다.
    
    WDATA랑 WSTRB는 서로 다른 신호선 이지만 같은 채널, 같은 핸드셰이크로 동작한다. 
    
    쓰기 데이터 채널은 Master → Slave 방향으로 쓰기 데이터를 전달하며, 다음을 포함한다. 
    
    - 데이터 버스
        - 폭: 8, 16, 32, 64, 128, 256, 512, 1024 비트
    - 바이트 레인 스트로브(byte lane strobe) 신호
        - 8비트(1바이트)마다 하나
        - 데이터 중 어떤 바이트가 유효한지 표시
        
    
    쓰기 데이터 채널의 정보는 항상 버퍼링된 것으로 취급되며, master는 이전 write 트랜잭션에 대한 slave의 응답을 기다리지 않고 연속적으로 write 트랜잭션을 수행할 수 있다.
    

1. Write Response Channel
    
    쓰기 “응답 채널”
    
    Slave는 쓰기 응답 채널을 사용해서 쓰기 트랜잭션에 대한 응답을 보낸다
    
    모든 write 트랜잭션은 쓰기 응답 채널을 통한 “완료 신호”를 반드시 요구한다. 
    
    | BRESP | 쓰기 응답 (OKAY / SLVERR / DECERR) |
    | --- | --- |
    | BVALID | 응답 유효 |
    | BREADY | Master가 받을 준비 됨 |
    | BID | 트랜잭션 ID |

**Interface and Interconnect**

![image.png](/assets/posts_image/AXI/AXI%20Architecture/image%201.png)

 ****

AXI에서 말하는 “인터페이스”는?

- 읽기 주소 채널 (AR)
- 쓰기 주소 채널 (AW)
- 읽기 데이터 채널 (R)
- 쓰기 데이터 채널 (W)
- 쓰기 응답 채널 (B)

그리고 각 채널의:

- 신호 이름
- 방향
- VALID / READY 규칙
- 타이밍

👉 이 전체 규칙 묶음이 “AXI 인터페이스”

“master - interconnect, interconnect - slave, master - slave 누구랑 붙든지 AXI Interface는 신호 모양은 똑같음”

경우 1 : Master ↔ Slave (직결)

CPU (master) —— AXI ——- GPIO (slave)

여기서:

CPU는 AXI master 인터페이스

GPIO는 AXI slave 인터페이스

```
신호 예:
	CPU.ARADDR  ─────────▶ GPIO.ARADDR
	GPIO.RDATA  ─────────▶ CPU.RDATA
```

경우 2 : Master ↔ interconnect ↔ Slave

CPU —— AXI —— Interconnect —— AXI —— GPIO

interconnect는 AXI인터페이스를 2개 갖고 있다 ( 역할만 다를 뿐 신호 규칙은 둘다 AXI 인터페이스 규칙임)

1. CPU 쪽으로 향한 인터페이스 (slave 처럼 행동)
2. GPIO 쪽을 향한 인터페이스 (master 처럼 행동)

```smalltalk
	CPU → Interconnect (쓰기 예)
	
		CPU.AWADDR  ─▶ Interconnect.AWADDRCPU.WDATA   ─▶ Interconnect.WDATA
		👉 이때 인터커넥트는 AWREADY / WREADY 내주는 쪽
		→ slave 역할

	Interconnect → GPIO
	
		Interconnect.AWADDR ─▶ GPIO.AWADDRInterconnect.WDATA  ─▶ GPIO.WDATA
		👉 이때 인터커넥트는 AWVALID / WVALID 내보내는 쪽
    → master 역할
```

전제: 시스템에 "주소 맵"이 이미 정해져 있음

SoC 설계할 때 메모리 맵(memory map)을 이렇게 정해냄.

예를 들어:

**Slave별 주소 범위**

- **Slave1 (GPIO)**: 0x4000_0000 ~ 0x4000_0FFF
- **Slave2 (UART)**: 0x4001_0000 ~ 0x4001_0FFF
- **Slave3 (I2C)**: 0x4002_0000 ~ 0x4002_0FFF
- **Slave4 (DDR ctrl)**: 0x8000_0000 ~ 0x8FFF_FFFF

이건 HW 설계 시점에 고정

CPU, DMA, 인터커넥트 모두 동일하게 알고 있음

Master1 (예:CPU)에서 Slave4 레지스터에 write 하고 싶다면?

Master1: “0x8000_0100에 쓰자”

```smalltalk
주소 = 0x8000_0100
데이터 = 0x1234_5678
```

1. Master1이 AXI로 보내는것
    
    Write Address Channel (AW)
    
    ```smalltalk
    AWADDR = 0x8000_0100
    AWVALID = 1
    ```
    
    Write Data Channel(W)
    
    ```smalltalk
    WDATA = 0x1234_5678
    WSTRB = 111
    ```
    
2. Interconnect가 하는 일
    
    인터커넥트 내부에는 주소 디코더가 있음
    

```smalltalk
if (AWADDR in 0x4000_000 ~ 0x4000_0FFF) -> Slave1
else if (AWADDR in 0x4001_000 ~ 0x4001_0FFF) -> Slave2
else if (AWADDR in 0x4002_000 ~ 0x4002_0FFF) -> Slave3
else if (AWADDR in 0x8000_000 ~ 0x8FFF_FFFF) -> Slave4
-> 0x8000_0100 -> Slave4 선택
```

c. Interconnect가 Slave4에게 전달

Slave4 입장에서는:

인터커넥트가 master처럼 보임

Master1 입장에서는:

인터커넥트가 slave처럼 보임

```smalltalk
Interconnect -> Slave4
	AWADDR = 0x0100 
	WDATA = 0x1234_5678
```

d. Slave4가 write 수행

```smalltalk
if (AWADDR == 0x0100)
	REG_X <= WDATA;
	
나중에
BRESP = OKAY
```

**Register Slices**

![image.png](/assets/posts_image/AXI/AXI%20Architecture/image%202.png)

각 AXI 채널은 오직 한 방향으로만 정보를 전달하며, 이 아키텍처는 채널들 사이에 어떤 고정된 관계도 요구하지 않는다.

이는, 추가적인 한 사이클의 지연(latency) 이 발생하는 대가로, 어느 채널의 거의 어느 위치에든 레지스터 슬라이스(register slice)를 삽입할 수 있음을 의미한다.

Register slice란:

- “AXI 신호를 중간에서 한 클럭 저장했다가 다음 클럭에 보내는 버퍼”
- AXI는 채널들이 서로 독립적이고, 단 방향이기 때문에 필요한 곳마다 레지스터를 끼워 넣어 고클럭 설계를 할 수 있게 만든 구조이다
- 채널들이 서로 묶여있지 않기 때문에 중간에 레지스터를 끼워도 프로토콜이 안 깨짐
    
    
    CPU → Interconnect → GPIO (레지스터 슬라이스 없음)
    
    이 배선이 길고 IP가 많으면
    
    ```smalltalk
    CPU 출력
     → 3cm 배선
     → Interconnect 로직
     → 4cm 배선
     → GPIO 입력
    ```
    
    한 Clock 안에 이 신호가 도착해야 함
    
    Clock이 200MHz명:
    
    1 Clock = 5ns
    
    이 5ns 안에 다 통과해야 함 → 빡셈
    
    CPU → [REG] → Interconnect → [REG] → GPIO (레지스터 슬라이스 있음)
    
    ```smalltalk
    1클럭: CPU → REG
    2클럭: REG → Interconnect
    3클럭: Interconnect → REG → GPIO
    ```
    
    한 단계가 짧아짐 → 고클럭 가능
    

실제 예시 (Zynq AXI GPIO)

```smalltalk
PS ──AXI── Interconnect ──AXI── GPIO
```

이게 150MHz에서 안 돌아간다면

Vivado에서 “AXI Register Slice”를 켬

```smalltalk
PS ──AXI── [REG] ──AXI── Interconnect ──AXI── [REG] ──AXI── GPIO
자동 삽입됨
GPIO가 느려지지 않고 Clock만 1~2개 늦어짐
```

<aside>
💡

**짧게 쪼개서 전송하면 고클럭으로 보낼 수 있다?의 의미**

> “신호가 한 클럭에 가야 할 물리적 거리를 짧게 쪼갠다”
> 

CPU에서 GPIO까지 가는 길이

CPU → 스위치 1 → 배선 → 스위치 2 → 배선 → Interconnect → 배선 → GPIO

이걸 통과하는데 시간이 걸린다.

**클럭이 빠르다는건?**

200MHz → 1 Clock = 5ns

300MHz → 1 Clock = 3.3ns

> 한 Clock안에 신호가 도착해야 함
> 

근데 CPU → GPIO가 6ns이 걸리면

200MHz : OK

300MHz : 3,3ns안에 못 도착함

그래서 고클럭이 안됨

**짧게 쪼갠다는건?**

레지스터 슬라이스 넣으면:

CPU → [REG] → Interconnect → [REG] → GPIO

이렇게 나뉘고, 이제 한번에 가야 하는 거리가

CPU → REG
REG → Interconnect
Interconnect → GPIO

각각 보면 짧아짐

| 구간 | 걸리는 시간 |
| --- | --- |
| CPU → GPIO (한 번에) | 6ns |
| CPU → REG | 2ns |
| REG → Interconnect | 2ns |
| Interconnect → GPIO | 2ns |

→ 이렇게 하면 3ns Clock도 사용할 수 있어지게 됨

</aside>

각각의 채널들은 단방향임

AW, W, AR : master → slave

R, B : slave → master

채널 간 고정 관계가 없음

AW가 언제 오든, W가 언제 오든, B가 언제 오든 서로 강제 타이밍 관계가 없음

⇒ 채널 간 독립

그래서 Register slice를 어디든 넣을 수 있음

**예 1️⃣: AW 채널만 레지스터 슬라이스 삽입**

CPU ── AW ── [REG] ──
Interconnect

- 주소 전달만 1클럭 늦어짐
- W, B 채널은 영향 없음

**예 2️⃣: R 채널에만 삽입**

Slave ── R ── [REG] ── CPU

- 읽기 데이터만 1클럭 늦음
- AR 채널은 그대로

**❗ 왜
“거의 any point”일까?**

완전히 아무
데나 ❌ 인 이유:

- VALID/READY 규칙을 깨면 안 됨
- combinational loop 금지

하지만:

- **채널 내부에서는 자유롭게 가능**