---
layout: post
title:  "AXI4 SIGNAL – PROT"
date:   2026-03-04 12:00:00 +0900
categories: AXI4 AMBA
show_on_home: false
---

# [SIGNAL] PROT

AXI는 불법 트랜잭션들을 방지하기 위해 ‘접근 권한 신호’들을 설정 할 수 있게 AxPROT 신호를 제공한다. 

읽기 권한 접근 : ARPROT[2:0] 

쓰기 권한 접근 : AWPROT[2:0]

AxPROT는 주소와 함께 전달되는 메타 정보 같은 신호

지금 하는 접근이 어떤 종류의 접근인지 나타내줌

- 특권 접근인지
- 보안 접근인지
- 데이터 / 명령어 접근인지

이 신호를 누가 보나?

- Slave / Interconnect
    - 권한 체크 후 허용할 수 없는 권한이 오면 → DECERR / SLVERR 발생시킴
- Memory Protection Block
- TrustZone Controller

이 신호는 데이터 자체를 바꾸는게 아니라 접근을 허용/거부하는 판단 근거가 됨

즉, slave / interconnect에서 이 AxPROT 신호를 보고 접근 권한을 판단하고 실제 데이터를 보호하기 위해서는 정책을 구현해야 하는 것 

![image.png](/assets/posts_image/AXI/SIGNAL%20PROT/image.png)

1. **Unprivileged / Privileged**

AXI Master는 하나 이상의 운영 권한 레벨(privilege level)을 지원 할 수 있는데, 이 권한 개념은 메모리 접근 시에도 적용 된다. 

AxPROT[0]은 접근이 unprivileged인지, privileged인지 식별할 수 있도록 하는 판단 근거가 된다. 

<aside>
💡

일부 프로세서는 여러 단계의 권한 레벨을 지원한다.

해당 프로세서에서 AXI 권한 레벨으로의 매핑은 프로세서 문서를 참고해야 한다. 

AXI가 제공할 수 있는 구분은 오직 “특권/비특권” 두 단계 뿐이다

</aside>

대부분의 SoC에서,

- CPU가 커널 레벨(OS kernel, EL1이상)에서 메모리에 접근하면 AxPROT[0] = 0 (Privileged)로 신호가 나간다
- User 레벨(EL0)에서 메모리에 접근하면 AxPROT[0] = 1 (Unprivileged)로 신호가 나간다.

메모리 영역 아래처럼 나뉠 수 있다. 

```verilog
0x0000_0000 ~ 0x0FFF_FFFF : User 접근 가능
0xF000_0000 ~ 0xFFFF_FFFF : Kernel만 접근 가능
```

interconnect / firewall / MMU에서는 AxPROT신호를 갖고 권한을 이렇게 판단한다 

```verilog
if (AxPROT[0] == Unprivileged && addr == kernel_area)
    → ACCESS DENY
```

예 1 ) CPU

- User App 실행 중:
    - load / store → AxPROT[0] = 0
- Kernel 코드 실행 중 :
    - load / store → AxPROT[0] = 1

CPU가 자동으로 이 비트를 바꿔서 내보내게 됨(개발자가 매번 설정하지 않음)

상황 : Linux 커널이 드라이버 레지스터 접근 

- Linux kernel (EL1)
- MMIO 레지스터 write (UART, DMA, GPIO 등)

AXI 주소 채널:

AWADDR   = device register

AWVALID  = 1

AWPROT[0]= 0   // Privileged

AWPROT[1]= (Secure/Non-secure 상태에 따라)

⇒  커널 드라이버 접근 = Privileged

상황 : 사용자 앱이 /dev/mem으로 직접 접근 시도

- User app (EL0)
- 물리 주소 접근 시도

AXI 트랜잭션:

ARPROT[0] = 1   // Unprivileged

Interconnect / MPU:

주소 = kernel-only region

AxPROT[0] = Unprivileged

⇒ 접근 차단, 유저 앱은 하드웨어 직접 접근 못 함

예 2) DMA

- DMA는 CPU가 아니니까 누가 설정해줘야 한다.
- DMA Descriptor에 보통 설정 하는게 있다
- 이 DMA 전송을
    - “User 요청으로 할 건가?”
    - “Kernel 권한으로 할 건가?”

CPU는 여러 privilege level이 있는데 AXI는 2단계만 있는 듯…

ARM CPU 예:

EL0 (user)

EL1 (kernel)

EL2 (Hypervisor)

EL3 (Secure monitor)

AXI

EL0 → Unprivileged

EL1/2/3 → Privileged

커널 메모리를 유저 앱에서 건드리면 안돼

디바이스의 레지스터를 앱이 직접 건드리면 안돼

DMA가 멋대로 커널 영역을 덮으면 안돼 

⇒ AxPROT[0] 으로 일단 1차 방어 하는거임

1. **Secure / Non-Secure**

AXI Master는 Secure / Non-Secure 운영 상태를 지원할 수 있으며, 이런 보안 개념을 메모리 접근에도 확장할 수 있다. 

AxPROT[1]은 접근이 Secure인지 Non-Secure인지를 식별 할 수 있는 판단 근거가 된다.

<aside>
💡

이 비트는 asserted(1)일 때,

해당 트랜잭션이 Non-Secure 임을 나타내도록 정의되어 있다.

이는 ARM Security Extensions(TrustZone)의 다른 신호 정의와 일관성을 유지하기 위함이다

</aside>

즉, AxPROT[1]은 “나는 어떤 세계에서 왔다!” 를 알려주는 신분증이라고 보면 될 것 같다

- Secure World에서 온 트랜잭션?
- Non-Secure World에서 온 트랜잭션?

SoC에서는 아래와 같은 단위로 나뉘고

| Secure + Priv | Secure monitor, TEE OS |
| --- | --- |
| Secure + Unpriv | Secure app (TEE user app) |
| Non-sec + Priv | Linux kernel |
| Non-sec + Unpriv | Linux user app |

CPU에서 내보내는 AXI 트랜잭션에서는 AxPROT[0] + AxPROT[1]을 보통 같이 결합해서 권한 표현한다. 

참고 ) “Secure World에서 발생한 트랜잭션의 경우 CPU가 내보내는 AXI 트랜잭션의 AxPROT[1]이 0으로 설정된다.”

Secure world에서 load/store 또는 instruction fetch 발생하면,

CPU 내부 :

현재 상태 = Secure world

↓

AXI 트랜잭션 생성

↓

AxPROT[1] = 0 (Secure)

- Secure world 코드가 실행 중일 때 발생한 AXI 접근 → AxPROT[1]=0
- Normal world 코드가 실행 중일 때 발생한 AXI 접근 → AxPROT[1]=1

⇒ 컨텍스트 기반으로 AXI 트랜잭션 신호가 자동 설정됨

Secure 메모리는 실제로 누가 지키냐?

Secure 메모리를 실제로 보호하는 주체들은 아래 애들이다

- TrustZone Address Space Conroller
- AXI Interconnect의 Security filter
- Memory Controller
- Firewall / MPU / IOMMU

이런 보호 주체들이 신호를 기반으로 내부적으로 아래처럼 판단을 하게 된다 

```verilog
if (address ∈ secure_region) {
	if (AxPROT[1] == Secure) 
		allow;
	else 
		deny; // Non-secure 접근 차단
}
```

근데, 이론적으로는 AxPROT[1] 조작이 가능하기 때문에 AxPROT만 믿으면 안 된다.

CPU가 *Secure world* 에서 Secure DMA의 레지스터에 접근할 때 아래 내용을 실어서 보내고,

“주소 채널(AW/AR)의 주소” + “AxPROT[1]=0” + “VALID”  

SoC에서는 “누가 보냈는지” + “뭐라고 주장하는지”를 같이 검사해서 판단한다.

(master ID == Cortex-A Secure port) AND (AxPROT[1] == 0)

→ Secure access 허용

또는

(master ID == DMA_non_secure)  AND (AxPROT[1] == 0)

→ 차단 (위조 시도)

예 1) 

**상황 설정**

- CPU: Cortex-A
- 현재 실행 상태: **Secure world**
- 대상: **Secure DMA 컨트롤러 레지스터**
- 동작: DMA 설정 레지스터 write

CPU 내부 상태

Secure Monitor / TEE 코드 실행 중

```verilog
CPU Status = Secure
Privilege = EL3 또는 Secure EL1
-> CPU는 이 상태를 AXI 속성으로 변환함
```

AXI Write Address Channel에서 나가는 신호

- CPU가 Secure DMA 레지스터에 write 시도 하면
- AW 채널에서 아래처럼 나감

```verilog
AWADDR = Secure DMA Register Address
AWVALID = 1
AWPROT[0] = (priv / unpriv) - 커널이냐 유저냐
AWPORT[1] = 0 //secure
AWPORT[2] = 0 // data access (보통)
```

Interconnect에서 처리

Interconnect / Security Firewall은 이렇게 봄

```verilog
이 master는 보안 접근 권한이 있는 놈이 맞나? (master id check)
secure 접근이 맞나? (AxPROT[1] == 0)
가려는 주소가 Secure DMA 영역인가?

전부 OK면
-> Secure DMA Slave 로 트랜잭션 전달
아니면 
-> 아예 전달 안해버림 (DECERR / SLVERR)
```

Write Data Channel은 별도지만, 속성은 이미 확정

“AXI에서는 ‘보안, 권한 판단’이 주소 채널(AW)에서 이미 끝나고, 데이터 채널(W)에서는 그 결정 결과를 그대로 따라간다.” 

```verilog
WDATA = 실제 값
WSTRB = 어떤 바이트가 유효한지
WVALID = 1

W채널에는 AxPROT가 없음
이미 AW에서 어디로 갈지 확정됨
```

Secure DMA 입장

Secure DMA까지 트랜잭션이 도착 했다는건 이미 “보안 조건 통과” 라는 뜻임

그래서 Secure DMA는

- Secure 설정 레지스터 write 허용
- Secure Mode로 동작 가능

### “그럼 Secure DMA 내부에선 아무 검사도 안 해?”

구현에 따라 다름

- 어떤 IP는:
    - Interconnect만 믿음
- 어떤 IP는:
    - 내부에서도 AWPROT 다시 확인

문서에 있는 내용 :

> “또는 내부에서 다시 AxPROT 확인 (구현에 따라)”
> 

**Instruction 또는 Data**

이 비트는 해당 트랜잭션이 명령어 접근(instruction access) 인지 데이터 접근(data access) 인지를
나타낸다.

AXI 프로토콜에서는 이 표시를 힌트(hint) 로 정의한다.

즉, 모든 경우에 정확하지 않을 수 있다.

예를 들어, 하나의 트랜잭션에 명령어와 데이터가 섞여 있을 수도 있다.

이 사양에서는, 접근이 명령어 접근임이 명확히 알려진 경우가 아니라면, master가 AxPROT[2] 를 LOW로
설정하여 데이터 접근임을 표시할 것을 권장한다.

**실제로 어떻게 쓰이냐면**

- **AxPROT[0]**→ 커널/유저 접근 구분
- **AxPROT[1]**→ TrustZone Secure/Non-secure 메모리 보호
- **AxPROT[2]**→ 캐시/프리페치/보안 로직 참고용 힌트

중요:

- AxPROT는 **강제 보호 메커니즘이 아니라 태그**
- 실제 차단 여부는 interconnect / slave / security controller가 결정