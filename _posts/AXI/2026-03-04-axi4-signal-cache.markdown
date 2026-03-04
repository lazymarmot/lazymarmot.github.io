---
layout: post
title:  "AXI4 SIGNAL – CACHE"
date:   2026-03-04 12:00:00 +0900
categories: AXI4 AMBA
show_on_home: false
---

# [SIGNAL] CACHE

AXI에서 트랜잭션이란 :

“하나의 메모리/레지스터 접근 요청이 시작되어서 데이터 전송과 응답까지 전부 끝나는 전체 절차를 말한다”

크게 3가지 트랜잭션이 있다 (주소 채널, 데이터 채널, 응답 채널)

Write Transaction

1. Write Address Phase (AW : 주소 채널)
    
    AWADDR
    
    AWPROT
    
    AWVALID <-> AWREADY
    
    "어디에, 어떤 속성으로 쓸거야"를 알림
    
2. Write Data Phase (W : 데이터 채널)
    
    WDATA
    
    WSTRB
    
    WVALID <-> WREADY
    
    WLAST
    
    실제 데이터 전달, Burst면 여러 beat 전송
    
3. Write Response Phase (B : 응답채널)
    
    BRESP
    
    BVALID <-> BREADY
    
    "쓰기 완료됐어 / 에러 발생 했어" 응답
    
    이 3단계 전체가 합쳐져서 1개의 write transaction
    

**Transaction types and Attributes**

AXI에서 분리해서 봐야 할 것

1. 트랜잭션 속성
    
    cacheable?, bufferable?, speculative가능?, reorder 가능?
    
    AxCACHE 신호로 설정
    
2. 슬레이브 성격
    
    Memory Slave 
    
    Peripheral Slave
    
3. 트랜잭션 종류
    
    Read / Write
    
    Burst / Single
    

“Slave의 성격(type)에 따라 어떤 트랜잭션 속성(AxCACHE 등)을 지원해야 하느냐 가 달라진다. “

1. Memory Slave
    1. 메모리 슬레이브는 AXI가 제공하는 모든 속성(AxCACHE등)이 의미가 있도록 설계되어야 한다.
    2. 모든 AXI 트랜잭션 조합을 제대로 처리해야 한다. 
    3. Burst, cacheable, redorder, speculative 등 전부 고려해야 한다
    
    예 : DDR Controller, SRAM, Cache-backed memory
    
    이런 애들 특징은
    
    데이터가 주소 기반으로 저장된다
    
    읽고 쓰면 같은 주소면 같은 의미
    
    CPU 캐시랑 연동될 수 있음
    
    Burst OK / Out-of-order OK / Reorder OK / Speculative access OK / Cacheable, bufferable 이해 가능 ⇒ AXI의 모든 트랜잭션 속성을 의미 있게 처리 가능 
    
2. Peripheral Slave (주변장치 슬레이브)
    1. ‘레지스터 + 동작 트리거’로 움직이는 애들
    2. UART, SPI, GPIO, Timer, DMA Register Block
    3. 이런 애들의 특징
        - 주소 = 레지스터를 의미함
        - write = 동작 발생
        - read = 상태 읽기
        - 순서 꼬이면 안됨
        - speculative 안돼, cache 안돼
        
        안되는게 많이서
        
        - burst 의미 없음 (있어도 제한적)
        - reorder불가
        - speculative access 위험
        - Cacheable 절대 안돼
    4. 주변장치 슬레이브는 대부분 고급 트랜잭션 속성을 지원하지 않아도 된다.
    5. 주변장치 슬레이브는 구현 정의(implemetation defined)방식의 접근 방법을 갖는다
        
        <aside>
        💡
        
        구현 정의(implementation defined) 접근 방식  
        
        이 주변 장치가 '정상적으로 동작한다고 보장하는 접근 규칙'을 AXI가 아니라 
        
        그 IP의 설계자(datasheet)가 정해 놓았다는 뜻
        
        AXI가 보장하는 것   
        
        VALID/READY 핸드셰이크   
        
        응답(BRESP/RRESP)   
        
        데드락 없음
        
        AXI가 보장하지 않는 것 (주변 장치 설계자의 영역)   
        
        이 레지스터를 이렇게 쓰면 어떤 값이 된다   
        
        burst로 써도 된다   
        
        byte write가 의미 있다
        
        - 구현 정의 접근 방식은 해당 컴포넌트의 데이터시트에 정의 되어 있으며, 주변 장치가 트랜잭션 타입을 설명한다.
        - 정해진 접근 방식만 정상 동작하고 나머지는 결과가 이상해 질 수도 있다 (레지스터가 꼬일 수 있음)
        - 응답은 반드시 해야 한다. (deadlock 방지)
        
        <구현 정의 접근 방식 예시>
        
        UART 레지스터
        
        - 32bit aligned access only
        - byte or halfword access is not supported
        - burst access is not supported
        
        ⇒ 요런게 바로 구현 정의 접근 방식
        
        예 : " 이 레지스터는 32bit write만 지원", "burst access 금지", "unaligned access금지”
        
        </aside>
        
        주변장치 슬레이브에 대해 "구현 정의된 접근 방식이 아닌 접근"이 발생해도 그 접근은 "AXI 프로토콜을 준수"해서 완료되어야 한다.
        
        주변 장치가 아래의 접근을 받았다고 하면
        
        - Burst write, narrow transfer, cacheable access
        
        데이터시트에서 금지한 접근이라도 : SLVERR/OKAY응답은 반드시 해야 함
        
        다만, 그런 접근이 한 번이라도 발생한 이후에는 주변장치 슬레이브가 정상적으로 동작을 계속할 필요는 없다.
        
        단, 이후의 트랜잭션들도 프로토콜 측면에서는 정상적으로 완료되어야 한다.
        

**Transaction Attribute**

AXI 프로토콜은 “메모리 슬레이브”와 “주변장치 슬레이브”를 지원하기 위해 트랜잭션 속성(attribute)집합을 정의 한다.

AxCACHE는 “이 트랜잭션은 메모리 용인지, 주변장치 용인지” 시스템에게 알려주는 힌트

```verilog
DDR Access:
	AxCACHE = cacheable / bufferable
	
UART register access:
	AxCACHE = non-cacheable / non-bufferable

이걸 보고 캐시는 저장할지 말지 결정, interconnect는 reorder가능 여부 판단
```

1. AxCACHE는 Master가 주소 채널을 통해 Slave(및 interconnect)로 보내는 신호다
    
    누가 이 신호를 보나?
    
    AxCACHE는 단순 슬레이브만 보는게 아니라 아래 애들이 다 참고 한다. 
    
    - AXI Interconnect
        - 버퍼링 가능?
        - 재정렬 가능?
    - Cache / Snoop logic
        - 캐시 라인 할당?
    - Memory controller
        - write merge?
    - Peripheral slave
        - 보통은 “cacheable=0”만 기대함
    
    ⇒ AxCACHE는 “마스터 → 시스템 전체”로 보내는 힌트에 가까움
    

1. AxCACHE는 특정 슬레이브 전용 신호가 아니라 → 시스템 전체에서 트랜잭션을 다루는 힌트라고 보면 될듯하다. 
    
    “주소 채널을 통과하는 모든 컴포넌트가 볼 수 있는 위치에 있다”
    
    “주소 채널을 의사 결정에 사용하는 컴포넌트들은 AxCACHE를 본다, 단순 전달만 하는 애는 AxCACHE를 볼 수는 있지만 안 쓴다”
    
    ```
    AxCACHE는 :
    	주소 채널(AW/AR)에 포함됨
    	주소 채널은 interconnect를 통과해서 라우팅 됨
    -> 주소 채널을 해석하는 컴포넌트라면 AxCACHE를 볼 수 밖에 없음
    ```
    
    ---
    
    AXI의 중요한 구조 포인트
    
    “인터커넥트는 AxCACHE를 기본적으로 그대로 전달하지만,
    
    내부에 정책 로직이 있으면 AxCACHE를 해석해 의사 결정에 사용한다.
    
    단순 전달 용 인터커넥트는 AxCACHE를 보지만 사용하지 않는다.”
    
    기본 동작: “AxCACHE는 그대로 전달” (단순 crossbar / 단순  bus fabric)
    
    Master ──(AWADDR, AxCACHE, AxPROT…)──▶ Interconnect ──▶ Slave
    
    주소 디코딩만 하고, buffering/reorder 없음, firewall 없음
    
    여기서 Interconnect는
    
    - AxCACHE를 **변형하지 않고** 그대로 slave 쪽으로 전달
    - 내부 로직에서 전혀 사용 안함
    
    Interconnect는 AxCACHE를 볼 수는 있는데 아래 상황에서 주로 본다.
    
    1. Memory vs Peripheral 디코딩
        
        Interconnect가 주소 디코딩 후:
        
        - 대상이 **Memory slave**면
            
            → AxCACHE 그대로 신뢰
            
        - 대상이 **Peripheral slave인데,**
            
            → cacheable access면 에러 발생 시킴
            
    2. Reorder / buffering 허용 여부
        
        Interconnect 내부에 :
        
        - write address queue
        - write data buffer
        - write combining buffer
        - reorder queue(buffer)
        - outstanding transaction 관리 logic
        
        이 있을 경우 :
        
        - AxCACHE = non-cacheable
            
            → 순서 강제
            
        - AxCACHE = cacheable
            
            → 성능 최적화 가능
            
    3. QoS / arbitration 힌트
        
        Interconnect는:
        
        - cacheable burst
        - streaming access
        
        을 보고 **중재 우선순위**를 바꿈
        
    
    ---
    
    **예 1) AXI Interconnect (ARM NIC / CCI / CMN)**
    
    NIC, CCI, CMN같은 ‘고급 인터커넥트’는 실제로 내부에 write buffer / merge / reorder 로직이 있기 때문에 AxCACHE를 의사 결정에 사용한다
    
    AxCACHE[0] = Bufferable
    
    AxCACHE[1] = Cacheable
    
    이걸 보고 인터커넥트가 자기 내부 정책을 정한다.
    
    1. write buffer에 쌓아도 되나?
        
        AxCACHE[0] = 1 (Bufferable)
        
        - write를 내부 버퍼에 쌓아도 됨
        - master에게 빠르게 AW/WREADY 반환 가능
        
        AxCACHE[0] = 0
        
        - 버퍼링 최소화
        - write가 실제로 내려갈 때까지 기다림
        
        **⇒ 이건 인터커넥트 내부 write buffer가 있다는 뜻이 맞다**
        
    2. Write merge 가능한가?
        
        AxCACHE[0]=1(Bufferable)가 핵심 조건
        
        AxCACHE[1]=1(Cacheable)는 merge를 해도 의미가 깨지지 않는 대상이라는 추가 보증
        
        현실적으로는 둘 다 1인 경우에만 interconnect가 안심하고 merge함
        
        ```verilog
        여러 작은 write를 하나의 burst나 wider write로 합칠 수 있음
        store 1 byte × 4번
        → interconnect 내부에서
        → 32-bit write 1번
        ```
        
        Write merge 란?
        
        CPU 관점
        
        ```verilog
        *(uint8_t *)(0x8000_0000) = 0x11;
        *(uint8_t *)(0x8000_0001) = 0x22;
        *(uint8_t *)(0x8000_0002) = 0x33;
        *(uint8_t *)(0x8000_0003) = 0x44;
        CPU는 1byte write * 4번 발생시킴
        
        ```
        
        인터커넥트/메모리 시스템 관점
        
        이 주소가 Memory Slave (DDR)이라면:
        
        > “어차피 연속 주소고 중간 상태는 의미 없고 최종 메모리 값만 맞으면 되잖아?”
        > 
        
        ```verilog
        내부에서 변경함
        원래:
          write 0x11 @ +0
          write 0x22 @ +1
          write 0x33 @ +2
          write 0x44 @ +3
        
        merge 후:
          write 0x44332211 @ 0x8000_0000 (32-bit 1번)
         
        이게 write merge (write combining)
        - 트랜잭션 수 감소
        - 버스 효율 ↑
        - 성능 ↑
        ```
        
        - peripheral register 였다면 절대 하면 안되는 동작이기 때문에 AxCACHE를 보고 결정
        
        | **AxCACHE[0] = Bufferable** | **AxCACHE[1] = Cacheable** |
        | --- | --- |
        | “이 write는 중간에 잠시 쌓여도 된다
        즉, 즉시 observable 할 필요 없다”
        
        이게 write buffer / merge의 최소 조건 | “이 접근 대상은 메모리처럼 동작한다
        (중간 상태보다 최종 값이 중요)
        최적화 해도 된다”
        
        reorder / merge / speculative 해도 의미가 깨지지 않는다 |
    3. 순서를 보장해야 하나?
        - AxCACHE = non-cacheable
            - 강한 순서 보장
            - 앞 write 끝나야 다음 write
        - AxCACHE = cacheable
            - reorder 허용
            - 성능 최적화 가능
        
        ⇒ reorder queue / outstanding transaction 관리가 있다는 뜻 (최적화)
        
        ![image.png](/assets/posts_image/AXI/SIGNAL%20CACHE/image.png)
        
    - MMIO든 DDR이든 모든 접근은 먼저 interconnect를 통과하게 된다.
    
    UART 같은 peripheral들은 아래 같은 성질이 있음
    
    - 레지스터 write 순서가 의미있음
    - Write가 side-effect가짐
    - Write가 바로 반영되어야 함
    
    ```
    			Slave 가 MMIO(peripheral)인 경우
    			UART 레지스터 write
    				주소 : 0x4000_0000 (UART)
    				AxCACHE = 0000 (non-cacheable non-bufferable)
    				
    			Interconnect 내부 판단 :
    				이 주소는 peripheral 영역이다.
    				
    				AxCACHE = 0000
    					-> write buffer에 쌓지 마라 (내부 buffer에 보관 금지)
    					-> write merge하지마라 (merge / combine 금지)
    					-> 순서 그대로 보내라 (reordering 금지)
    					-> 요청 오면 바로 슬레이브로 전달해라
    					
    				그래서 결과적으로
    					CPU가 쓴 순서 그대로, 한번에 하나씩 UART로 전달됨
    					
    		=> 이걸 "interconnect에서 MMIO 접근은 write merge/reordering금지"라고 함
    
    ```
    
    **예 2) CPU L2/L3 Cache / Snoop Logic**
    
    Cacheable = 0인데 캐시에 넣으면 치명적 버그 발생됨
    
    UART 레지스터 read를 캐시해 버린다면?
    
    ⇒ 계속 같은 값 만 읽힘
    
    그래서 AxCACHE[1] = 0 (캐시 할당 금지) 로 함
    
    < 캐시에 캐시 한다는 내용을 전달하는 구조>
    
    L2/L3 캐시가 AXI 버스의 AxCACHE핀을 물리적으로 관찰 하는게 아니라, 
    
    CPU가 먼저 결정
    
    “이 접근은 cacheable이다 / bufferable이다 / device 다”
    
    결정된 결과를
    
    - 캐시 컨트롤러 쪽으로 보내고
    - AXI 쪽으로는 AxCACHE 핀으로 보냄
    
    ```verilog
    [LDR/STR 명령]
          ↓
    MMU + page table lookup
          ↓
    Memory Attribute 결정
    (cacheable? bufferable? device?)
          ↓
    ┌───────────────┬────────────────┐
    │               │                │
    캐시 컨트롤러      AXI 버스 인터페이스
    (L1/L2/L3)         (AXI master port)
    │               │
    "캐시에 넣어?"     AxCACHE[3:0]로 인코딩
    "writeback?"      → Interconnect로 전달
    
    ```
    
    캐시는 AxCACHE를 보고 판단하지 않고, AXI도 캐시에게 뭘 물어보지 않음, 
    
    둘 다 같은 원본(MMU 결과)를 CPU 내부에서 받아서 쓰는것임
    
    UART 기반 예시
    
    상황 : CPU가 UART 레지스터 읽기
    
    연결 : CPU -> AXI -> interconnect -> AXI -> UART
    
    MMU 설정
    
    UART 주소 영역
    
    - Device Memory
    - Non-cacheable
    - Strongly-ordered or device-nGnRnE
    
    이 때 내부에서 벌어지는 일
    
    1. CPU 내부 결정
        - “이 접근은 캐시 금지”
        - 캐시에 라인 할당 안 함
        - speculative read 금지
    2. AXI로 나갈 때
        - ARADDR = UART 주소
        - ARCACHE[1] = 0 (Non-cacheable)
        - ARPROT 설정
    3. Interconnect
        - reorder / merge 안 함
        - 그대로 UART로 전달
    4. UART
        - 실제 하드웨어 레지스터 값 반환
    
    만일 캐시가 UART 설정 상관 없이 지 맘대로 캐시 처리 하면?
    
    - cacheable로 처리 해버린다면?
    - 첫 read:
        - 값 읽고 캐시에 저장
    - 두 번째 read:
        - UART 안 가고 캐시에서 반환
        - 상태 레지스터 변화 못 봄
        - 인터럽트 플래그 안 바뀜
        - 드라이버 망가짐
    
    ⇒ 이게 “치명적 버그”발생 시킴
    
    ![image.png](/assets/posts_image/AXI/SIGNAL%20CACHE/image%201.png)
    
    **예 3) Peripheral Slave (UART, GPIO, DMA Reg)**
    
    AxCACHE 보통 안 씀
    
    왜?
    
    - 캐시/버퍼 개념 자체가 없음
        
        > (datasheet)  “Access must be non-cacheable”
        > 
    
    그래서:
    
    - cacheable access가 오면
        - 무시
        - undefined behavior
        - 또는 DECERR
    
    **예 4) Memory Controller (DDR Controller)**
    
    memory controller는 CPU가 설정한 cacheable/bufferable 속성(AxCACHE)의 의미를 직접 또는 interconnect를 통해 반영하여 write buffer / write combine / ordering 정책을 결정한다.
    
    AXI 스펙에서 AxCACHE는 이렇게 정의 되어 있음
    
    > “AxCACHE controls how a transaction progresses through the system
    > 
    
    여기서 시스템에는 : Interconnect, cache, memory controller(DDR controller)가 포함
    
    즉, 메모리 컨트롤러도 AxCACHE 속성의 영향을 받는 구성 요소임
    
    DDR 컨트롤러는 보통 2가지 정보를 조합해서 트랜잭션을 판단한다.
    
    1. 주소가 memory영역인지
    2. 이 트랜잭션이 cacheable/bufferable 성격인지
    
    실제 SoC 구현에서의 메모리 컨트롤러 2가지 패턴이 있음
    
    패턴 1 ) AxCACHE가 그대로 전달 되는 경우
    
    ```verilog
    CPU ----> Interconnect ----> DDR Controller (AWCACHE 수신)
    - AWADDR
    - AWCACHE
    
    이 경우 DDR Controller RTL에서 
    	- AWCACHE[0] (bufferable)
    	- AWCACHE[1] (cacheable)
    를 직접 보고
    	- write buffer 사용 여부 
    	- write combine 허용 여부
    	- flush 타이밍을 결정
    ```
    
    패턴 2) Interconnect가 정책을 먼저 적용
    
    ```verilog
    CPU --------> Interconnect --------> DDR Controller (단순 수신)
    - AWADDR          - non-cacheable 이면 즉시 전달
    - AWCACHE         - cacheable 이면 병합/버퍼링
    
    이 경우 DDR Controller에서는
    	- 이미 정리된 burst만 받음
    	- cacheable/non-cacheable의 결과만 반영
    
    "DDR Controller가 AxCACHE를 직접 보지는 않지만 AxCACHE에 의해 결정된 정책의 결과를 받는다."
    ```
    

**Cacheable write (AxCACHE=1111)**

- interconnect:
    - write buffer에 적재
    - 여러 write 병합
- DDR controller:
    - burst write 수신
    - 고효율 처리

**Non-cacheable write (AxCACHE=0000)**

- interconnect:
    - 즉시 전달
    - 병합/지연 없음
- DDR controller:
    - 단건 write
    - 순서 보장
    - flush 성격 처리

⇒ DDR controller의 관점에서는 “cacheable이냐 / 아니냐”가 분명히 다르게 보인다

[용어 설명](%5BSIGNAL%5D%20CACHE/%EC%9A%A9%EC%96%B4%20%EC%84%A4%EB%AA%85%202e96feb16a3e80dfa3f5e5e8b4b8f887.md)

[AXI3 memory attribute signaling](%5BSIGNAL%5D%20CACHE/AXI3%20memory%20attribute%20signaling%202e96feb16a3e80d79752e09c7201b1f5.md)

[**AXI Changes to memory attribute signaling**](%5BSIGNAL%5D%20CACHE/AXI%20Changes%20to%20memory%20attribute%20signaling%202e96feb16a3e80339d8fe88753446038.md)