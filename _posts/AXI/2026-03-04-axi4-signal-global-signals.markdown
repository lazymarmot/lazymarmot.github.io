---
layout: post
title:  "AXI4 SIGNAL – Global Signals"
date:   2026-03-04 12:00:00 +0900
categories: AXI4 AMBA
show_on_home: false
---

# [SIGNAL] Global Signals

![image.png](/assets/posts_image/AXI/SIGNAL%20Global%20Signals/image.png)

**ACLK**

AXI Component는 단일 Clock 신호 ACLK를 사용한다.

| **출력** | **입력** |
| --- | --- |
| **CPU의 출력 신호**
CPU가 **구동(drives)** 하는 신호들
• AWVALID
• AWADDR
• WDATA
• WVALID
• ARVALID
• ARADDR
• BREADY
• RREADY
👉 “내가 밖으로 내보내는 신호” | **CPU의 입력 신호**
CPU가 **받아서 샘플링**하는 신호들
• AWREADY
• WREADY
• BVALID
• BRESP
• RVALID
• RDATA
• RRESP
👉 “상대(slave/인터커넥트)가 보내주는 신호” |
| **GPIO의 출력 신호**
• AWREADY
• WREADY
• BVALID
• BRESP
• RVALID
• RDATA
• RRESP | **GPIO의 입력 신호**
• AWVALID
• AWADDR
• WVALID
• WDATA
• ARVALID
• ARADDR
• BREADY
• RREADY
👉 **같은 신호라도, 기준이 바뀌면 입력/출력이 바뀐다** |

모든 입력 신호는 ACLK의 상승 엣지에서 샘플링 된다.

“입력 신호 = 밖에서 들어오는 말”

Master 입장 : AWREADY, WREADY, RDATA

Slave 입장 : AWVALID, WDATA, ARADDR

샘플링 = "그 순간의 값을 읽어서 내부 레지스터에 저장"

언제? = > ACLK 상승 엣지 딱 그 순간 (엣지 순간 값만 본다)

```verilog
always @(posedge ACLK) begin

	in_reg <= INPUT_SIGNAL;  // ← 이게 샘플링

end
```

모든 출력 신호의 변경은 ACLK의 상승 엣지 이후에 발생해야 한다.

“출력신호 = 내가 밖으로 말하는 것”

Mastser 입장 : AWVALID, WDATA

Slave 입장 :AWREADY, RVALID

=> 엣지에서 계산하고, 엣지가 지나고 나서 바꿔라

```verilog
always @(posedge ACLK) begin

	OUTPUT_SIGNAL <= next_value;

end
```

![image.png](/assets/posts_image/AXI/SIGNAL%20Global%20Signals/image%201.png)

![image.png](/assets/posts_image/AXI/SIGNAL%20Global%20Signals/image%202.png)

![image.png](/assets/posts_image/AXI/SIGNAL%20Global%20Signals/image%203.png)

![image.png](/assets/posts_image/AXI/SIGNAL%20Global%20Signals/image%204.png)