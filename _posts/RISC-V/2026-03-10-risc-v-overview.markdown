---
layout: post
title: "RISC-V"
date: 2026-03-10 12:30:00 +0900
categories: RISC-V
show_on_home: false
---

# RISC-V

## RISC-V 하드웨어 플랫폼 용어

RISC-V 하드웨어 플랫폼은 하나 이상의 **RISC-V 호환 프로세싱 코어**를 포함할 수 있으며, 이와 함께 **RISC-V가 아닌 코어**, **고정 기능 가속기**, **다양한 물리적 메모리 구조**, **I/O 디바이스**, 그리고 이 구성 요소들이 서로 통신할 수 있도록 하는 **인터커넥트 구조**로 이루어질 수 있다.

쉽게 말해

> “RISC-V 코어만 있는 단순한 칩일 수도 있고, ARM 코어나 DSP, NPU 같은 애들이 **섞여 있는 SoC**일 수도 있다는 뜻”
> 

### 시스템 레벨 구조

“RISC-V는 작든 크든, 구조 생각하는 방식이 똑같다.”

아래 구조들 모두 전부 같은 RISC-V 철학으로 설계 가능하다는 뜻 ⇒ 명령어 체계(ISA)는 동일하다

- **아주 작은 MCU** (센서 하나 달린 칩)
    - CPU 1개,  메모리 작음, bare-metal 또는 간단한 RTOS
    - 센서 읽고 → 제어 신호 보내는 용도
- **중간급 SoC** (자동차, 로봇, 엣지 AI)
- **데이터센터 서버** (코어 수백~수천 개) - 엄청 복잡한 구조
    - CPU 수백~수천 개, 공유 메모리
    - Linux / 가상화 / 보안 영역
    - AI·데이터 처리

Hart와 Coprocessor는 CPU 코어 개념에 포함되고

Accelerator는 SoC레벨에서 CPU와 병렬로 존재하는 독립 실행 유닛

```c
[ SoC 전체 ]
 ├─ CPU Cluster
 │   ├─ Physical Core 0
 │   │   ├─ Hart 0
 │   │   ├─ Hart 1
 │   │   └─ Coprocessor (FPU, Vector 등)
 │   ├─ Physical Core 1
 │   │   └─ ...
 │
 ├─ Accelerator
 │   ├─ NPU
 │   ├─ I/O Accelerator
 │   └─ DMA Engine
 │
 ├─ Memory (SRAM / DRAM / Cache)
 └─ Interconnect (AXI, NoC)
```

### Core와 Hart

어떤 구성 요소가 독립적인 명령어 페치 유닛(instruction fetch unit)을 가지고 있다면, 이를 **코어(core)** 라고 부른다.

RISC-V 호환 코어는 **멀티스레딩**을 통해 하나 이상의 **RISC-V 호환 하드웨어 스레드**, 즉 **hart**를 지원할 수 있다.

- **Core** = 물리적인 CPU 한 덩어리
- **Hart** = 그 코어 안에서 동작하는 실행 주체
    - ARM으로 치면 “logical CPU” 같은 개념

> 코어 1개 + hart 4개 → OS 입장에서는 CPU 4개처럼 보일 수 있음
> 

physical core 안에 logical core (hart)

- **Core (physical core)**
    - 실제 연산 유닛
    - instruction fetch / decode / execute 파이프라인 보유
- **Hart (hardware thread)**
    - 코어 안에서 독립적인 실행 컨텍스트
    - PC, 레지스터 상태 따로 가짐

---

### Coprocessor (코프로세서)

RISC-V 코어는 **특정 목적의 명령어 확장(extension)** 이나 **추가된 코프로세서(coprocessor)** 를 가질 수 있다.

여기서 **코프로세서**란,

> “CPU가 하던 일 중 ‘특정 계산/처리’를 더 빠르고 효율적으로 대신 해주는 CPU직속 보조 유닛”
> 
> - **CPU에 붙어 있음**
> - **CPU 명령어로 직접 제어됨**
> - **특정 종류의 연산에 특화됨**

CPU 명령어 스트림에 묶여있음

CPU가 실행하는 명령어 중 일부 opcode가 coprocessor를 직접 사용한다. 

RISC-V에서는 F,D,V,K 같은 ISA extension

PC없음, main loop없음, OS 스케줄링 대상도 아님 → 독자적으로 프로그램을 돌리지 않음

⇒ CPU가 하라고 하면 수행, CPU가 멈추면 같이 멈춤

자체 레지스터/상태 있음

FPU / Vector /Crypto 상태 레지스터가 있는데 문맥 전환 시 CPU가 대신 store/restore 해줌

왜 필요하냐면, 

CPU가 모든걸 다 하게 되면

- 회로 복잡 /  클럭, 전력 불리 / 특수 연산 느림

⇒  그래서 **자주 쓰이지만 무거운 연산**을 CPU 밖(하지만 아주 가까이)에 분리한 게 coprocessor

하는 일 몇 가지 사례

- FPU
    - 부동소수점 연산 전용
        
        float, double연산
        
        ```c
        float c = a * b + d;
        
        이 코드에서:
        	CPU: 명령 흐름 관리
        	FPU(coprocessor): 실제 실수 곱셈/덧셈 수행
        ```
        
- Vector / SIMD extension 유닛
    - 한 명령으로 여러 데이터 처리
    - 영상, 신호, AI 전처리
        
        ```c
        for (i=0; i<8; i++)
          c[i] = a[i] + b[i];
        
        일반 CPU: 8번 반복
        Vector coprocessor: 한 번에 8개
        ```
        
- Crypto extension 유닛
    - 암호 연산 전용
    - AES, SHA, RSA 등
        
        ```c
        encrypt(data, key);
        
        CPU가 비트 연산으로 하면 수천 사이클
        Crypto coprocessor는 수십 사이클
        ```
        

---

### Accelerator (가속기)

CPU명령어 스트림에 직접 묶여있지 않고, 자체적으로 동작 가능하다

주로 인터커넥트(AXI 같은)로 연결되어 있다. 

**가속기(accelerator)** 는 다음 둘 중 하나를 의미한다.

1. **프로그래밍 불가능한 고정 기능 유닛**
2. **자율적으로 동작할 수 있지만 특정 작업에 특화된 코어**

RISC-V 시스템에서는 많은 **프로그래머블 가속기**들이

- RISC-V 기반 코어이거나
- 특수한 명령어 확장 / 커스텀 코프로세서를 가진 형태일 것으로 예상한다.

특히 중요한 가속기 종류 중 하나는 **I/O 가속기**로,

이는 **메인 애플리케이션 코어에서 I/O 처리 작업을 대신 수행(offload)** 한다.

예시:

- 네트워크 패킷 처리 전용 코어
- 스토리지 DMA + 프로토콜 처리 유닛
- NPU
- GPU
- I/O accelerator
- 패킷 처리 코어

위치:

- **SoC 내부**지만
- **CPU 밖**
- “같은 칩에 있는 별도 엔진”

| 구분 | Coprocessor | Accelerator |
| --- | --- | --- |
| CPU와 관계 | CPU에 직결 | CPU와 병렬 |
| 제어 방식 | CPU 명령어 | MMIO / DMA |
| 독립 실행 | ❌ | ⭕ |
| OS 스케줄링 | ❌ | ⭕(가능) |
| 예 | FPU, Vector | GPU, NPU, NIC |

---

[https://riscv.github.io/riscv-isa-manual/snapshot/unprivileged/](https://riscv.github.io/riscv-isa-manual/snapshot/unprivileged/)

[https://github.com/riscv/riscv-isa-manual?tab=readme-ov-file](https://github.com/riscv/riscv-isa-manual?tab=readme-ov-file)

