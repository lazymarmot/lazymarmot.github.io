---
layout: post
title:  "AXI4 SIGNAL – BURST, LEN & SIZE"
date:   2026-03-04 12:00:00 +0900
categories: AXI4 AMBA
show_on_home: false
---

# [SIGNAL] BURST & LEN & SIZE

**Burst length**

Burst length는 하나의 Burst 안에 포함된 전송(transfer)의 정확한 개수를 나타낸다.

이 정보는 해당 주소(Address)에 연결된 데이터 전송 횟수를 결정한다. 

데이터가 몇 번(몇 beat) 오가느냐?

주소는 한번만 보내고 데이터는 burst length만큼 반복 전송

Burst Length (버스트 길이) ⇒ AxLEN 신호로 결정

읽기 : AREN[7:0]

쓰기 : AWEN[7:0]

AXI3에서 버스트 길이는 다음과 같이 정의 된다

Burst_Length = AxLEN[3:0] + 1

AXI4에서 버스트 길이는 다음과 같이 정의 된다

Burst_Length = AxLEN[3:0] + 1

이는 AXI4의 INCR 타입의 버스트에서 확장된 버스트 길이를 수용하기 위함이다.

1. AxLEN 값의 의미 (공통)
    
    실제 전송 횟수 = AxLEN + 1
    
    - AxLEN = 0 → 1 beat
    - AxLEN = 3 → 4 beat
    - AxLEN = 15 → 16 beat
    - (AXI4 INCR) AxLEN = 255 → 256 beat
    
2. AXI3 vs AXI4 차이 (핵심 차이점)
    - AXI3 : 최대 16 beat
    - AXI4 :
        - INCR만 최대 256 beat
        - FIXED / WRAP은 여전히 16 beat
    
    ⇒ 대용량 메모리 전송(DMA, DDR)에 AXI4가 유리한 이유
    
3. WRAP 버스트 규칙
    - WRAP은 주소가 원형으로 돌아야 해서
    - 길이를 2, 4, 8, 16으로만 제한
    
4. 4KB 경계 규칙 (중요)
    
    하나의 버스트는 4KB boundary를 넘으면 안 됨
    
    예:
    
    - 시작 주소: 0x0FFC
    - 4-byte / per beat 를
    - 4 beat 버스트 전송 한다면?
        - 전송 에러 발생(4KB 경계 넘음)
    
    ⇒ 인터커넥트/메모리 단순화 목적
    
    주소 범위가 어떻게 나뉠까?
    
    1KB = 1024 byte
    
    4KB = 4 * 1024 = 4096 byte ( 0x1000 byte )
    
    0x0000 ~ 0x0FFF -> 4KB 
    
    0x1000 ~ 0x1FFF -> 다음 4KB
    
    (주소 하위 12비트가 변하는 구간이 4KB)
    
    <aside>
    💡
    
    **Page 개념부터 정리해보자**
    
    메모리는 이렇게 나뉘어져 있다
    
    1 Page = 4KB = 4096 byte
    
    주소 기준 : 
    
    페이지 0 : 0x0000 ~ 0x0FFF
    
    페이지 1 : 0x1000 ~ 0x1FFF
    
    페이지 2 : 0x2000 ~ 0x2FFF
    
    </aside>
    
    4KB boundary란?
    
    ```
    	... 0x0FFC
    	... 0x0FFD
    	... 0x0FFE
    	... 0x0FFF   ← 4KB의 끝
    	... 0x1000   ← ❗다음 4KB 시작 (boundary 넘음)
    ```
    
    0x0FFF -> 0x1000 이 순간이 "4KB 바운더리를 넘는다"
    

왜 하필 4KB일까?

CPU/OS/메모리 설계 전체의 공통 단위

1. Page 크기
    
    대부분 시스템에서 메모리 page는 4KB
    
    MMU, page table, 캐시, 보호 단위 전부 4KB 기준
    
    0x0000 ~ 0x0FFF는 한 페이지
    
    0x1000부터는 다른 페이지
    
2. 인터커넥트/메모리 컨트롤러 단순화
    
    AXI 설계자 입장에서:
    
    “한 burst가 두 페이지를 동시에 건드리면 너무 귀찮다…”
    
    왜냐면:
    
    - 페이지 A → 권한 OK
    - 페이지 B → 권한 NG 일 수도 있음
    - 주소 변환 두 번 필요
    - TLB miss 가능성 증가
    
    그래서:
    
    “burst는 한 페이지(4KB) 안에서만”
    

예시 조건

- Data width = 32bit
    
    ⇒ 1 beat = 4 byte
    
- Burst length = 4 beat
    
    ⇒ 총 전송 데이터 = 16 byte
    
- Burst type = INCR
- 시작 주소 = 0x0FFC

AXI 규칙대로 계산한 “실제 주소 진행”

AXI에서 주소는 byte address이고, INCR Burst에서는 beat마다 주소가 +4야

| beat | 시작주소 | 쓰여지는 바이트 | 페이지 |
| --- | --- | --- | --- |
| 1 | 0x0FFC | 0x0FFC ~ 0x0FFF | 페이지0 |
| 2 | 0x1000 | 0x1000 ~ 0x1003 | 페이지1 ❌ |
| 3 | 0x1004 | 0x1004 ~ 0x1007 | 페이지1 |
| 4 | 0x1008 | 0x1008 ~ 0x100B | 페이지1 |

⇒ 1개의 burst가 두 페이지를 침범

이런 식으로 데이터가 오면 페이지에 어떻게 저장 되느냐?

⇒ AXI 관점에서

“이 burst는 애초에 허용되지 않고, 슬레이브/인터커넥트는 이런 burst를 받으면”

- DECERR / SLVERR 응답 가능
- 또는 트랜잭션 자체를 생성하지 않음

실제 시스템에서는 어떻게 하냐?

CPU / DMA가 반드시 이렇게 쪼갠다

- 같은 16byte를 보내고 싶어도, burst를 나눔
    
    burst 1
    
    - 시작 : 0x0FFC
    - length : 1 beat
    - 쓰여지는 곳 : 페이지 0의 마지막 4byte
    
    burst 2
    
    - 시작 : 0x1000
    - length : 3 beat
    - 쓰여지는 곳 : 페이지 1의 처음 12byte
    
    ```verilog
    [burst #1]
    0x0FFC ~ 0x0FFF (페이지 0)
    
    [burst #2]
    0x1000 ~ 0x100b (페이지 1)
    ```
    
    0x0FFC에서 4-beat burst는 페이지에 ‘어떻게 저장되느냐’ 를 고민할 대상이 아니라, AXI 규칙 상 애초에 생성되면 안되는 burst이다. 
    
    그래서 실제 시스템에서는 반드시 페이지 경계에서 burst를 쪼개서 보낸다.
    
    **GPT가 말해준 예시**
    DMA가 실제로 AXI 버스트를 몇 번 날리는지?
    
    (전제 조건)
    
    - 주소 범위: 0x0000 ~ 0x1FFF (총 8KB, 2page)
    - boundary: **4KB = 0x1000**
    - bus width: **32bit →( 1 beat = 4 byte )**
    - 전송 크기: **4352 byte = 4096 + 256**
    - burst type: 보통 **INCR**
    - **(현실적인 가정)**
        
        DMA의 max_burst_beats = 256 (AXI4 INCR의 최대치, 흔히 DMA가 이 한도 내로 쏨)
        
        **1) “4KB boundary 때문에” 먼저 크게 2 덩어리로 잘림**
        
        시작 주소가 0x0000이면 4KB 경계는 0x1000이야.
        
        - chunk #1: 0x0000 ~ 0x0FFF → **4096 byte** (한 페이지)
        - chunk #2: 0x1000 ~ 0x10FF → **256 byte** (다음 페이지)
        
        여기까진 boundary 규칙 때문에 무조건 이렇게 나뉨.
        
        **2) 그런데 4096 byte는 한 번의 burst로 못 보냄 (1burst = 256 beats )**
        
        ![image.png](/assets/posts_image/AXI/SIGNAL%20BURST%20LEN%20SIZE/image.png)
        
        4096 byte를 4byte로 나눔 (bus width : 32bit 이니까)
        
        4096 / 4byte(per beat) = 1024 beats
        
        4096byte는 결국 4byte씩 1024번의 beat를 보내야 전체 전송 됨
        
        근데 DMA의 max_burst_beats가 256 beats 이니까 한번의 burst에서 보낼 수 있는 beat는 256개임
        
        1024 beat /  256 beat(per burst)  = 4 burst
        
        결국 4096byte를 보내기 위해서는 4번의 버스트를 통해 256beat 씩 보내야 함
        
    
    **3) 최종적으로 DMA가 날리는 버스트 목록 (쓰기 기준 AW/W)**
    
    **버스트 #1~#4: 첫 4Kbyte(4096byte)를 256 beat씩 4번**
    
    - 256 beats × 4byte = 1024byte = 0x400byte 씩 전송
        
        
        | **Burst** | **시작주소(AWADDR)** | **전송 바이트** | **beats** | **AWLEN (=beats-1)** | **마지막 주소** |
        | --- | --- | --- | --- | --- | --- |
        | 1 | 0x0000 | 1024byte | 256 | 255 (0xFF) | 0x03FF |
        | 2 | 0x0400 | 1024byte | 256 | 255 | 0x07FF |
        | 3 | 0x0800 | 1024byte | 256 | 255 | 0x0BFF |
        | 4 | 0x0C00 | 1024byte | 256 | 255 | 0x0FFF |
        
        여기까지가 딱 **첫 페이지(0x0000~0x0FFF)** 안에서 끝남 (4KB boundary 안 넘음)
        
    
    **버스트 #5: 남은 256byte를 다음 페이지에서 전송**
    
    - 256yte / 4byte = 64 beats
    - AWLEN = 63
        
        
        | **Burst** | **시작주소(AWADDR)** | **전송 바이트** | **beats** | **AWLEN** | **마지막 주소** |
        | --- | --- | --- | --- | --- | --- |
        | 5 | 0x1000 | 256byte | 64 | 63 (0x3F) | 0x10FF |
    
    👉 이것도 **페이지 2(0x1000~0x1FFF)** 안에서만 움직이니까 OK
    
1. 버스트 조기 종료 불가
    - WLAST/RLAST는 예정된 마지막 beat에서 만 가능
    - 중간에 “아, 취소” 이런 식으로 취소 못함
    
    예외처럼 보이는 경우 (하지만 예외 아님)
    
    - 쓰기: WSTRB를 전부 0으로 만들어도→ beat 수는 끝까지 진행
    - 읽기: 데이터를 버려도→ beat 수는 끝까지 진행
    
    ⇒ 트랜잭션 구조 자체는 반드시 완주
    
    예 1) AXI4 INCR 쓰기
    
    AWLEN = 7
    
    → Burst_Length = 8 beat
    
    → WLAST는 8번째 beat에서 만 1
    
    예 2) 쓰기 중 “실제로는 4번만 쓰고 싶을 때”
    
    - 1~4 beat: WSTRB 정상
    - 5~8 beat: WSTRB = 0
    - WLAST: 여전히 8번째 beat

---

---

**Burst Size (AxSIZE 신호)**

버스트 내에서 각 데이터 전송 (transfer), 즉 한 beat당 전송되는 최대 바이트 수를 지정하는 신호

읽기 : ARSIZE[2:0]

쓰기 : AWSIZE[2:0]

AXI 버스 폭이 버스트 크기보다 더 넓은 경우,

AXI 인터페이스는 **전송 주소를 기준으로 데이터 버스의 어떤 바이트 레인(byte lane)을 사용 할 지**를 결정해야 한다.

(자세한 내용은 A3-52 페이지의 *Data read and write structure* 참고)

![image.png](/assets/posts_image/AXI/SIGNAL%20BURST%20LEN%20SIZE/image%201.png)

1. AxSIZE는 “한 beat에 몇 바이트를 보내느냐”
    
    AxSIZE = log₂(한 beat당 바이트 수)
    
    예:
    
    - AxSIZE = 0 → 1 byte / beat
    - AxSIZE = 1 → 2 byte / beat
    - AxSIZE = 2 → 4 byte / beat
    - AxSIZE = 3 → 8 byte / beat
    - AxSIZE = 4 → 16 byte / beat (… 이런 식)
    
    ⇒ beat 크기 = 2^AxSIZE (byte)
    

1. “버스 폭 > burst size”인 경우 무슨 말이냐면
    
    예를 들어:
    
    AXI data bus 폭 = 32bit (4 byte)
    
    - AxSIZE = 2 (4 byte) → 딱 맞음
    - AxSIZE = 1 (2 byte) → 절반만 씀
    - AxSIZE = 0 (1 byte) → 1바이트만 씀
    
    이때:
    
    - 데이터는 항상 WDATA[31:0] 전체로 가지만
    - WSTRB로 실제로 쓰는 바이트 레인을 표시
    - 주소 하위 비트로 어느 바이트 위치인지 결정
    
    ⇒ 그래서 “주소 + byte lane 선택” 이야기가 나옴
    

1. 전송 크기는 버스 폭을 넘으면 안 된다
    - master가 32bit 버스인데
        - AxSIZE = 3 (8 byte) ❌
    - slave가 32bit 버스인데
        - AxSIZE = 4 (16 byte) ❌
    
    ⇒ 즉, 둘 중 작은 쪽 버스 폭이 상한선
    

예시로 한 번에 정리

예
1️⃣: 32bit AXI, 레지스터 접근

- AWSIZE = 2 → 4 byte / beat
- AWLEN = 0 → 1 beat
- 결과: 32bit 레지스터 1개 write

예
2️⃣: 32bit AXI, byte write

- AWSIZE = 0 → 1 byte / beat
- 주소 = 0x1003
- WSTRB = 1000
- 결과: 상위 1바이트만 write

---

---

**Burst Type**

3가지 타입이 있으며, 버스트 타입은 AxBURST 신호로 지정한다

- 읽기 전송 : ARBURST[1:0]
- 쓰기 전송 : AWBURST[1:0]

FIXED, INCR, WRAP 이 있는데 INCR을 디폴트로 쓰는것 같다

1. FIXED (고정 버스트)
    
    주소는 포트 입구 주소라고 보면 되고, 실제 저장 위치는 하드웨어가 관리하는 주고 
    
    “소프트웨어에서 겉보기엔 주소 하나 인데, 하드웨어가 내부적으로 다음 칸으로 주소를 이동 시킴”
    
    FIXED 버스트에서는 아래와 같이 동작한다
    
    - 버스트 내 모든 전송에서 주소가 동일하다
    - 유효한 바이트 레인(byte lane)은 모든 beat에서 동일하지만, 그 바이트 레인 안에서 WSTRB로 실제 쓰이는 바이트는 beat 마다 달라질 수 있다.
    
    FIXED 버스트 타입은 FIFO를 채우거나 비울 때 처럼 같은 주소를 반복 접근하는 경우에 사용된다.
    
    예 1) UART TX FIFO
    
    UART Register Map (전형적인 구조)
    
    ```verilog
    0x4000_0000 : TXDATA   ← 데이터 쓰는 곳 (FIFO 입구)
    0x4000_0004 : RXDATA
    0x4000_0008 : STATUS
    ```
    
    TXDATA 레지스터 주소는 딱 하나, 근데 FIFO 깊이는 예를 들면 64byte
    
    이때, CPU/DMA가 데이터를 보낸다면 아래와 같음
    
    ```verilog
    *(volatile uint32_t *)0x4000_0000 = 'H';
    *(volatile uint32_t *)0x4000_0000 = 'E';
    *(volatile uint32_t *)0x4000_0000 = 'L';
    *(volatile uint32_t *)0x4000_0000 = 'L';
    *(volatile uint32_t *)0x4000_0000 = 'O';
    ```
    
    주소는 모두 0x4000_0000로 보내는데, 데이터를 덮어쓴거 아니냐???
    
    덮어쓴게 아니라, 하드웨어 내부 FIFO Write Pointer가 write 할 때마다, 자동으로 다음 슬롯으로 이동
    
    ‘H’ → FIFO[0]
    
    ‘E’ → FIFO[1]
    
    ‘L’ → FIFO[2]
    
    …
    
    요렇게 됨 
    
    예 2) DMA관점에서 UART TXDATA 쓸 때
    
    DMA 설정
    
    source : DDR(INCR)
    
    destination : UART TXDATA(FIXED)
    
    burst : 256 beats
    
    ```verilog
    SRC_ADDR_MODE = INCR
    DST_ADDR_MODE = FIXED
    -> source, destination 주소 모드를 설정해준다
    ```
    
    DMA가 실제 하는 일
    
    ```verilog
    for (i = 0; i < 256; i++) {
      write dst_addr (항상 같음) with src_data[i];
    }
    -> 주소 증가 안함
    -> 데이터는 계속 바뀜
    -> FIFO 내부 포인터 자동 증가
    ```
    
    만일 destination을 ICNR로 하면
    
    ```verilog
    0x4000_0000
    0x4000_0004
    0x4000_0008
    0x4000_000C
    ...
    -> 주소가 증가
    -> STATUS, CONTROL 레지스터 등 다른 의미 없는 주소들까지 침범
    => 에러 발생
    ```
    
2. INCR (증가 버스트)
    
    INCR 버스트에서는 
    
    - 각 전송의 주소는 이전 전송 주소에 증가 값을 더한 것
    - 증가 값은 전송 크기(size)에 따라 결정
    
    예를 들어
    
    전송 크기가 4byte라면
    
    다음 주소 = 이전 주소 + 4
    
    이 버스트 타입은 일반적인 연속 메모리 접근에 사용된다.
    
3. WRAP (랩핑 버스트)
    
    WRAP 버스트는 INCR과 유사하지만 주소가 상한에 도달하면 낮은 주소로 되돌아간다(wrap).
    
    WRAP 버스트에는 제한이 존재한다
    
    - 시작 주소는 각 전송 크기에 정렬 (aligned)되어야 한다
    - 버스트 길이는 2, 4, 8, 16 전송 중 하나 여야 한다
    
    WRAP 버스트의 동작은 아래와 같다.
    
    - 버스트에서 사용되면 죄소 주소는 : 전송 크기 * 전송 갯수
        
        로 정렬되며, 이 주소를 wrap boundary라고 한다
        
    - 각 전송 후 주소는 INCR과 동일하게 증가한다
    - 증가한 주소가 ‘wrap boundary + 전체 전송 데이터 크기’ 가 되면 주소는 wrap boundary 로 되돌아간다.
    - 첫 번째 전송 주소는 wrap boundary보다 클 수 있는데, 이 경우 해당 WRAP 버스트는 반드시 중간에 wrap이 발생한다
    
    이 버스트 타입은 “캐시 라인 접근” 에서 사용된다.