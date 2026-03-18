---
layout: post
title:  "AXI4 – Glossary"
date:   2026-03-04 12:00:00 +0900
categories: AXI4 AMBA
show_on_home: false
---
<br>
# 용어 설명

### **Prefetch**

"곧 쓸것 같아서 미리 읽어두는 것"

```c
for (i = 0; i < 1000; i++)
	sum += arr[i];
```

CPU는
- arr[0] 읽고, 그 다음은?

	arr[1], arr[2] .. 올 확률이 높음

	그래서 지금 필요한 값 말고 다음 캐시 라인 까지 미리 가져와 둠

	- arr[0]의 주소  = 0x0001
	- Int = 4byte
	- 캐시 라인 = 64byte (보통 캐시 라인 크기 = 64byte가 흔함)
    
    ```c
    캐시 라인 0 : 0x0000 ~ 0x003F
    	arr[0]  @ 0x0001
    	arr[1]  @ 0x0005
    	arr[2]  @ 0x0009
    	...
    ```
    

arr[0] 하나 읽어도, CPU는 0x0000 ~ 0x003F 전체를 캐시에 가져옴 (이게 기본적인 cache line fill임)

<br>

arr[0]을 읽었을 때

1. 현재 캐시라인
    
    0x0000 ~ 0x003F    <- 반드시 가져옴
    
2. CPU가 패턴을 보고 판단
    
    "순차 접근이네?"
    
3. 그래서 다음 캐시 라인도 미리 가져옴
    
    0x0040 ~ 0x007F   **<= prefetch**
    
    많아야 다음 1~2 캐시라인 더 가져옴
    
    다음 read가 오면 바로 캐시 hit
    
    ```c
    시간 ->
    	arr[0]  -> [라인 0 가져옴] + [라인 1 prefetch)
    	arr[1]  -> 캐시 hit
    	arr[2]  -> 캐시 hit
    	...
    	arr[15]  -> 캐시 hit
    	arr[16]  -> 이미 prefetch 된 라인 hit
    	
    	==> 이렇게 되기 때문에 루프가 빨라지는 것
    ```
    

---

### Write Combine

“작은 쓰기 여러 개를 모아서 한번에 쓰기”

```c
arr[0] = 1;
arr[1] = 2;
arr[2] = 3;
arr[3] = 4;

이게 하나하나 메모리로 가면 => Write 4번 해서 느림
		“하나하나 메모리로 간다”는 말은
			arr[0] write → 응답
			arr[1] write → 응답
			arr[2] write → 응답
			arr[3] write → 응답
	: 이렇게 매번 따로따로 AXI 트랜잭션이 발생하는 상황을 말한다.
그래서 => 	이걸 버퍼에 잠깐 모아두고, 한번에 큰 write(burst)로 전송
```

Write combine이 *없는* 경우 (느린 경우)

```c
예를 들어 CPU 코드가 이렇다고 해보자:
	arr[0] = 1;
	arr[1] = 2;
	arr[2] = 3;
	arr[3] = 4;
시스템에서 실제로 벌어질 수 있는 일(단순화해서 표현하면)
	arr[0] 
		- AWADDR(주소0) + WDATA(1) 전송
		- 슬레이브 응답(BRESP)
	arr[1] 
		- AWADDR(주소1) + WDATA(2) 전송
		- 슬레이브 응답(BRESP)
	arr[2] 
		- AWADDR(주소2) + WDATA(3) 전송
		- 슬레이브 응답(BRESP)
	arr[3] 
		- AWADDR(주소3) + WDATA(4) 전송
		- 슬레이브 응답(BRESP)

주소/데이터/응답이 4번 왕복
버스 효율이 별로임
DDR 같은 경우 지연도 큼
```

Write combine이 *있는* 경우 (빠른 경우)

```c
interconnect / memory controller가 하는 일
- CPU가 연속 주소로 write 하는걸 봄
- "이거 묶어도 되겠다 (cacheable/bufferable) 판단"
그래서, 
	arr[0] write
	arr[1] write
	arr[2] write
	arr[3] write
를 내부 write buffer에 잠깐 저장

그리고 나서 하나의 burst write
	주소 = arr[0]
	데이터 = [1,2,3,4]
- 주소 1번, 데이터 burst 1번, 응답 1번
```

---

### Merge

“여러 개의 AXI Write 요청을, 더 큰 1개의 Write Burst로 합치는 것”

“버스에 나가는 요청 개수(AW)가 줄어드는” 최적화 

- 원래: AW가 10번, W가 10번(또는 10 burst)
- merge 후: AW가 1번(큰 burst 1개), W는 길게

<br>

combine이 “데이터를 모아 큰 폭으로 한 번에”라면,

merge는 “요청(트랜잭션) 자체를 합쳐서” 한 번에 가는 느낌.

<br>

merge는 어디 한 군데 만의 기능이 아니라, 시스템에 따라 3군데에서 일어날 수 있다.

1. **CPU Store Buffer / Write-Combine Buffer**
    - CPU가 여러 store를 내보내기 전에 내부에서 “연속 주소 store”를 묶어서
    - AXI로 나갈 때 이미 큰 burst로 만들어버림
2. **AXI Interconnect (NIC/NoC) 내부 Write Buffer**
    
    “interconnect가 AxCACHE 보고 merge 한다”
    
    - 여러 마스터에서 온 요청을 buffer에 쌓아두고
    - “같은 성격+연속 주소”면 burst를 키우거나 합침
3. **Memory Controller(DDR 컨트롤러) 쪽 write gathering**
    - AXI burst 자체는 그대로일 수도 있는데
    - DDR command로 나갈 때 더 효율적으로 묶기도 함 (이건 “AXI merge”라기 보다 하위 단 최적화)

<br>

- merge가 실제로 어떻게 합치냐 (AXI 채널 기준)

	AXI Write는 크게 아래와 같이 간다

		AW : 주소 + burst 정보

		W : 데이터 beat 들

		B : 응답

	merge는 결국

	여러개의 AW를 한 개의 AW(더 긴 burst)로 바꾸는것

	예시 1) 4Byte Store 4번 (연속 주소)

	```c
	CPU가 4번 store 했다고 치면,
		write 0x8000_0000 (4Byte)
		write 0x8000_0004 (4Byte)
		write 0x8000_0008 (4Byte)
		write 0x8000_000C (4Byte)
	
	merge 전 (트랜잭션 4개)
		AW * 4
		W * 4 (각각 1 beat)
		B * 4

	merge 후 (트랜잭션 1개)
		AW * 1 : addr=0x8000_0000, burst=INCR, size=4Byte, len=4(beats)
		W * 1 : 4 beats 연속
		B * 1
	
	=> 버스에서 주소 phase(AW) overhead가 확 줄어듦 + NoC/DDR 효율 좋아짐
	```

<br>

**combine / merge**

combine은 CPU 내부 최적화

merge는 AXI fabric(interconnect) 최적화

- CPU는 “이 4byte는 같은 word이다”
- Interconnect는 “이 4개의 word 요청은 연속이다”를 보고 burst로 바꿈

```c
byte writes
   │
   ├─ combine → [ 4B ]
   │
   └─ merge   → [ 4B ][ 4B ][ 4B ][ 4B ]  → 1 burst

```

```c
**combine**
여러 개의 작은 write 데이터를 모아서 한 번에 쓰는 것
트랜잭션은 그대로일 수도 있고, write buffer 안에서 내용만 합쳐짐
여러 byte write -> 하나의 bus beat로

CPU 내부 1 byte store buffer * 4번 (주소는 연속)
*(char*)0x80000000 = 0x11;
*(char*)0x80000001 = 0x22;
*(char*)0x80000002 = 0x33;
*(char*)0x80000003 = 0x44;
=> 4개의 1byte store를 모음 

combine 가능하면 write buffer에서 combine 후
0x80000000 ← 0x44332211 (32-bit write)
=> 1개의 32-bit write (WSTRB=1111)

AXI 관점이서 이렇게 됨 (burst 길이는 1)
AW: addr=0x80000000, len=0 (1 beat)
W : data=0x44332211, strobe=1111
B

메모리 입장에서는 한 번만 씀
```

```c
merge
여러 개의 AXI write 트랜잭션을 하나로 합치는 것
combine 다음 단계
"이미 만들어진 write beat 여러개를 모아서 한개의 AXI burst로 만든다"

CPU가 이런걸 만들었다 (각각은 combine으로 이미 4byte beat 상태)
store 32bit @0x80000000
store 32bit @0x80000004
store 32bit @0x80000008
store 32bit @0x8000000C

merge 가능하면 interconnect가:
AW: addr=0x80000000, len=3, size=4B
W : 4 beats

AXI 트랜잭션 개수 자체가 줄어듦
```

“이미 만들어진 write beats를 interconnect가 burst로 묶을 수 있다”

> “cacheable & bufferable writes may be merged into larger bursts”
> 

⇒ 이미 존재하는 write beats를 burst로 묶어도 된다.

---

### Reorder

“CPU가 낸 메모리 접근 순서와, 실제로 버스/메모리에 도착하는 순서가 달라지는 것”

```c
A = 1;
B = 1;

이 코드가 항상
	A 먼저 
	B 나중
으로 메모리에 쓰일까?

CPU + 버스 + 인터커넥트 입장에서 보면
	B 먼저
	A 나중
으로 나가도 프로그램이 깨지지만 않으면 OK 임
==> 이게 reorder
```

**Reorder 허용**

```c
int A;
int B; 

선언 시,

A = 0x8000_0000
B = 0x8000_0004   ← 거의 항상 이렇게 연속
DDR내부는 주소가 연속이어도 "길" 이다를 수 있다
```

---

DDR Bank?

DDR Bank는 전압 구역이 아니라 독립적으로 일 할 수 있는 메모리 작업 구역? 이라고 보면 될듯?

```c
DDR 칩 내부는 대략 이렇게 생겼음:
	DDR Chip
	 ├─ Bank 0
	 │    ├─ Row buffer
	 │    └─ Memory array
	 ├─ Bank 1
	 │    ├─ Row buffer
	 │    └─ Memory array
	 ├─ Bank 2
	 └─ Bank 3
각각의 Bank는
	자기만의 row buffer
	자기만의 sense amplifier
	자기만의 precharge / active를 갖는다고 한다. 
	
=> bank끼리는 동시에 동작 가능
```

DDR은 느린 구조라서

한 bank 안에서 수십 ns 걸림

```c
Row 열기 (ACTIVATE) → Column 읽기 → PRECHARGE
```

그래서 bank를 여러 개 둬서

```c
Bank 0에서 Row A 읽는 동안
Bank 1에서 Row B 준비
```

병렬로 돌리기 위함

---

CPU가 쓰는 ‘물리 주소’ 를 DDR 컨트롤러가 내부 DRAM 구조에 맞게 분해해서 사용하는데

```c
CPU 입장 →  연속된 메모리
	0x8000_0000
	0x8000_0004
	0x8000_0008

DDR 칩 내부
[ Bank ]
  └─ [ Row ]
        └─ [ Column ]
        
DDR 컨트롤러는 CPU주소를 받아서 
주소 → (bank 번호, row 번호, column 번호)
요렇게 쪼개서 DRAM에 명령어를 보냄

DDR는 이렇게 접근함:
	어떤 bank를 쓸지 선택
	그 bank의 어떤 row를 열지 (ACTIVATE)
	그 row의 어떤 column을 읽거나 쓸지 (READ/WRITE)
	다 쓰면 row 닫기 (PRECHARGE)
```

“연속 주소인데 bank 가 바뀐다”

DDR 컨트롤러는 CPU 주소 쪼갤 때 아래 처럼 매핑했다고 해보자,

- addr[15:12] = bank
- addr[11:2]  = column
- addr[31:16] = row

그러면

- 0x8000_0000 = bank 0, column 0
- 0x8000_0004 = bank 1, column 0   ← bank bit가 바뀜

CPU 입장에서는 고작 +4 증가했는데, 

interconnect / DDR 입장에서는 다른 bank로 이동했다는 뜻이라, 

	A → bank 0 (바쁨)
	B → bank 1 (비어 있음)

 <span style="color:green">“A는 bank 0이 막혔네, B는 Bank 1이 비어있으니 B부터 보내자”  ⇒ reorder 가 될 수 있는 거임</span>

```c
Core
  ↓
Store Buffer <- store buffer는 “CPU와 메모리 사이의 완충 탱크”
  ↓
AXI Write Queue
  ↓
Interconnect
  ↓
DDR Controller
  ↓
DRAM Banks
```

CPU는:

- store를 store buffer에 넣고 바로 다음 명령을 실행함

**store buffer :** 

```c
store buffer는?
	“store 명령을 일단 임시로 담아두고, 메모리(캐시/AXI)로 천천히 흘려보내는 완충 버퍼”
	“CPU코어 안에 있는 queue”

CPU Core
 ├─ Load/Store Unit
 │    └─ Store Buffer   ← 여기
 ├─ L1 Cache
 └─ MMU
 
 CPU는 store를 아래처럼 처리함
 Core
  |
  v
 Store buffer  →  AXI write queue  →  Interconnect  →  DDR
 
 이게 없으면 모든 store 명령 마다 메모리에 진짜 써질 때까지 CPU는 멈춰서 기다려야 함
```

메모리는:

- 뒤에서 천천히 처리

<br>

여기서 reorder를 허용한다면?

```c
CPU code
	A = 1;
	B = 1;

store buffer
	[A=1]
	[B=1]

DDR 상태:
	A → Bank 0 (지금 바쁨)
	B → Bank 1 (지금 비어 있음)

interconnect 판단 :
	"B를 먼저 보내면 바로 처리 된다"
	
AXI → B 먼저
AXI → A 나중
```

store buffer 입장에서는 일석이조인 상황

두 개 다 빨리 비워 치울 수 있음 

CPU는 계속 다른 일 할 수 있음

<br>

reorder를 금지한다면?

“A가 끝나야 B를 보낼 수 있다”

```c
store buffer :
	[A=1]  ← Bank0 바쁨 → 멈춤
	[B=1]  ← 뒤에서 줄서기

DDR 상태
	Bank 1 (B용) 비어 있음 😭

근데 B를 먼저 못보냄
-> bus + DDR 절반이 놀고 있는 상태 (최악)
```
<br>
store buffer 가 무력화 됨

- store buffer 는 “아무 순서로나 보내도 된다” 라는 가정 하에 설계된 앤데

- reorder가 금지되면 병목 발생

	buffer의 첫 entry가 막혀서 못나가면

	이후 뒤로 오는 모든 entry가 못나감

DDR burst도 깨짐

DDR은 아래처럼 한번에 burst로 보내는걸 좋아함

```c
0x8000
0x8004
0x8008
0x800C
→ 한 번에 burst
```

근데 A가 0x9000, B가 0x8000이라면

```c
reorder 허용
	B(0x8000) 먼저 보내서 burst 구성
	A 나중

reorder 금지
	A가 끝날 때까지
	0x8000도 대기
	→ burst 기회 상실

```

<br>

reorder 금지는 성능이 폭락의 결과를 가져올 수 있음

- store buffer 병목
- interconnect 유휴
- DDR bank 유휴
- burst 기회 상실

→ CPU, 버스, DDR 전부 서로 발목 잡음

<br>

Buffering = “잠깐 들고 있다가 나중에 보낸다”

Write merge = “여러 개를 하나로 재구성해서 보낸다”

<br>

1️⃣ Write buffering (버퍼링)

뭐냐면

- interconnect가 write를 임시 저장해 두는 것
- 형태는 그대로, 단지 시간만 늦춤

특징

- write A는 write A 그대로 유지
- 순서 유지 (보통)
- 나중에 그대로 전달

왜 하냐면

- CPU에게 write response를 빨리 주기 위해
- 파이프라인 안 막히게 하려고

예시

CPU:

write 0x1000 = 1

write 0x1004 = 2

Interconnect (buffer):

[write 0x1000 = 1]  ← 저장

[write 0x1004 = 2]  ← 저장

→ 나중에 그대로 DDR로 전송

👉 write 개수 = 그대로

2️⃣ Write merge (write combine)

뭐냐면

- 여러 개 write를 하나의 큰 write로 합침
- 트랜잭션 형태가 바뀜

특징

- 여러 write → 하나의 burst write
- 주소 / size / len 변경됨
- Modifiable일 때만 가능

왜 하냐면

- DDR 대역폭 효율 ↑
- burst 전송이 훨씬 빠름

예시

CPU:

write 0x1000 = 1

write 0x1004 = 2

write 0x1008 = 3

write 0x100C = 4

Interconnect (merge):

→ burst write

addr=0x1000

size=4B

len=4

👉 write 4개 → write 1개

![image.png](/assets/posts_image/AXI/AXI%20Glossary/image.png)