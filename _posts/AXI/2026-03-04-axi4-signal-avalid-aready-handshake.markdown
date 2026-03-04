---
layout: post
title:  "AXI4 SIGNAL – AVALID, AREADY & Handshake"
date:   2026-03-04 12:00:00 +0900
categories: AXI4 AMBA
show_on_home: false
---

# [SIGNAL] AVALID, AREADY  & Handshake

**Handshake process (핸드셰이크 과정)**

![image.png](/assets/posts_image/AXI/SIGNAL%20AVALID%20AREADY%20Handshake/image.png)

다섯 개의 모든 트랜잭션 채널은 주소, 데이터, 제어 정보를 전달하기 위해 동일한 VALID/READY 핸드셰이크 방식을 사용한다.

이 양방향 흐름 제어(two-way flow control) 메커니즘을 통해 master와 slave 양쪽 모두 정보가 이동하는 속도를 제어할 수 있다.

정보를 보내는 쪽(source)은 주소, 데이터 또는 제어 정보가 준비되었음을 나타내기 위해 VALID 신호를 생성한다.

정보를 받는 쪽(destination)은 해당 정보를 받아들일 수 있음을 나타내기 위해 READY 신호를 생성한다.

VALID와 READY 신호가 둘 다 HIGH일 때만 전송이 발생한다.

master 인터페이스와 slave 인터페이스 모두에서 입력 신호와 출력 신호 사이에 조합 논리 경로(combinatorial path)가 존재해서는 안 된다.

그림 A3-2에서,

source는 T1 이후에 주소, 데이터 또는 제어 정보를 제시하고 **VALID 신호를 활성화**한다.

destination은 T2 이후에 **READY 신호를 활성화**하고,

source는 T3에서 전송이 인식될 때까지 **자신의 정보를 안정적으로 유지해야 한다.**

쓰기 주소 채널 (AW )

```verilog
T1 : AWVALID = 1 (주소 준비 됨)
T2 : AWREAD = 1 (받을 준비 됨)
T3 : AWVALID = 1 & AWREADY = 1 -> 전송 발생
```

**T1 이후** 

- VALID = 1 (source 가 정보 준비 완료)
- READY = 0
    
    ⇒ 아직 전송 안함
    
    ⇒ 이때 INFORMATION은 반드시 고정(stable) 되어 있어야 함
    

**T2 이후**

- VALID = 1
- READY = 1 (destination이 받을 준비 완료)
    
    ⇒ 이 상태가 다음 클럭 엣지까지 유지
    
    ⇒ 이 때는 아직 “조건이 만족된 상태”일 뿐 아직 전송은 안 일어남 (AXI는 엣지 트리거임)
    

**T3 클럭 엣지**

- VALID = 1
- READY = 1
    
    ⇒ 이 클럭 엣지에서 INFORMATION이 샘플링 됨
    
    ⇒ 전송이 공식적으로 성립
    

“T3에서 전송이 시작된다” 라기 보다는 “T3 클럭 엣지에서 전송이 발생한다” 가 정확

AXI 전송은 VALID와 READY가 모두 1인 상태에서 “다음 클럭 엣지”에 발생한다. 

그 지점이 그림의 T3다,  T3 상승 엣지에서 VALID=1 && READY=1이면 INFORMATION이 캡처됨

RTL관점으로 보면 (AXI 수신측 로직)

```verilog
	always @(posedge ACLK) begin  
		if (VALID && READY) begin    
			recv_data <= INFORMATION;  // 이 엣지에서 바로 캡처  
		end
	end
	
	○ VALID && READY 평가
	○ INFORMATION 샘플링→ 모두 같은 posedge
```

**Figure A3-3(READY가 VALID보다 먼저 올라오는 경우)** 의 규칙과 의미를 정확히 설명하는 부분

![image.png](/assets/posts_image/AXI/SIGNAL%20AVALID%20AREADY%20Handshake/image%201.png)

destination이 T1 이후에 READY 신호를 먼저 활성화 하는데,이는 주소, 데이터 또는 제어 정보가 아직 유효하지 않더라도 해당 정보를 받을 준비가 되었음을 의미한다.
source는 T2 이후에 정보를 제시하고 VALID 신호를 활성화하며, 이 VALID 활성화가 인식되는 시점인 T3에서 전송이 발생한다.

그림 A3-3에서는,

destination이 T1 이후에 READY 신호를 먼저 활성화하는데 이는 주소, 데이터 또는 제어 정보가 아직 유효하지 않더라도 해당 정보를 받아들일 준비가 되어 있음을 의미한다.

source는 T2 이후에 정보를 제시하고 VALID 신호를 활성화하며, 이 VALID 활성화가 인식되는 시점인 T3에서 전송이 발생한다.

이 경우, 전송은 단일 클럭 사이클(single cycle) 에서 이루어진다.

**전송이 실제로 일어나는 시점**

**VALID = 1 && READY = 1 인 상태의 “상승 클럭 엣지”**

Figure A3-3에서는 그 시점이 **T3**.

VALID는 READY를 “보고” 올리면 안 된다

- READY가 1이 될 때까지 기다렸다가 VALID 올리기 ❌
- 데이터가 준비되면 바로 VALID 올리기 ⭕

이 규칙이 있어야 교착(deadlock) 이 생기지 않는다.

**Figure A3-4 VALID와 READY가 함께 올라오는 handshake**

![image.png](/assets/posts_image/AXI/SIGNAL%20AVALID%20AREADY%20Handshake/image%202.png)

그림 A3-4에서는,

source와 destination **양쪽 모두가 T1 이후에** 주소, 데이터 또는 제어 정보를 **전송할 수 있음을 동시에 표시**한다.

이 경우, **VALID와 READY가 둘 다 활성화되었음이 인식되는 상승 클럭 엣지에서 전송이 발생한다.**

이는 전송이 **T2에서 발생한다**는 것을 의미한다.

T1 이후

source :

INFORMATION 준비됨

VALID =  1

destination :

받을 준비 완료

READY = 1

이미 조건 충족 상태 (VALID = 1 && READY = 1)

T2 상승 클럭 엣지

VALID = 1

READY = 1

이 클럭 엣지에서 

- INFORMATION이 샘플링 됨
- 전송 즉시 발생

T2이후 

source : 

- 다음 데이터 준비 가능
- VALID 내릴 수도 있음

destination : 

- READY 내릴 수도 있음