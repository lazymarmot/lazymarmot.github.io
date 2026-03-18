---
layout: post
title: "코어 설계란?"
date: 2026-03-10 12:20:00 +0900
categories: RISC-V
show_on_home: false
---

# 코어 설계란?

코어 설계 = CPU 내부를 만드는 것

> “ISA를 실제 하드웨어로 구현하는 것”
> 

- 명령어를 어떻게 해석할지
- 어떤 순서로 명령어를 실행할지
- 레지스터/파이프라인을 어떻게 구성할지

RISC-V 코어 설계 = ISA 선택 → 실행 방식 결정 → 파이프라인 구성 → 유닛 붙이기 → 검증

![image.png](/assets/posts_image/RISC-V/Core_Design/image.png)

개발 목적에 따라 코어 구조가 완전히 달라지기 때문에 코어를 설계해서 만든다

저전력 MCU용? / Linux 돌릴 메인 CPU? / AI 가속기 옆 컨트롤 코어?

<br>

### 코어 설계 단계

1. ISA 선택
    
    ```c
    **RV64IMAC**
    ```
    
    > ISA 선택 = 명령어 구조를 어떻게 가져갈지 (ISA = 코어의 스펙서)
    > 
    > 
    > RV64 : 64비트
    > 
    > I : 기본 정수
    > 
    > M : 곱셈/나눗셈
    > 
    > A : atomic
    > 
    > C : compressed
    > 
    
    설명 : 
    
    - **RV64**
        
        ❌ 64비트 *명령어*
        
        ⭕ **64비트 레지스터 / 주소 공간**
        
    - **I**
        
        ⭕ ADD, SUB, LOAD, STORE 같은 **기본 정수 명령**
        
    - **M**
        
        ⭕ MUL, DIV 같은 **정수 연산 확장**
        
    - **A**
        
        ⭕ atomic (Linux에 중요)
        
    - **C**
        
        ⭕ 16bit compressed (코드 사이즈 줄임)
        
    
    ISA 선택은:
    
    > “CPU가 이해해야 할 ‘명령어 집합의 계약서’를 정하는 것”
    > 

1. 실행 방식 결정 (in-order / out-of-order)
    
    in-order의 정확한 의미
    
    > 명령어를 “순서대로 가져와서, 순서대로 끝낼 때까지 기다리는 방식”
    > 
    
    ```c
    lw x1, 0(x2)
    add x3, x1, x4
    ```
    
    in-order CPU에서 일어나는 일
    
    1. `lw` 실행
    2. 메모리에서 데이터 올 때까지 **대기**
    3. `add`는 **기다림** (앞 명령 끝나야 실행)
    
    👉 **뒤에 있는 명령이 앞 명령을 절대 추월하지 못함**
    
    “in-order = **앞 명령이 느리면, 뒤 명령도 같이 멈춘다”** 
    
    ⇒ 구조 단순 /  전력 적음 / 성능 제한적
    

1. 파이프라인단계 정하기
    
    “명령어 실행을 몇 개의 하드웨어 단계로 나눌지 정하는 것”
    
    대표적인 기본 파이프라인
    
    ```c
    IF  →  ID  →  EX  →  MEM  →  WB
    
    IF : 명령어 가져오기
    ID : 해석
    EX : 연산
    MEM : 메모리 접근
    WB : 결과 저장
    ```
    
    사이클은 언제 정해지나?
    
    단계 수를 정하면 각 단계가 1 사이클을 차지하도록 설계 하는게 보통
    
    ```c
    MEM1 → MEM2
    
    이렇게 하면 메모리 접근이 2사이클짜리 구조가 됨
    ```
    

1. **내부 유닛 설계**
    
    Fetch / decode / ALU / LSU 같은 유닛들이 CPU 코어 내부에서 어떻게 연결되고, 어떤 순서/규칙으로 동작하는지 마이크로아키텍처 설계서를 만드는 것
    
    코어 그 자체를 구성하는 내부 블록들을 정의하는 단계
    
    **4-1) Fetch Unit (명령어 가져오기)**
    
    **역할**
    
    - 다음에 실행할 PC(Program Counter)를 만들고
    - I-cache(또는 메모리)에서 **명령어 바이트를 가져와서** Decode로 넘김
    
    **설계에서 결정하는 것**
    
    - PC 증가 규칙: 보통 +4 (C 확장 있으면 +2도 가능)
    - fetch 폭: 한 사이클에 1개? 2개? (superscalar면 더)
    - I-cache miss 시 stall 정책
    - branch/jump 발생 시 **fetch flush**(잘못 가져온 명령어 버리기) 방식
    
    **예시**
    
    - “Compressed(C) 지원하면 fetch가 어려워짐”
        - 16/32비트가 섞이니까 “다음 명령어 경계”를 잘 맞춰야 함
    
    ---
    
    **4-2) Decode Unit (명령어 해석)**
    
    **역할**
    
    - 32비트(또는 16비트) 명령어를 뜯어서
        - opcode / rd / rs1 / rs2 / funct3 / funct7 / imm
    - “이게 ADD인지, LW인지, BRANCH인지” 결정
    - 컨트롤 신호(ALU가 뭐 할지, reg write 할지 등) 생성
    
    **설계에서 결정하는 것**
    
    - 디코딩 테이블(매핑) 구조: hardwired? microcode? (RISC-V는 보통 hardwired)
    - 즉시값(imm) sign-extend 규칙 정확히
    - illegal instruction 처리 (지원 안 하는 확장 명령이 들어왔을 때 trap)
    
    **예시**
    
    - RV64I만 지원하는 코어에 `MUL`이 들어오면?
        - Decode에서 illegal로 보고 **trap(예외)** 발생시킴
    
    ---
    
    **4-3) Register File (레지스터 파일)**
    
    **역할**
    
    - x0~x31 일반 레지스터 값 저장
    - 매 사이클 rs1/rs2 읽고 rd에 쓰기
        - Decode가 : “이번 명령은 rs1 = x5, rs2 = x7에 써라” 라고 알려주면
        - register file이 : x5, x7 값을 동시에 꺼내서 ALU / LSU에 전달
        - 보통 읽기 포트(rs1, rs2)가 있고, 쓰기 포트(rd)가 있음
    
    **설계에서 결정하는 것**
    
    - 포트 수: (읽기 2, 쓰기 1)이 기본
        - superscalar면 읽기/쓰기 포트 늘어나서 면적/전력 급증
    - write-back 타이밍: 같은 사이클에 읽고 쓰면 “읽는 값이 새 값인지” 규칙 필요
    - x0는 항상 0 유지(쓰기 무시)
    
    **실수 포인트**
    
    - “동일 레지스터에 동시에 쓰기” 같은 케이스의 우선순위 규칙이 없으면 버그 지옥
    
    ---
    
    **4-4) ALU (정수 연산 유닛)**
    
    **역할**
    
    - decode된 명령을 받아서 레지스터 값들을 갖고 실제 비트 연산 / 산술 연산을 수행하는 블록
    - ADD/SUB/AND/OR/XOR/SHIFT/비교(SLT) 등 실행
        - decode 가  :  “이 명령은 ADD, AND, SHIFT이다” 라고 지시
        - Register File이  :  “연산에 쓸 값 rs1, rs2” 주면
        - ALU가 실제로 더하고, 빼고, 비트 AND/OR/XOR하고, shift하고, 비교해서 결과 만듬
    
    **설계에서 결정하는 것**
    
    - 단일 사이클로 끝낼지, 멀티사이클로 할지
    - shift 같은 연산을 1사이클로 할지, 반복 시프트로 할지(저면적 코어에서 고민)
    - M 확장(MUL/DIV)을 ALU에 붙일지 별도 유닛으로 둘지
    
    **예시**
    
    - MCU급 초저면적 코어:
        - DIV는 멀티사이클(32~64cycle)로 가는 경우 흔함
    
    ```c
    add x3, x1, x2
    
    CPU 내부에서:
    	Decode
    		→ rs1 = x1, rs2 = x2, rd = x3
    	
    	Register File
    		→ x1 값, x2 값 꺼내줌
    	
    	ALU
    		→ 두 값 더함
    	
    	Register File
    		→ 결과를 x3에 저장
    ```
    
    ---
    
    **4-5) Load/Store Unit (LSU)**
    
    레지스터 ↔ 메모리 사이를 택배 기사? 라고 보면 될듯하다
    
    메모리는 느리고 / 버스는 복잡 / load/store는 예외가 많이 생김 (miss, fault등)
    
    그래서 ALU에서 분리해서 메모리 전용 처리 유닛으로 둠
    
    **역할**
    
    - 주소 계산 (rs1 + imm)
        - lw x1, 8(x2)
        Register File에서 x2값 받음, 즉시 8을 더함 (실제 메모리 주소 : x2 + 8)
        이 계산은 보통 LSU안의 주소 계산 로직에서 진행
    - 메모리 읽기/쓰기 트랜잭션 발행
        - 계산된 주소로
            - load : “읽어와”
            - store : “써”
            
            SoC 버스 (AXI등)에 트랜잭션 요청 보냄
            
    - 데이터 크기 & 정렬 처리
        - RISC-V load/store는 크기가 다름
            - LB / LH / LW / LD
            - SB / SH / SW / SD
        - 몇 바이트 쓸지
        - 읽어온 데이터 중 어느 바이트가 유효한지
        - sign-extend 할지 / zero-extend 할지
    - misaligned(비정렬 접근) 처리 정책
        - lw x1, 1(x2)
        주소가 4바이트 정렬이 아닐 경우 정책 정함
            
            ⇒ trap 발생 하지 말고(illegal/misaligned access), 두 번 읽어서 내부에서 조합해
            
    
    **설계에서 결정하는 것**
    
    - 메모리 응답이 늦을 때 stall/버퍼링 방식
    - store buffer 둘지 말지 (성능 vs 복잡도)
    
    **예시**
    
    - `lw x1, 1(x2)` 처럼 주소가 4바이트 정렬이 아니면?
        - (A) trap 발생
        - (B) 두 번 읽어서 합쳐서 제공
            
    
    ---
    
    **4-6) CSR Unit (Control and Status Registers)**
    
    **역할**
    
    - CPU의 상태 + 관리자 콘솔”
    - ALU가 계산기라면, CSR은 CPU가 지금 어떤 상태인지, 앞으로 어떻게 행동 할 지를 정하는 중앙 제어실
    - 코어의 상태/제어 레지스터들 관리 / 예외/인터럽트/권한모드 전환과 직결
        - CPU자체를 제어하고 상태를 기록하는 특수한 레지스터
        - 지금 CPU가 어떤 모드(M/S/U)에 있는지
        - 인터럽트가 켜져 있는지
        - 방금 난 예외가 뭐였는지
        - 다음에 어디로 trap handler로 점프할지
    - 세부 설명
        - CPU 상태 항상 기록
            - 현재 privilege mode(M/S/U)
            - 인터럽트 enable 상태
            - 마지막 예외 원인
            - 예)
                - mstatus / sstatus → CPU 상태 비트들
                - mcause / scause → 왜 Trap이 발생했는지
                - mepc / sepc → trap 터진 C
        - 예외 / 인터럽트 발생 시 “자동 동작”
            - 어떤 명령에서 illegal instruction / page fault / timer interrupt 발생
                - CSR Unit이 자동으로 처리
                    - 현재 PC를 mepc/sepc에 저장
                    - 원인을 mcause/scause에 기록
                    - privilege mode 변경 (U→S / S→M등)
                    - mtvec/stvec에 설정된 주소로 점프
        - CSR 명령 수행 (RISC-V에는 전용 CSR명령 있음)
            
            ```c
            csrrw x1, mstatus, x2
            csrrs x0, mie, x3
            
            CSR Unit은 여기서:
            	접근 권한 검사 (U/S/M에서 가능한지)
            	비트 set/clear 규칙 적용
            	읽기/쓰기 결과 반환
            
            만일 잘못 구현하면:
            	OS가 CSR 쓰다가 바로 죽음
            	Linux 부팅 안 됨
            ```
            
    
    **설계에서 결정하는 것**
    
    - 어떤 CSR들을 지원할지(특히 privileged 쪽)
    - 접근 권한: U/S/M 모드에서 읽기/쓰기 가능 여부
    - CSR 명령(CSRRW/CSRRS/CSRRC) 동작 정확성
    
    **예시**
    
    - OS가 기대하는 CSR이 빠져있으면 Linux 부팅이 안 됨
    
    ---
    
    **(선택) Branch Predictor (분기 예측기)**
    
    **역할**
    
    - “분기할까 말까”를 미리 맞춰서 fetch를 앞당김
    
    **왜 필요한가**
    
    - 파이프라인이 깊어질수록 분기 미스 비용↑
        
        (미스하면 fetch한 것 다 버리고 다시 시작)
        
    
    **단순한 1-bit 예시**
    
    - 지난번에 Taken이면 다음도 Taken이라고 예측
    - 문제: `T N T N ...` 같은 패턴에서 계속 틀림
    
    현실적으로는
    
    - 2-bit saturating counter(가장 흔함)
    - BTB(분기 타겟 캐시) + BHT(히스토리) 조합
    
    ---
    
    **(선택) MMU / PMP**
    
    - **MMU**: 가상주소→물리주소 변환 + 페이지 보호 (Linux 필수)
    - **PMP**: 물리주소 범위 보호(간단한 메모리 보호) (MCU/펌웨어에서 자주)
    
    즉,
    
    - “MMU 없이 PMP만” = Linux는 어렵고, RTOS/베어메탈은 가능

    <br>

1. 메모리 모델 & 인터페이스 
    
    5-1) 캐시 유무 결정
    
    캐시 없음 (MCU급)
    
    - 장점: 단순, 예측 가능(리얼타임), 검증 쉬움
    - 단점: 메모리 느리면 성능 바로 박살
    
    캐시 있음 (Linux/고성능)
    
    - I-cache / D-cache
    - miss 처리, refill, write-back/through 정책 등 설계 요소 폭증
    
    ---
    
    5-2) L1 I/D 분리 여부 결정
    
    **Harvard(I/D 분리)**: 동시에 명령 fetch + 데이터 load 가능 → 성능↑
    
    **Unified**: 단순하지만 병목 생김
    
    대부분 CPU는 L1에서 I/D 분리 많이 함.
    
    ---
    
    5-3) 인터페이스: AXI / TileLink 같은 “버스 프로토콜” 결정
    
    코어 입장에선:
    
    - LSU가 “메모리 읽어!”라고 요청하면
    - 그걸 SoC interconnect가 이해하는 프로토콜로 내보내야 함
    
    AXI 선택하면
    
    - AR/AW/W/R/B 채널 핸드셰이크
    - outstanding 트랜잭션 수(동시에 몇 개 요청할지)
    - reorder 허용/불허 정책
    
    TileLink 선택하면
    
    - (오픈 생태계에서 많이 쓰는) 일관성/캐시 프로토콜 확장이 쉬움
        
        (단, 네가 연결할 SoC 환경에 따라 결정)
        
    
    ---
    
    5-4) “메모리 모델”이란 말의 의미
    
    - 하드웨어 관점: 캐시/버스/리오더가 “메모리 접근 순서”에 어떤 영향을 주는가
    - ISA 관점: fence 같은 명령으로 순서를 어떻게 보장하는가
    
    즉, “코어가 load/store를 어떤 순서로 밖에 내보낼 수 있는가”도 설계 포인트.
    
    <br>

1. 예외 / 인터럽트 / privilege 설계 (OpenSBI / OS직결 단계)
    
    RISC-V는 크게 아래 3단계가 “코어가 세상을 나누는 방법”임
    
    - **U-mode**: 유저 앱
    - **S-mode**: OS 커널(Linux)
    - **M-mode**: 머신/펌웨어(OpenSBI 같은 것)
    
    <br>

    **6-1) trap이 뭐냐** 
    
    trap : ARM에서 말하는 exceptio 이라고 보면 될 듯
    
       “정상 실행을 멈추고, 정해진 핸들러로 점프”
    
    ARM 
    
    - exception : 
    
        Synchronous exception (undef, data abort, SVC 등)
    
        Asynchronous exception(IRQ, FIQ)
    
    - RISC-V
    
    trap :
    
        Exception(동기적) : illegal instruction, page fault, ecall
    
        Interrupt(비동기적) : timer, external interrupt
    
    trap 원인 예시:
    
    - illegal instruction
    - page fault (MMU 있을 때)
    - ecall (시스템콜)
    - timer interrupt / external interrupt
    
    ---
    
    6-2) trap 진입 방식에서 코어가 해야 하는 일
    
    trap이 걸리면 코어는 최소한 이걸 자동으로 해야 함:
    
    1. **어디서 터졌는지 저장**
        - `mepc` / `sepc` : trap 발생 PC 저장
    2. **왜 터졌는지 저장**
        - `mcause` / `scause` : 원인 코드
        - `mtval` / `stval` : fault 주소 같은 부가 정보
    3. **권한 모드 변경 + PC 변경**
        - `mtvec` / `stvec` : trap handler 시작 주소로 점프
    4. **인터럽트 enable 비트 저장/갱신**
        - `mstatus` / `sstatus`의 MIE/SIE 관련 비트 처리
    
     이 시퀀스가 제대로 안 되면 OS/부트로더가 코어를 신뢰 못 해서 바로 죽음…
    
    ---
    
    6-3) CSR 맵 설계
    
    CSR 맵 설계 : 이 CPU가 ‘어떤 기능을 갖고 있는지’ OS/펌웨어에게 공식적으로 알려주는 설계
    
    (= CPU의 자기소개서 라고 생각하면 됨)
    
    RISC-V privileged 스펙에 “표준 CSR 주소”가 있음
    
    ```c
    0x301 → misa
    0x180 → satp
    0x304 → mie
    0x344 → mip
    
    이 주소공간 전체를 통틀어 CSR 맵이라고 부름
    ```
    
    코어 설계자는:
    
    - 그 CSR을 **정말 구현할지 / 말지 결정**
    - 구현한다면 **어떤 비트가 실제로 동작 하는지**를 결정해.
        - 각 CSR레지스터 안에 정의된 비트 하나하나를 코어 설계자가 “이 기능 지원함/안함”에 맞게 하드웨어로 정해 놓음 (코어가 태어날 때부터 그렇게 보이도록 고정해 두는것임)
            
            예:
            
            - `misa`: 어떤 확장(IMAFDC 등)을 지원한다고 보고할지
                - bit 8 → I / bit 12 → M / bit 0 → A / bit 2 → C ….
                - 코어 설계자는 아래처럼 세팅해야 함
                    - M 확장은 지원하니까 bit 12는 1
                    - FPU 없으니까 F/D 비트는 0
            - `satp`: MMU(페이지 테이블) 지원이면 필수
            - `mie/mip`: 인터럽트 enable/pending 관리
    
    OS는 부팅하면서 코어를 탐색하는데
    
    1. OS가 misa 읽음 → “아, RV64 + IMAC 지원하넹?”
    2. satp CSR 접근해봄
        1. 읽힘 → MMU가 있구나
        2. trap 발생 → MMU가 없네
    3. 2번에 맞춰서
        1. 페이지 테이블 enable 할지
        2. atomic 쓸지
        3. FPU 컨텍스트 저장할지
    
    ⇒ 결국 OS는 CSR만 보고 CPU의 능력을 판단하게 됨
    
    예)
    
    satp 레지스터
    
    ```c
    Core입장 
    	MMU 구현했음
    	page table walker 있음
    
    -> satp CSRP을 구현해야 함
    ```
    
    ```c
    OS입장
    	csrw satp, x1
    		이게 정상 동작 하면 -> "이 CPU는 가상 메모리를 지원하네"
    		trap -> "이 CPU는 베어메탈/RTOS용 CPU구만?"
    ```
    
    mie / mip 레지스터
    
    - mie : 인터럽트 허용 스위치
    
    - mip : 인터럽트 들어왔는지 표시
    
    - OS / 펌웨어는
    
        - 타이머 인터럽트 켜고
    
        - 외부 인터럽트 받고
    
        - IPI처리 함 
    
        ⇒ CSR들이 제대로 안 설정되면 타이머, 스케줄러, 멀티코어 동작 안됨
    
    ---
    
    6-4) interrupt latency는 뭘로 결정되냐
    
    인터럽트가 들어왔는데 “몇 사이클 후에 핸들러 첫 명령어 실행되냐”가 레이턴시야.
    
    이 레이턴시는 보통:
    
    - 파이프라인 깊이 (flush 비용)
    - 현재 명령이 trap에 “즉시 반응 가능”한 지점인지
    - 메모리 트랜잭션 대기 중인지
    - 인터럽트 마스킹 상태
    
    에 의해 결정돼.
    
    **in-order + 단순 파이프라인**은 보통 레이턴시 예측이 쉬움.
    
    OoO는 더 복잡해져(리오더 버퍼 비우기 등).
    
    ---
    
    6-5) OpenSBI / OS랑 직결되는 이유 (진짜 핵심)
    
    Linux 같은 OS는 보통 S-mode에서 돌고,
    
    M-mode에는 OpenSBI가 “중간 관리자”로 있음.
    
    예:
    
    - 타이머 인터럽트 설정
    - IPI(코어간 인터럽트)
    - 부팅 시 초기화/환경 제공
    
    즉,
    
    - **M-mode trap/CSR 구현이 허술하면 OpenSBI가 못 돌아가고**
    - OpenSBI가 못 돌면 Linux 부팅이 막힘

    <br>

1. RTL 작성 (Verilog / SystemVerilog)
    
    이제야 **코딩 시작** 
    
    - fetch.sv
    - decode.sv
    - alu.sv
    - lsu.sv
    
    ⇒ 여기까지 와야 “코어 하나 만들었다”고 말함
    
    <br>

1. 검증 (코어 설계의 절반)
    
    필수
    
    - ISA compliance test
    - directed test
    - random test
    
    고급
    
    - formal verification
    - pipeline hazard 검증
    
    검증이 안 되면: “돌아는 가는데 믿을 수 없는 CPU” 라고 여겨지게 됨
    
    <br>

1. (선택) 성능 튜닝
    - IPC 측정
    - stall 원인 분석
    - branch miss 줄이기

