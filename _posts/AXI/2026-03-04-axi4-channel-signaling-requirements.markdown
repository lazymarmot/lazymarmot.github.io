---
layout: post
title:  "AXI4 Channel Signaling Requirements"
date:   2026-03-04 12:00:00 +0900
categories: AXI4 AMBA
show_on_home: false
---

<br>

각 채널은 자신만의 VALID/READY 핸드셰이크 신호를 쌍으로 가진다.

![image.png](/assets/posts_image/AXI/Channel%20Signaling%20Requirements/image.png)

<br>

**Write Address Channel**

![image.png](/assets/posts_image/AXI/Channel%20Signaling%20Requirements/image%201.png)

master는 유효한 주소와 제어 정보를 실제로 구동(drives)하고 있을 때에만 AWVALID 신호를 활성화할 수 있다.
AWVALID가 한 번 활성화되면, slave가 AWREADY를 활성화한 이후의 상승 클럭 엣지까지 AWVALID는 반드시 유지되어야 한다.
AWREADY의 기본 상태(default)는 HIGH일 수도 있고 LOW일 수도 있다.
이 사양에서는 기본 상태를 HIGH로 두는 것을 권장한다.
AWREADY가 HIGH인 경우, slave는 제시되는 어떤 유효한 주소라도 받아들일 수 있어야 한다.

즉, 

1. AWVALID는 “주소가 진짜 준비됐을 때만”
    - 주소 아직 안 정해졌는데 AWVALID=1 ❌
    - AWADDR, 제어 신호가 안정된 상태 → AWVALID=1 ⭕
    
    AWVALID = “이 주소는 진짜다”라는 약속
    
2. AWVALID는 READY 올 때까지 내려오면 안 됨
    - master가 AWVALID를 올렸으면
    - slave가 AWREADY를 올려서
    - 전송이 성립되는 클럭 엣지까지
    - AWADDR/제어 신호 그대로 유지

1. AWREADY 기본 값이 HIGH여도 된다 (중요 포인트)
    
    이 말의 의미는:
    
    - slave는
        - “항상 받을 준비가 된 상태”로 설계해도 되고
    - 그러면
        - master가 AWVALID를 올리는 즉시
        - 다음 클럭 엣지에서 바로 전송 완료

1.  “AWREADY가 HIGH면 뭐든 받아야 한다”의 뜻
    
    AWREADY = 1 인 순간에는:
    
    slave는 어떤 합법적인 주소가 와도 거부하면 안 된다
    
    즉:
    
    - 내부 큐가 비어 있어야 하고
    - back-pressure를 걸 상황이면
        - AWREADY를 내려야 함

**Write Data Channel**

![image.png](/assets/posts_image/AXI/Channel%20Signaling%20Requirements/image%202.png)

쓰기 버스트 동안, master는 유효한 쓰기 데이터를 실제로 구동하고 있을 때에만 WVALID 신호를 활성화할 수 있다.
WVALID가 한 번 활성화되면, slave가 WREADY를 활성화한 이후의 상승 클럭 엣지까지 WVALID는 반드시 유지되어야 한다.
WREADY의 기본 상태는 HIGH일 수 있지만, 이는 slave가 항상 단일 클럭 사이클 내에 쓰기 데이터를 받아들일 수 있는 경우에만 허용된다.
master는 버스트의 마지막 쓰기 전송을 구동하는 동안 반드시 WLAST 신호를 활성화해야 한다.

즉,

1. WVALID 규칙 = VALID/READY 기본 규칙 그대로
    - 데이터 준비 안 됐는데 WVALID = 1 ❌
    - WDATA, WSTRB가 안정됨 → WVALID = 1 ⭕
    
    그리고:
    
    - WVALID = 1 이후에는
    - WREADY가 올라와서 전송이 성립되는 클럭 엣지까지
    - WDATA / WSTRB / WLAST 전부 고정
    
2. WREADY 기본값을 HIGH로 둘 수 있는 조건 (중요)
    
    “항상 받을 수 있으면 HIGH로 두고, 못 받을 상황이면 반드시 LOW로 내려라”
    
    즉:
    
    - 내부 FIFO 여유 있음 → WREADY = 1
    - 내부 처리 못 함 / 막힘 → WREADY = 0 (back-pressure)
    
3. WLAST의 의미 (매우 중요)
    
    WLAST = “이 beat가 이 버스트의 마지막이다”
    
    - write burst의 마지막 데이터 beat에서만
    - 반드시 WLAST = 1
    
    AXI4-Lite에서는:
    
    - burst 길이 = 1
    - 👉 WLAST는 항상 1

**Write Response Channel**

![image.png](/assets/posts_image/AXI/Channel%20Signaling%20Requirements/image%203.png)

1. BVALID 규칙 = VALID/READY 기본 규칙 그대로
    - 응답 준비 안 됐는데 BVALID = 1 ❌
    - BRESP가 안정됨 → BVALID = 1 ⭕
    
    그리고:
    
    - BVALID = 1 이후에는
    - BREADY가 올라와서 전송이 성립되는 클럭 엣지까지
    - BRESP 값 고정
    
2. BREADY 기본값을 HIGH로 둘 수 있는 조건
    
    “항상 받을 수 있으면 HIGH, 못 받을 상황이면 LOW로 내려라”
    
    대부분의 CPU는:
    
    - write response를 항상 받을 수 있음
    - 그래서 BREADY를 상시 HIGH로 두는 경우가 많음

1. 쓰기 응답은 “트랜잭션당 1번”
    - write burst가 여러 beat여도
    - B 채널 응답은 딱 한 번
    
    그래서:
    
    - W 채널 끝난 뒤
    - BVALID 한 번 올라옴

**Read Address Channel**

![image.png](/assets/posts_image/AXI/Channel%20Signaling%20Requirements/image%204.png)

master는 유효한 주소와 제어 정보를 실제로 구동하고 있을 때에만 ARVALID 신호를 활성화할 수 있다.

ARVALID가 한 번 활성화되면, slave가 ARREADY 신호를 활성화한 이후의 상승 클럭 엣지까지 ARVALID는 반드시 유지되어야 한다.

ARREADY의 기본 상태(default)는 HIGH일 수도 있고 LOW일 수도 있다.

이 사양에서는 기본 상태를 HIGH로 두는 것을 권장한다.

ARREADY가 HIGH인 경우, slave는 제시되는 어떤 유효한 주소라도 받아들일 수 있어야 한다.

```smalltalk
💡 이 사양에서는 ARREADY의 기본값을 LOW로 두는 것을 권장하지 않는다.

그 이유는, ARREADY가 LOW이면 전송에 최소 두 클럭 사이클이 필요해지기 때문이다.

하나는 ARVALID를 활성화하는 데, 또 하나는 ARREADY를 활성화하는 데 사용된다.
```

즉,

1. ARVALID = “읽기 주소가 진짜 준비됨”
    - AWVALID 때와 완전히 동일
    - ❌ 주소 불안정한데 ARVALID = 1
    - ⭕ ARADDR, 제어 신호 안정 → ARVALID = 1
    
    그리고:
    
    - ARVALID 올렸으면
    - ARREADY가 올라와 전송이 성립되는 상승 엣지까지
    - 주소/제어 정보 고정

1. ARREADY 기본 값을 HIGH로 권장하는 이유 (핵심)
    
    ARREADY = HIGH (권장)
    
    ARVALID ↑ 
    
    ⇒ ARVALID High이후 다음 상승 엣지에서 바로 전송 (1사이클 지연)
    
    ARREADY = LOW (비권장)
    
    1 Cycle : ARVALID 올림
    
    2 Cycle : ARREADY 올림
    
    ⇒ 그 다음 엣지에서 전송 ( 최소 2사이클 지연 )
    
    그래서 특별히 막아야 할 이유가 없으면 ARREADY는 기본 HIGH로 두는 게 성능 상 유리할듯
    

1. “ARREADY가 HIGH면 뭐든 받아야 한다”의 의미
    - ARREADY = 1 상태에서는
        - slave는:
            - 내부 큐 여유 있음
            - 주소 디코딩 가능
            - 어떤 합법적인 주소도 거부 ❌
        
        만약:
        
        - 내부가 바쁘면
        - 큐가 가득 찼으면
        
        ⇒ ARREADY를 내려야 함
        

Read Data Channel

![image.png](/assets/posts_image/AXI/Channel%20Signaling%20Requirements/image%205.png)

slave는 유효한 읽기 데이터를 실제로 구동하고 있을 때에만 RVALID 신호를 활성화할 수 있다.
RVALID가 한 번 활성화되면, master가 RREADY를 활성화한 이후의 상승 클럭 엣지까지 RVALID는 반드시 유지되어야 한다.

slave가 읽기 데이터의 단 하나의 소스만 가지고 있더라도, 데이터 요청에 대한 응답으로만 RVALID를 활성화해야 한다.
master 인터페이스는 RREADY 신호를 사용하여 해당 데이터를 받아들일 수 있음을 표시한다.

RREADY의 기본 상태는 HIGH일 수 있지만, 이는 master가 읽기 트랜잭션이 시작되는 즉시 언제든지 읽기 데이터를 받아들일 수 있는 경우에만 허용된다.
slave는 버스트의 마지막 읽기 전송을 구동할 때 반드시 RLAST 신호를 활성화해야 한다.

즉,

1. RVALID = “이 읽기 데이터는 진짜다”
- 데이터 아직 없는데 RVALID = 1 ❌
- RDATA, RRESP 안정 → RVALID = 1 ⭕

그리고:

- RVALID = 1 이후에는
- RREADY가 올라와서 전송이 성립되는 상승 엣지까지
- RDATA / RRESP / RLAST 전부 고정

1. “요청 없이 RVALID 올리면 안 된다” (중요)

읽기 데이터는 반드시 AR 요청에 대한 응답으로만 나와야 한다

즉:

- slave 내부에 데이터가 이미 있어도
- master가 AR 요청을 보내기 전에는
- RVALID ❌

1. RREADY 기본 값 HIGH 조건
- master가:
    - 언제든 데이터 받을 준비가 돼 있으면
    - RREADY = 1 유지 가능
- 아니면:
    - back-pressure 필요
    - RREADY = 0

CPU는 보통:

- RREADY 상시 HIGH
- DMA는 상황에 따라 토글

1. RLAST 의미

RLAST = “이 beat가 이 버스트의 마지막이다”

- AXI4:
    - burst 마지막 beat에서만 1
- AXI4-Lite:
    - burst 길이 1
    - 👉 항상 RLAST = 1