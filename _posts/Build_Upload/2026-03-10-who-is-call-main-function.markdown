---
layout: post
title: "main 호출은 누가 할까"
date: 2026-03-10 11:20:00 +0900
categories: Build Upload
show_on_home: false
---

임베디드 환경에서 main함수를 누가, 어떻게 호출 하는 걸까?

<br>

### 로딩 단계

리눅스에서는 ./test 명령을 쉘에서 실행하면 execve라는 로더가 실행 파일 test를 메모리로 로드하고 프로그램의 시작 부분으로 분기한다. 

임베디드 환경에서는 링커가 실행 파일을 만들면 로더가 보드에 로딩을 하는 과정을 거치게 된다. 

여기서 로더는 상황에 따라 다르게 불릴 수도 있다

- JTAG/다운로더(OpenOCD, ST-Link 등..)가 Flash/RAM에 write한다.
- 부트로더(U-boot같은) 또는 ROM 부트가 FLASH에서 RAM으로 옮겨줌
- 그냥 FLASH에서 XIP로 실행(코드 그대로 실행)

<br>

### 타겟 보드에 펌웨어 업로드

JTAG / ST-Link로 개발할 때 (디버깅 모드)

일반적으로 IDE에서 *.elf 파일 선택해서 보드에 업로드 하는데

이 때 내부적으로, 디버거(ST-Link, JTAG)은 elf 파일 내부를 확인하게 된다.

이 파일 내부를 보고

- Program Header
- Load Address

필요한 섹션들을 뽑아서  → flash에 write하거나, RAM에 직접 write한다.

⇒ 유저는 elf파일을 올린 것으로 알고 있지만 디버거는 내부적으로 bin처럼 처리(.raw / .img)해서 저장 장치에 저장

```markdown
※ bin 파일 
- elf 에서 실행에 필요한 부분만 링커가 정한 주소 기준으로 쭉 늘어놓은 byte덩어리 
- (포함).text(코드), .rodata, .data의 초기값 부분 
- (비포함) .bss, stack/heap(개념만 존재)

※ bin 파일 레이아웃
	BIN 파일:
	  [ startup code (.text 시작) ]
	  [ 나머지 .text ]
	  [ .rodata ]
	  [ .data 초기값 ]
```
<br>

### 타겟 보드 부팅

→ 전원 인가

CPU는 하드웨어 규칙에 따라 시작하게 되는데, 

- 이 시작 지점은 내부 Boot ROM(0x0)일 수도 있고,
- SoC 내부 고정 주소 일 수도 있다 (startup 코드랑은 무관)

→ 내부 Boot ROM 실행

1. 부트 모드 관련된 핀 확인(eMMC부팅?, SD 부팅?, SPI Flash 부팅?)
    
    
2. 저장장치에서 부트 이미지 읽어서 지정된 주소로 로드
    - eMMC/SD 같은 저장장치에서 0번지부터 n바이트 읽어서 RAM의 0x8000_0000번지에 그대로 복사
        
        (저장 장치에 있는 *.bin을 고대로 읽어서 RAM에 memcpy 함)
        
        (Flash XIP 사용하는 경우 복사 안하고 그냥 그 주소로 실행)
        
    - 이 때 저장장치에서 읽는 .bin파일 형식은 .text/.data 초기 값들이 배치되어 있고  .text영역의 가장 초기에는 startup 코드가 들어있다.
        
        
3. 로드 한 주소로 PC 점프 뜀
    
    보통 링커 스크립트 파일에서 결정한 주소에 의해서 .text영역의 시작 순서 결정됨
    
    ```markdown
    링커 스크립트
    .text :
    {
        KEEP(*(.isr_vector))   // (Cortex-M면)
        *(.text.startup*)
        *(.text*)
    }
    ENTRY(_start)
    
    - .text 섹션 안에 vector테이블 -> startup -> 나머지 코드들 순으로 넣겠다
    - 실행 시작 PC는 _start 심볼 주소로 하겠다.
    ```
    
    ```markdown
    startup파일
    Cortex-M 시리즈
    	- 보통 _start를 직접 안씀
    	- 링커 스크립트에서 .text 맨 앞에 "KEEP(*(.isr_vector))"이거 때문에 벡터 테이블을 0x0쪽에 배치하게 됨
    	- Vector Table 0번에는 Reset Handler의 주소가 있는데, 
    	- CPU는 0x0에서 값을 2개 읽게 되는데, 
    			초기 SP(stack pointer)값이랑 0번에 있는 Reset Handler주소 임 
    			이후 Reset Handler가 존재하는 코드로 점프 함
    	- ".isr_vector는 실행 코드가 아니라 점프 대상 주소 목록(테이블)일 뿐 시작하는 지점은 아니라는것임"
    	=> 즉, 엔트리 포인트가 개념상 Reset Handler라고 볼 수 있음
    
    Cortex-A/R(리눅스/부트로더 쪽)나 bare-metal일때 
    	- _start 엔트리 흔하게 사용함
    	- startup.s 파일에 보통 아래와 같은 내용이 있음
    			.section .text.startup, "ax"
    			.global _start
    			_start:
    			    ...
    			(위 내용은 
    				=> "이 코드부터는 .text.startup 섹션에 들어가고, 그 시작 지점에 _start라는 심볼(라벨)을 붙일거야"를 의미
    	- 링커 스크립터에서 _start엔트리를 작성했다면, 
    		"로더/ROM 코드는 일단 _start 엔트리 주소로 PC를 맞춰서 시작함"
    	- A는 M처럼 0번지에 벡터 테이블이 고정이 아니라
    		VBAR(벡터 베이스 레지스터) 같은 걸로 정하거나
    		특정 high/low vector 설정, SoC/부트 단계 설정으로 결정
    	
    ```
    

1. data/bss/stack/heap은 언제 생길까?
    
    startup 코드가 진행되면서 영역이 생기게 됨
    
    **startup 코드란?**
    
    startup코드는 .s 어셈블리 코드로 되어 있으며, 임베디드 환경에서는 startup코드를 개발자가 직접 수정해서 사용하는 경우도 많다.
    
    링커에서 코드 병합 할 때 startup코드도 .text의 일부가 되기 때문에 링커 스크립트로 정한 .text 시작 주소가 startup 코드 + 나머지 코드 전부에 적용된다. 
    
    ```c
    startup.o → _start, reset handler내용이 있음
    
    코드 병합 시
    링커가 하는 일 :
    			.text =
    			  startup.o(.text)
    			  b.o(.text)
    			  c.o(.text)
    	스타트업 코드 내용도 통쨰로 .text안에 들어가며, 이 시점에서는 아직 주소가 미지정이고 그냥 순서만 정해져 있음
    	
    	링커 스크립트로 .text시작 주소 결정
    		.text 0x80000000 : {
    	    *(.text*)
    		}
    		
    		.text 시작주소 = 0x80000000
    		startup 코드의 첫 명령어 주소 = 0x80000000
    		foo, bar도 그 뒤 주소로 줄줄이 배치
    ------------------------------------------------------------------
    	
    	주소 해석/확정 시
    		링커가 주소 계산
    			심볼 절대주소 = .text 시작주소 + 섹션 내부 offset
    				_start = 0x80000000
    				foo = 0x80000040
    				bar = 0x80000120
    		이 계산은 startup 코드에도 그대로 적용됨
    ```
    <br>

    ***스타트업 코드가 쓰는 심볼***
    
    - 왼쪽 : ELF/C 언어 관점 구조
    - 오른쪽 : 스타트업/로더 관점 구조
    
    ![image.png](/assets/posts_image/Build_Upload/who_is_call_main_function/image.png)
    
    스타트업 코드에서 C 프로그램이 사용할 메모리 영역을 할당하려면 각 세그먼트의 시작과 끝을 나타내는 심볼들이 있어야 된다.
    
    세그먼트 시작, 끝 심볼과 ROM/RAM의 시작 주소 정보는 .ld (링커 스크립트) 파일에 있다.
    
    위 그림에서, text 세그먼트는(code+ro-data)의 시작 주소는 ro-base 끝 주소는 ro-limit라는 심볼을 사용한다. 
    
    rw-data 세그먼트의 시작주소는 rw-base이고 끝 주소는 zi 세그먼트의 시작 주소와 같기 때문에 zi-base 심볼을 이용하면 된다.
    
    zi 세그먼트의 시작 주소는 zi-base 끝 주소는 zi-limit를 사용한다.
    
    <br>
    
    **startup 코드가 하는 일**
    
    ARM을 기준으로 보자면, M시리즈와 A시리즈의 startup 코드가 약간 다르다
    
    **M시리즈**
    
    1. 스택 포인터 / 예외 처리 벡터 설정
        
        .text영역 시작 주소(0x0번지)에 예외 처리 테이블을 배치 한다. 
        
        (ARM의 경우 예외 처리 벡터는 보통 0번지에 위치한다.)
        Vector Table Base Address = `0x0000_0000`
        
        CPU는 무조건 0x0000_0000부터 벡터 테이블로 해석
        
        | 오프셋 | 내용 |
        | --- | --- |
        | 0x00 | Initial MSP 값 |
        | 0x04 | Reset Handler 주소 |
        | 0x08 | NMI Handler |
        | 0x0C | HardFault Handler |
        | … | 기타 예외/인터럽트 |
        
        **ARMv7-M Architecture Reference Manual**
        
        (ARM Cortex-M3/M4/M7 계열)
        
        > **“On reset, the processor reads the vector table starting at address 0x00000000.”**
        > 
        
        **ARMv6-M Architecture Reference Manual**
        
        (Cortex-M0/M0+)
        
        > 동일하게 0x00000000이 기본 벡터 베이스
        > 
        
        startup 코드 시작하게 되면 0x0000_0000에서 4바이트 읽음
        
        → MSP(Main Stack Pointer) 초기 값인데 CPU는 이 값을 SP레지스터에 로드
        
        ```markdown
        MSP = *(uint32_t*)0x00000000;
        ```
        
        실행이 아니라 그냥 레지스터 설정
        
        → 다음 주소 0x0000_0004에서 4바이트 읽음
        
        Reset Handler 주소 가 PC에 설정됨
        
        ```markdown
        PC = *(uint32_t*)0x00000004;
        ```
        
        → PC가 Reset Handler 코드가 있는 주소를 가리키게 됨
        
        Reset Handler 주소의 첫 명령어부터 시작되는 것 
        
        ※ 왜 스택 포인터 먼저 설계 했지??
        
        인터럽트/예외는 스택을 사용하는데, 리셋 직후에도 예외가 발생 가능하기 때문에 코드 실행 전에 스택부터 안전하게 준비
        
        ```markdown
        Reset_Handler:
            bl SystemInit     // ① 시스템 클록/PLL/Flash 설정
            bl __libc_init    // (있다면)
            ...
            bl main
        리셋 핸들러에서 SystemInit 함수를 실행
        ```
        
    2. (필요 시) 인터럽트 disable
        
        __disable_irq() 해줌 (보통 reset 시 이미 disable 상태이긴 함)
        
    3. STM32면 SystemInit() 실행
        - HSI On
        - HSE On
        - PLL 설정
        - Flash latency 설정
        - AHB / APB 분주 설정
        - 벡터 테이블 위치 설정(VTOR)
        
        ---
        
        ※ 시스템 클럭 / PLL 설정
        
        M시리즈 MCU는 리셋 직후 보수적인 상태로 시작한다. 
        
        - HSI → HSE → PLL 순서로 설정
            - HSI (내부 RC 오실레이터)
                - 리셋 직후 바로 사용 가능
                - 외부 부품 필요 없음
                - 느림 (예를 들어 8MHz) → 고속 불가능
                - 정확도 낮음 → 주파수 오차 큼
                - 전력/안정성 위주 사용 ⇒ 부팅 용 임시 클록 느낌
            - HSE (External Crystal)
                - 외부 크리스탈
                - 정확함(UART/USB/ETH 필수)
                - 안정화 시간이 필요해서 바로 못씀 (그래서 HSI로 시작, HSE 준비)
            - PLL (Phase Locked Loop)
                - HSE나 HSI를 입력으로 받음
                - CPU가 쓸 고속 클럭 생성
                - HSE 8MHz, PLL *9 ⇒ CPU 72MHz
        
        ※ Flash wait-state 설정 (Flash Latency)
        
        - CPU가 주소를 보내면, Flash 메모리는 그 주소에 해당하는 데이터를 내부에서 읽어서 데이터 버스에 올려주고 CPU가 그걸 샘플링한다.
            - CPU의 PC값이 주소 버스로 나감 → 이 주소가 Flash 메모리로 전달
            - Flash는 주소를 받고 내부에서 read 동작 수행 (50ns 동안 처림)
            - Flash가 데이터 버스에 값을 올림 (데이터가 Data bus에 valid 상태로 올라감)
            - CPU는 자기 클럭 엣지에서 Data Bus의 값을 캡쳐
            - 한 클럭 안에 읽는다 ⇒ CPU는 같은 클럭 주기 안에서 데이터 읽기 성공
        - CPU가 Flash에서 명령/데이터 읽을 때 Flash가 느려서 “한 박자 쉬게 하는 시간” 임
            - Flash는 SRAM 처럼 즉시 못 읽음
            - 클럭이 빨라질 수록 읽는 데 여러 사이클이 필요함
                
                (몇 클럭 기다렸다가 읽어 라고 설정하는 거임 이때 기다리는 클럭 수를 wait-state 라고 함)
                
                ```markdown
                Flash는 자기 속도가 있는데, 
                CPU 클럭이 빨라질 수록 Flash 입장에서는 더 많은 CPU사이클이 필요해짐
                
                Flash read access time = 50ns (클럭 상관 없이 고정된 물리 시간)
                - flash에서 주소를 받아서 내부 read access time(50ns) 동안 처리
                
                CPU Clock = 8MHz (1 Clock = 125ns)
                		시간 →
                		Flash read:  |----------| 50ns
                		CPU clock:   |------------------------| 125ns
                	Flash 읽기가 50ns이니까 
                		첫 클럭 엣지 : 
                				CPU가 PC 기반으로 주소를 Address Bus에 출력
                		Flash는 내부에서 50ns 동안 읽기 진행
                		다음 클럭 엣지 : 
                				Data Bus에 값이 valid
                				CPU가 이 시점에 데이터 캡쳐
                	wait-state = 0
                	
                #----------------------------------------#
                	
                CPU Clock = 72MHz (1 Clock = 13.9ns)
                		시간 →
                		Flash read:  |--------------------| 50ns
                		CPU clock:   |-|-|-|-|-|            (13.9ns × 4 ≈ 56ns)
                	Flash 읽기가 50ns이니까 
                		CPU가 Flash 주소 데이터 요청
                		Flash는 내부에서 50ns 동안 읽기 진행
                		CPU는 이때:
                				1 clock : 아직 안됐네
                				2 clock : 아직도 안왔네
                				3 clock : 아직도?
                				4 clock : 이제 왔네 
                		그제서야 다음 명령 실행
                				=> 그니까 좀 기다렸다가 읽어야 겠다 를 설정해야 함
                	wait-state = 3 (기본 1 클럭 + 3번 더 기다리셈)
                ```
                
        - Flash는 클럭이 빨라질 수록 바로 못 따라옴
            - (HSI) CPU 8MHz → wait-state 0 으로 설정
            - (PLL) CPU 72MHz → wait-state 2~3 으로 설정 (Flash가 CPU를 못 따라감)
            
            ```markdown
            FLASH->ACR = FLASH_ACR_LATENCY_2; // flash wait-state증가
            RCC->CR |= RCC_CR_PLLON;
            ```
            
        - Flash wait-state 먼저 올리고, 클럭 올려야 함 (거꾸로 하면 HardFault 직행 임)
        
        ※ AHB / APB 분주
        
        ※ (필요 시) VTOR 설정
        
    4. (필요 시) RAM 컨트롤러 초기화
        
        Cortex-M MCU는 보통 내부 SRAM이 있기 때문에 별도의 초기화 필요 없고, 바로 사용이 가능해서 .data복사 / .bss 초기화를 바로 할 수 있음 (그래서 필요 시 초기화 라고 하는것)
        
        - 왜 SRAM은 별도 초기화가 필요 없을까??
            - 내부 SRAM은 컨트롤러라 부를 만큼의 복잡한 컨트롤러가 없고
                
                전부 하드와이어링 되어있음
                
            
            reset 직후,
            
            - 이미 전원 연결되어서 SRAM 전원 이미 공급되어있음
                - CPU 다이 안에 SRAM에 같이 들어있어서 전원 도메인도 CPU랑 같고
            - 클럭 이미 연결됐고
            - SRAM이 주소 디코더랑 연결되어 있어서 이미 주소도 내부적으로 매핑 되어 있음
                - 주소 디코더는 CPU/인터커넥터 버스 내부에 들어있는 하드웨어 로직(블록)이다.
                - CPU는 메모리를 “주소 + 데이터 + 읽기/쓰기” 이렇게 본다.
                - 근데 이 주소가 Flash인지 SRAM인지 GPIO 인지는 모르기 때문에 중간에 주소 디코더가 필요하다.
                    
                    ```markdown
                    주소 디코더 내부 논리
                    if (주소가 0x2000_0000 ~ 0x2000_FFFF)
                        → 내부 SRAM enable
                    	   설계자가 주소 디코더를 0x2000_0000를 SRAM으로 배선해뒀다. 
                    else if (주소가 0x0800_0000 ~ ...)
                        → Flash enable
                    else if (주소가 0x4000_0000 ~ ...)
                        → Peripheral enable
                    
                    (실리콘 설계 단계에서 고정)
                    ```
                    
                    ```markdown
                    CPU
                     │  (주소 0x2000_1234)
                     ▼
                    주소 디코더
                     │
                     ├─ Flash 선택
                     ├─ SRAM 선택  ← 여기!
                     └─ Peripheral 선택
                    
                    ```
                    
            - 이미 주소가 매핑되있어서 CPU가 내부 SRAM 셀(0x2000_0000)에 바로 write 가능해짐
        - 링커 스크립트는  주소를 어떻게 정한거지?
            - M에서 “SRAM은 0x2000_0000부터 시작한다”는 게 사실상 고정 규칙이고, LENGTH는 MCU 마다 다르다
            
            ```markdown
            RAM (rwx) : ORIGIN = 0x20000000, LENGTH = 64K
            ```
            
        
        필요 시 RAM 초기화를 하게 되는 경우는 어떤게 있을까?
        
        Cortex-M + 외부 SDRAM이 있는 경우
        
        SDRAM : 타이밍 설정 필요, 초기화 시퀀스 필요 (그래서 클럭 설정 후 초기화 함)
        
        - SoC 내부에 SDRAM 컨트롤러 IP 가 있고, 그 IP에서 외부 SDRAM을 제어하는 구조
            
            ```markdown
            CPU
             └─ AXI/AHB
                 └─ SDRAM Controller (IP)
                     └─ 외부 SDRAM 칩
                     
            외부 SDRAM은 
            	row activate
            	precharge
            	refresh
            	CAS latency
            	Burst 길이 
            => 이걸 CPU가 제어 못해서 중간에 SDRAM 컨트롤러 IP두고 CPU 요청을 처리함
            ```
            
        - SDRAM 초기화 ⇒ SDRAM 컨트롤러 IP를 초기화 하는것
        - 컨트롤러 IP에
            - 클럭 주기, CAS Latency, burst 길이, refresh 주기 같은 타이밍/동작 파라미터 설정
            - 컨트롤러가 JEDEC 규칙에 맞춰 SDRAM 칩에 초기화 명령 시퀀스를 보내주는 구조
            
            ```markdown
            클럭 설정
            → SDRAM 컨트롤러 IP 설정
            → 컨트롤러가 SDRAM 칩 초기화
            → 그 다음에 RAM 사용 가능
            
            ```
            
        
    
    CPU 모드 개념이 없음 / MMU도 없음 ⇒ 설정 안 함
    
    Thread / Handler 모드는 하드웨어가 자동 관리 
    
    **A시리즈**
    
    ```markdown
    .global _start
    _start:
        // 여기서부터 Cortex-A 실행 시작
    ```
    
    Cortex-A의 startup코드에서 거의 고정인 패턴이 있는데
    
    ```markdown
    _start:
        // 1. CPU 모드 설정 (SVC/EL1 등)
        // 2. MMU OFF 상태 정리
        // 3. 스택 포인터 설정
        // 4. 예외 벡터 주소 설정
        // 5. ...
    ```
    
    1. CPU 모드 고정 설정
        
        Cortex-A는 모드/권한이 여러 개 있음 (나중에 어딘가에 정리할 예정)
        
        - ARMv7-A: User / SVC / IRQ / FIQ / Abort / Undefined
        - ARMv8-A: EL0 / EL1 / EL2 / EL3
        
        리셋 직후 상태는 SoC마다 다른데 Boot ROM 코드에서 어디까지 설정하고 넘겼는지를 startup에서는 모름, 그래서 _start에서 확실한 모드 하나로 통일하는데 
        
        ⇒ 보통 SVC 또는 EL1 임
        
        모드 설정 먼저 안 하면
        
        - 스택 레지스터가 다른 bank를 참조하거나,
        - privileged 명령이 실행 불가 하거나,
        - 예외 발생 시 다른 모드의 핸들러로 점프하게 됨(원하는 예외 처리가 안됨)
    2. MMU OFF 모드 설정
        
        MMU가 ON 되어 있으면 “주소 = 가상 주소”설정이 되어서 실제 접근 하는 물리 주소는 페이지 테이블에 따라 달라지게 됨
        
        근데 문제는, 
        
        - 페이지 테이블이 아직 없다는 것임..
        - 그리고 캐시/속성도 미정인 상태이고..
        
        이런 상태에서 스택 설정, 전역 변수 접근하면 이상한 주소에 접근하게 되어버림
        
        ```markdown
        // MMU, 캐시 전부 OFF
        mrc p15, 0, r0, c1, c0, 0     // read SCTLR
        bic r0, r0, #(1<<0)           // M (MMU) off
        bic r0, r0, #(1<<2)           // C (D-cache) off
        bic r0, r0, #(1<<12)          // I (I-cache) off
        mcr p15, 0, r0, c1, c0, 0     // write SCTLR
        isb
        ```
        
        MMU, 캐시 전부 OFF해서 “물리 주소 = 코드에 적힌 주소”가 되게 해 내가 예측이 가능한 상태를 확보하는 것임
        
    3. IRQ / FIQ / WDT(Watch dog) disable
        
        아직 스택 없음 / 벡터도 없음 이때 인터럽트 오면 바로 시스템 죽음
        
        그래서 초반에 disable 시킴
        
    4. 스택 포인터 설정 (임시 스택 설정 - On chip SRAM)
        
        Cortex-M에서 말했듯이 인터럽트/예외는 스택을 사용해서 벡터 점프 하니까 스택 먼저 설정
        
        ldr sp, =early_stack_top   // on-chip SRAM
        
    5. 예외 벡터 주소 설정
        
        네 번째로 예외 벡터 주소를 설정함
        
        Cortex-A는 예외 벡터를 하드웨어가 자동으로 안 잡아 줘서 
        
        VBAR(Vector Base Address Register)에 벡터 테이블 주소를 직접 설정한다.
        
        ```markdown
        ARMv7-A (32bit)기준
        
        벡터 테이블 준비
        .section .vectors, "ax"
        .global vector_table
        vector_table:
            b reset
            b undef
            b svc
            b pabort
            b dabort
            b .
            b irq
            b fiq
        
        이건 데이터가 아닌 실행 코드이고 어디에 있든 상관 없음
        ```
        
        ```markdown
        _start에서 VBAR 설정
        
        _start:
            ldr r0, =vector_table
            mcr p15, 0, r0, c12, c0, 0   // VBAR = vector_table
            isb
        
        "이 주소부터 예외 벡터 테이블임" 이라고 CPU에게 알려줌
        ```
        
        VBAR 이 설정된 이후부터 IRQ/Abort/FIQ가 정상 동작
        
        ```markdown
        ARMv8-A (AArch64)기준
        
        EL1에서 설정
        	
        	ldr x0, =vector_table
        	msr VBAR_EL1, x0
        	isb
        
        EL2/EL3면,
        	VBAR_EL2, VBAR_EL3임
        ```
        
    1. 시스템 클럭 / PLL / DRAM 컨트롤러 설정
        - CPU와 메모리 하드웨어를 “제대로 동작 가능한 속도/상태”로 만드는 단계
        - 리셋 직후 SoC 상태 ⇒ 살아는있는데 제대로 일은 못하는 이상한 상태
            - CPU 클럭 : 내부 RC / 저속 클럭으로 돌고 있음(24MHz, 32kHz등)
            - PLL : OFF
            - DDR 컨트롤러 : OFF 또는 Reset 상태
            - 버스 클럭 : 불명확 / 기본값
            
        - 클럭 설정 (CPU/버스/DDR이 서로 약속된 속도로 움직이게 기준을 맞추는 것)
            - CPU는 리셋 직후 아주 느린 기본 클럭으로 동작하며, 성능/타이밍도 엉성해서
                
                ⇒ 외부 클록을 선택해서 사용
                
                ⇒ PLL로 목표 CPU 주파수를 생성
                
                ⇒ 버스 클럭 분주 설정
                
                ```markdown
                [외부 기준 클록]
                        ↓
                      PLL
                        ↓
                  CPU / BUS / DDR
                ```
                
                [ 외부 기준 클럭 설정 ]
                
                - 외부 기준 클럭으로는  아래를 주로 선택한다.
                    - 외부 크리스탈 (24MHz, 25MHz 등)
                    - SoC 입력 클럭
                
                ⇒ 이 클럭을 설정 하는 이유
                
                PLL은 기준 주파수(reference)가 필요, 
                
                내부 RC는 오차가 커서 타이밍 불안정
                
                ```markdown
                CLK_SRC_SEL = XTAL_24MHZ
                ```
                
                [ PLL 설정 ]
                
                - PLL은 기준 클록을 원하는 고속 클록으로 증폭시키는 놈이라
                    - 입력 클럭 :24MHz
                    - 목표 CPU 클럭 : 1GHz
                    
                    ⇒ 이 경우 PLL 설정을 아래처럼 한다
                    
                    - PLL 파라미터 설정
                    - PLL ON
                    - PLL LOCK 을 기다리면서 대기 (안기다리면 클럭 흔들려서 CPU 오동작함)
                
                [CPU 클럭 소스 전환]
                
                - PLL이 안정화 되면 ⇒ CPU 속도 급상승, 성능 확보됨
                - 이 전에 Flash wait-state, SRAM/버스 타이밍 설정해두기( 안 맞으면 바로 크래시 발생)
                
                [버스 클럭 분주 (AHB/AXI/APB)]
                
                - CPU만 빠르면 되는게 아님
                - 주변 장치/DDR/인터커넥트는 CPU 만큼 빠르면 안됨
                - 분주기(divider) 설정으로 버스 클럭 분주 함
                    
                    ```markdown
                    CPU_CLK  = 1GHz
                    AXI_CLK  = 400MHz
                    AHB_CLK  = 200MHz
                    APB_CLK  = 100MHz
                    ```
                    
                - 이걸 안하면 버스 타이밍 깨지고, UART/TIMER 이상 동작해버림
                
                [DRAM 클럭 설정]
                
                - 클럭이 없으면 DDR은 그냥 벽돌임
                - 클럭 주파수가 타이밍 그 자체임
                - 설정하는 것들 :
                    - DDR 클럭 소스 설정(PLL에서 분기)
                    - DDR 주파수 설정(800MHz, 1600MT/s…)
                    - DDR 컨트롤러 타이밍 파라미터
            
            ```markdown
            타이머
            	- 타이머는 클록 기반
            	- 클록 바뀌면:
            			1ms 타이머가 10ms가 됨
            
            인터럽트
            	- GIC, 타이머 인터럽트 주기 전부 클록 의존
            	- 클록 미설정 → 인터럽트 폭주 or 무응답
            
            DDR
            	- DDR 타이밍은 클록에 절대 의존
            	- 잘못되면:
            			읽기 OK / 쓰기 깨짐
            			랜덤 크래시
            
            한 문장으로 정리
            	시스템 클록 / PLL / DRAM 설정이란
            	“CPU, 버스, 메모리가 같은 박자로 춤추게 만드는 작업”이다
            ```
            
        
        - DRAM 컨트롤러 설정 (하드웨어 단계)
            - 컨트롤러 IP를 설정하는 단계
            - Cortex-A는 외부 DDR이 기본이라 DRAM 컨트롤러가 DDR을 직접 제어하기 때문에 이 컨트롤러 IP를 설정 해줘야 함
                - DDR 종류 설정(DDR3 / DDR4 / LPDDR 등)
                - 타이밍 레지스터 설정
                    - CAS latency
                    - refresh 주기
                - DDR 클럭 소스 / 분주 설정
                - 초기화 시퀀스 트리거
                    - Precharge
                    - Mode Register Set
                    - Refresh enable
            - DRAM 컨트롤러 설정 단계에서 :
                - 컨트롤러 IP는 설정
                - DDR 클럭 / 타이밍 / 초기화 시퀀스 트리거 완료
            - 하지만 이 시점에서는 :
                - DDR이 정말 정상 동작하는지 확신 불가
                - 소프트웨어 입장에서는 아직 “위험한 메모리
            
            ⇒ 이걸 설정하면 컨트롤러는 준비되었으나, RAM이 정상인지 보장은 아직 없음
            
            컨트롤러 설정 실패 →  아예 접근 불가
            
    2. 메모리 초기화 (DDR 포함) (사용 준비 단계)
        
        “DDR 하드웨어는 켜졌고, 이제 소프트웨어가 RAM으로 믿고 써도 되는 상태를 만든다”
        
        위에 6번에서 DRAM 컨트롤러 IP 설정을 했으나 DDR 칩을 초기화 하지는 않았기 때문에 DDR 칩을 초기화 해 주는 작업을 해줘야 함
        
        startup 코드 7번에서 이 과정을 해준다고 보면 됨.
        
        - startup 코드의 역할 (중요 포인트)
            
            startup 코드는 DDR 초기화를 직접 구현하지는 않지만 아래 내용들을 진행한다. 
            
            1. DDR을 건드릴 수 있는 최소 실행 환경 확보
                - MMU OFF
                - 캐시 OFF
                - 코드 / 스택은 내부 SRAM에서 실행
            2. 전용 DDR 초기화 루틴 호출 (init 호출)
                - 보통 “SoC 벤더 제공 코드” 이거나,  “1차 부트로더(SPL, BL1 등)”
                
                이 루틴에서 DRAM 컨트롤러를 통해 DDR 칩에 JEDEC 초기화 명령 수행
                
        - DDR “칩 자체” 초기화를 할 때 실제로 일어나는 일은 뭘까??
            - 컨트롤러 IP에서 DDR 칩에게 명령을 보내는 과정
                - 전원 안정 대기
                - Precharge All (모든 뱅크 닫기)
                - Mode Register Set (CAS, burst length 등)
                - ZQ Calibration (온다이 저항 보정)
                - Refresh enable (데이터 유지 시작)
        - DDR 초기화 완료 이후에 간단한 메모리 테스트 (read/write 패턴) ⇒ 접근 안정성 확인
        - 메모리 초기화 실패 → 접근은 되는데 데이터 깨짐
        - 이 단계 완료 후 스택을 DDR로 이동 가능, .data ROM → RAM 복사, .bss 0 초기화, heap 사용… 등 소프트웨어적인 접근 가능
    
    
---

< A, M 동일 >

1. .data 세그먼트 ROM→RAM복사
    
    초기화 한 전역 변수/정적 변수는 ROM의 data 세그먼트에 저장된다. 
    
    근데 전역 변수나 정적 변수는 값이 변할 수 있기 때문에 변수를 write 가능한 RAM으로 복사한다. 
    
    ```markdown
    int a = 10;   // .data
    
    - ROM(Flash)에는 초기 값 10이 있음
    - 실행 중에는 RAM에서 read/write 해야함
    
    그래서 
    RAM의 data 영역으로 복사
    ```
    
    startup 예제 코드 (.data 세그먼트 ROM → RAM)
    
    ```markdown
    ARMv7-A (word 단위 복사)
    __data_load__ : ROM/Flash 쪽 .data 초기값 시작(LMA)
    __data_start__ : RAM에서 .data 시작(VMA)
    __data_end__ : RAM에서 .data 끝(VMA)
    
        .global copy_data
    copy_data:
        ldr   r0, =__data_load__    @ src (LMA: ROM/Flash)
        ldr   r1, =__data_start__   @ dst (VMA: RAM)
        ldr   r2, =__data_end__     @ end (VMA end)
    
        cmp   r1, r2
        bhs   2f                    @ if start >= end, nothing to copy
    
    1:  ldr   r3, [r0], #4          @ r3 = *src++; (word)
        str   r3, [r1], #4          @ *dst++ = r3;
        cmp   r1, r2
        blo   1b
    
    2:  bx    lr
    
    ```
    
    ```markdown
    ARMv8-A (8byte 단위 복사)
    __data_load__ : ROM/Flash 쪽 .data 초기값 시작(LMA)
    __data_start__ : RAM에서 .data 시작(VMA)
    __data_end__ : RAM에서 .data 끝(VMA)
    
        .global copy_data
    copy_data:
    		심볼 주소 준비
    		
    		// __data_load__ : ROM/Flash 쪽 .data 초기값 시작 주소(LMA)
    		// AArch64에서는 절대 주소를 한번에 못 만들기 때문에 
    			// adrp : 페이지 상위 52비트 가져오고
    			// add :lo12: : 페이지 안 오프셋(하위 12비트) 더함
    			// x0 == src (복사 시작지점)
        adrp  x0, __data_load__      // x0 = src page
        add   x0, x0, :lo12:__data_load__
    
    		// __data_start_ : RAM에서 .data 시작주소
    		// x1 = dst (복사 도착지점)
        adrp  x1, __data_start__     // x1 = dst page
        add   x1, x1, :lo12:__data_start__
    
    		// __data_end_ : RAM에서 .data 끝주소
    		// x1 = dst (복사 종료 기준)
        adrp  x2, __data_end__       // x2 = end page
        add   x2, x2, :lo12:__data_end__
    
    		// x1, x2비교 == .data 크기가 0이면 바로 종료 (2f로 점프)
        cmp   x1, x2
        b.hs  2f
    
    		// x0이 가리키는 곳에서 8byte 읽기
    		// 읽은 값 -> x3, 그 다음 x0 += 8
    1:  ldr   x3, [x0], #8           // 8 bytes copy
        str   x3, [x1], #8           // x3 값을 x1이 가리키는 RAM 주소에 저장, 그 다음 x1 += 8
    	  cmp   x1, x2                 // dst < end이면 다시 1:로 돌아감
        b.lo  1b
    
    2:  ret
    
    ```
    
2. .bss영역 0으로 초기화 
    
    링커에서 “이 구간은 .bss 다 RAM에는 이만큼의 bss 공간이 필요하다”알려주면
    
    startup 코드에서 그 공간을 전부 0으로 초기화 하는 것 
    
    - 초기화 되지 않은 전역 변수가 저장된 bss를 0으로 초기화 한다.
    
    ```markdown
    int b;   // .bss
    
    ROM에 값 자체가 없음 (공간 낭비라서)
    RAM의 bss 영역을 전부 0으로 초기화 (memset(0))
    ```
    
    ***startup 코드 예시***
    
    ```markdown
    ARMv7-A (4byte(word) 단위로 0 채움)
    심볼
    __bss_start__ : RAM에서 .bss 시작
    __bss_end__ : RAM에서 .bss 끝
     
        .global zero_bss
    zero_bss:
        ldr   r0, =__bss_start__   @ r0 = bss start
        ldr   r1, =__bss_end__     @ r1 = bss end
        mov   r2, #0               @ r2 = 0 (채울 값)
    
        cmp   r0, r1
        bhs   2f                   @ if start >= end, skip
    
    1:  str   r2, [r0], #4         @ *r0 = 0; r0 += 4
        cmp   r0, r1
        blo   1b
    
    2:  bx    lr
    
    ```
    
    ```markdown
    ARMv8-A (8 byte 단위로 0 채움)
    x0 : 현재 bss에 0을 쓸 주소 (포인터)
    x1 : bss 끝 주소
    x2 : 0 값 (xzr에서 옮겨둔 0)
    
        .global zero_bss
    zero_bss:
        adrp  x0, __bss_start__
        add   x0, x0, :lo12:__bss_start__   // x0 = bss start
    
        adrp  x1, __bss_end__
        add   x1, x1, :lo12:__bss_end__     // x1 = bss end
    
        mov   x2, xzr                        // x2 = 0
    
        cmp   x0, x1
        b.hs  2f
    
    		// x2에 있는 값(0)을 x0이 가리키는 메모리에 저장하고, x0 = x0 + 8
    1:  str   x2, [x0], #8                   // *x0 = 0; x0 += 8
        cmp   x0, x1                // 현재 포인터 x0과 끝주소 x1 비교
        b.lo  1b                    // lo = unsigned lower 
                                    // x0 < x1이면 1이라는 이전 라벨로 점프
    
    2:  ret
    
    ```
    
3. stack 생성
    
    링커가 잡아준 스택 영역의 top 주소를 CPU의 SP레지스터에 넣음
    
    Cortex-M
    
    - Reset 때 **MSP는 벡터 테이블[0] 값으로 자동 설정**
    - 보통 벡터 테이블 첫 엔트리에 `__StackTop` 같은 심볼이 들어있음
    - (추가로 PSP 쓸 거면 따로 설정)
    
    Cortex-A
    
    - 모드가 여러 개라서
        - SVC/IRQ/FIQ/ABT/UND 각각의 `sp`를 직접 넣어줌
    - 그래서 startup에서 모드 바꿔가며 `ldr sp, =xxx_stack_top` 해줌
    1. 모드 별 스택 생성 (IRQ/FIQ/ABT/UND등) 
        
        CPU 모드가 여러 개 있는데, ARMv7-A 기준 아래와 같음
        
        - **SVC** : 일반 커널 실행
        - **IRQ** : 일반 인터럽트
        - **FIQ** : 빠른 인터럽트
        - **ABT** : 데이터/명령 abort
        - **UND** : 미정의 명령
        - (User 모드는 보통 부트 초기에 안 씀)
        
        이 모드들은 각자 자기 전용 SP(Stack Pointer)를 가진다.
        
        | 모드 | 사용하는 SP |
        | --- | --- |
        | SVC | `SP_svc` |
        | IRQ | `SP_irq` |
        | FIQ | `SP_fiq` |
        | ABT | `SP_abt` |
        | UND | `SP_und` |
        
        리셋 직후 Boot ROM에서 어떤 모드를 넘겼는지 모르는 상태라 각 모드의 SP값은 
        
        ⇒ “초기화 안 됐거나, 쓰레기 값이 가능성이 크다”
        
        모드 별 스택 생성이 뭘 하는 건가?
        
        ⇒ 각 CPU 모드로 잠깐 들어가서, 그 모드의 SP를 지정해 주는 것
        
        ```markdown
        // IRQ 모드 스택
        msr CPSR_c, #IRQ_MODE | I_BIT | F_BIT
        ldr sp, =irq_stack_top
        
        // FIQ 모드 스택
        msr CPSR_c, #FIQ_MODE | I_BIT | F_BIT
        ldr sp, =fiq_stack_top
        
        // ABT 모드 스택
        msr CPSR_c, #ABT_MODE | I_BIT | F_BIT
        ldr sp, =abt_stack_top
        
        // UND 모드 스택
        msr CPSR_c, #UND_MODE | I_BIT | F_BIT
        ldr sp, =und_stack_top
        
        // 다시 SVC 모드
        msr CPSR_c, #SVC_MODE | I_BIT | F_BIT
        ldr sp, =svc_stack_top
        ```
        
        왜 모드 별 스택을 나눠서 쓸까?

        | 항목 | 설명 |
        | --- | --- |
        | 예외 중첩 안정성 | IRQ 처리 중에 Data Abort 발생 가능성이 있어서, 각자 스택을 쓰면서 서로 침범이 없게 함.<br><br>※ 예외 발생 시 CPU가 하는 일<br>정상 실행 중 (SVC 모드):<br>&nbsp;&nbsp;SP = SP_svc<br><br>IRQ 발생 시, CPU가 자동으로 수행:<br>&nbsp;&nbsp;CPSR → SPSR_irq<br>&nbsp;&nbsp;PC → LR_irq<br>&nbsp;&nbsp;모드 전환: SVC → IRQ<br>&nbsp;&nbsp;SP 레지스터 전환: SP_svc → SP_irq (SP = SP_irq)<br><br>IRQ 핸들러 진입 후 보통 하는 일:<br>&nbsp;&nbsp;irq_handler:<br>&nbsp;&nbsp;&nbsp;&nbsp;sub sp, sp, #64<br>&nbsp;&nbsp;&nbsp;&nbsp;stmfd sp!, {r0-r12, lr}<br>&nbsp;&nbsp;r0~r12 (작업용 레지스터) push, lr(복귀 주소) push<br><br>IRQ 처리 중 Data Abort 발생 시, ABT 진입 과정:<br>&nbsp;&nbsp;현재 PC → LR_abt<br>&nbsp;&nbsp;CPSR → SPSR_abt<br>&nbsp;&nbsp;모드 전환: IRQ → ABT<br>&nbsp;&nbsp;SP 레지스터 전환: SP_irq → SP_abt (SP = SP_abt)<br>&nbsp;&nbsp;ABT 핸들러 시작:<br>&nbsp;&nbsp;abt_handler:<br>&nbsp;&nbsp;&nbsp;&nbsp;sub sp, sp, #64<br>&nbsp;&nbsp;&nbsp;&nbsp;stmfd sp!, {r0-r12, lr}<br><br>&lt;스택 메모리 구조&gt;<br>RAM 상에서:<br>&nbsp;&nbsp;[ SVC stack ]  ← SP_svc<br>&nbsp;&nbsp;[ IRQ stack ]  ← SP_irq (IRQ 진입 시 push는 여기에서만 발생)<br>&nbsp;&nbsp;[ ABT stack ]  ← SP_abt (ABT 진입 시 push는 여기에서만 발생)<br><br>만일 모든 모드가 SP 하나를 쓴다면?<br>&nbsp;&nbsp;IRQ 진입 → push (복귀에 필요한 LR, CPSR, 레지스터들)<br>&nbsp;&nbsp;IRQ 처리 중 ABT 발생 → push<br>&nbsp;&nbsp;ABT 복귀 → pop<br>&nbsp;&nbsp;IRQ 복귀 → pop<br><br>SP 위의 스택 프레임 예시:<br>&nbsp;&nbsp;IRQ r0, IRQ r1, ... , IRQ lr<br>&nbsp;&nbsp;그 위에 ABT r0, ABT r1, ... , ABT lr 이 덮어써짐<br><br>⇒ 스택 프레임이 섞여서 복귀 주소가 깨질 수 있고, 결국 복귀 불가(Crash) 상황이 발생할 수 있다. |
        | 디버깅/안정성 | Abort 스택이 따로 있으면, 메모리 오류 시 최소한의 상태 보존이 가능하다. |
        | 커널 설계 | 실제 OS에는 스택 분리가 필수이다.<br>SVC = 커널 모드<br>IRQ/FIQ = 인터럽트 컨텍스트 |
        
        | 구분 | Cortex-M | Cortex-A |
        | --- | --- | --- |
        | CPU 모드 | Thread / Handler | SVC/IRQ/FIQ/ABT/UND |
        | 스택 | MSP / PSP | **모드별 SP 전부 다름** |
        | 스택 설정 | 하드웨어가 일부 자동 | **소프트웨어가 전부 설정** |
        | 안 하면 | 거의 없음 | 인터럽트 즉사 |
3. heap 생성
    
    링커 스트립트에서 “.bss 끝 이후부터 스택 시작 전까지” 이런 식으로 heap 후보 구간을 정함
    
    런타임(newlib 같은 libc)에서 
    
    - `sbrk()` / `_brk` 포인터를 움직이며 `malloc()`에 메모리를 공급
4. 인터럽트 사용가능(enable)로 설정
    - VBAR 설정
    - 스택(모드별 최종 스택까지)
    - GIC 초기화 + 타이머/소스 설정
    
    CPU레벨에서 enable 한다고 인터럽트가 바로 들어오는게 아니라 보통 GIC/Timer 설정까지 끝나야 진짜 IRQ가 들어오기 시작함
    
5. main()으로 분기
    
    IRQ/FIQ enable + main분기
    
    ```markdown
     ARMv7-A (A32)
     
        .global _start
    _start:
        /* 0) 가장 먼저: 인터럽트 잠그기 (안전) */
        cpsid   if              @ I=1(IRQ off), F=1(FIQ off)
    
        /* 1) (여기서) 임시 스택, VBAR, DDR init, data/bss, 모드별 스택 등 완료했다고 가정 */
    
        /* 2) GIC / 타이머 같은 인터럽트 컨트롤러 초기화 (보통 C 함수로) */
        bl      gic_init        @ (선택) Distributor/CPU IF enable, priority mask, routing 등
        bl      timer_init      @ (선택) 타이머 인터럽트 설정
    
        /* 3) 이제 CPU 레벨에서 IRQ enable (필요하면 FIQ도) */
        cpsie   i               @ I=0 → IRQ enable
        /* cpsie f */           @ FIQ까지 쓰면 enable (보통은 IRQ만 먼저)
    
        /* 4) main()으로 분기 */
        bl      main
    
    hang:
        b       hang
    
    cpsid if : 안전하게 시작(초반엔 IRQ 오면 죽으니까)
    gic_init/timer_init : “인터럽트가 들어올 준비” 만들기
    cpsie i : 마지막에 CPU가 IRQ를 받아들이도록 허용
    bl main : main 호출
    ```
    
    ```markdown
    ARMv8-A (AArch64)
    	- cpsie 대신 DAIF 플래그 사용
    	
        .global _start
    _start:
        /* 0) 가장 먼저: 인터럽트 잠그기 (안전) */
        msr     DAIFSet, #0xf       // D,A,I,F 모두 mask (디버그/SError/IRQ/FIQ)
    
        /* 1) (여기서) 임시 스택, VBAR_ELx, DDR init, data/bss, 모드별 스택 등 완료했다고 가정 */
    
        /* 2) GIC / 타이머 초기화 */
        bl      gic_init
        bl      timer_init
    
        /* 3) CPU 레벨에서 IRQ enable (I 비트만 클리어) */
        msr     DAIFClr, #0x2       // I 마스크 해제 → IRQ enable
        /* 필요하면 FIQ도:
           msr  DAIFClr, #0x1       // F 마스크 해제 → FIQ enable
        */
    
        /* 4) main()으로 분기 */
        bl      main
    
    hang:
        b       hang
    
    ```

