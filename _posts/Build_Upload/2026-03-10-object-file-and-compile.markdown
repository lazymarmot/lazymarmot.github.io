---
layout: post
title: "오브젝트 파일과 컴파일"
date: 2026-03-10 11:10:00 +0900
categories: Build Upload
show_on_home: false
---

# 오브젝트 파일과 컴파일

## 오브젝트 파일

오브젝트 파일은 “링커가 이해하고 다룰 수 있는 ELF 단위 결과물” 이라고 하면 될 것 같다. 

(코드(.text), 데이터(.data, .bss), 심볼테이블, relocation 정보 같은걸 구조적으로 담고 있는 파일)

단지 오브젝트 파일의 종류가 다른 것일 뿐 .o / .so / .elf 실행 파일 전부 오브젝트 파일이라고 한다

```c
[ ELF ]  ← 오브젝트 파일 포맷 (형식)
   ├─ .o    : Relocatable Object
   ├─ .elf  : Executable Object
   └─ .so   : Shared Object
```

1. .o  (relocatable object)
    - 재배치 가능 오브젝트 파일
    - 링크 전 단계
    - 심볼, relocation 정보가 있음
    - 컴파일 결과물로 아직 주소가 미확정
    - ELF 타입 : ET_REL
    
    ```c
    file main.o
    # ELF relocatable
    
    "가공 전 부품”
    ```
    
2. a.out, app, app.elf (실행 가능 파일)
    - 실행 가능한 오브젝트 파일
    - 링크 단계 완료 됨
    - 엔트리 포인트 있음
    - 커널이나 부트로더가 로드 가능
    - EFL 타입 : ET_EXEC 또는 ET_DYN
    
    ```c
    file app.elf
    # ELF executable
    
    “완제품”
    ```
    
3. .so (shared object)
    - 공유 라이브러리 오브젝트 파일
    - 런타임에 로더가 동적 로딩
    - 실행 파일이랑 구조가 거의 동일
    - ELF 타입 : ET_DYN
    
    ```c
    file libfoo.so
    # ELF shared object
    
    “공용 부품 (실행 중 끼워 넣는)”
    ```
    

위에서 오브젝트 파일은  ELF 단위 결과물 이라고 했는데, ELF포맷의 오브젝트 파일은 헤더와 섹션으로 구성된다.

헤더 : 파일 구성에 대한 정보

섹션 : 실행 코드나 데이터 저장, 링킹에 필요한 제어 정보

아래 내용은 ELF “relocatable object”의 내부 구조를 설명하고 있으며, 딱 컴파일 직후 .o 파일의 특징을 갖는다

```c
ELF Header
.text     <--- 컴파일된 바이너리 코드
.rodata   <--- 상수 같은 읽기 전용 데이터  
.data     <--- 초기화된 전역변수 / 정적변수
.bss      <--- 초기화되지 않은 전역변수 / 정적변수 
.symtab   <--- 정의된 전역변수와 함수들에 대한 참조 정보, 링크 전에만 존재, 실행 파일에서는 보통 제거됨
.debug    <--- 디버거용 정보, 실행파일에는 없어도 됨
.line
.strtab
```

섹션 중심 구조

- .text, .data, .bss 같은 섹션 위주로 아직 메모리 배치(program header)개념은 없음
- 제어 정보를 갖는 섹션은 주로 심볼 테이블이나, 재배치 정보를 갖는 섹션들이다

| relocation ELF | 실행 가능 파일 ELF |
| --- | --- |
| 섹션 중심<br>심볼 많음<br>relocation 정보 있음 | program header(PT_LOAD) 있음<br>섹션보다 segment가 중요<br>커널/부트로더가 바로 로딩함<br>.symtab, .debug 보통 없음 |

<br>

## 컴파일 과정

아래는 test.c 파일을 컴파일해서 실행파일을 만드는 과정을 보여준다. 

![image.png](/assets/posts_image/Build_Upload/object_file_and_compile/image.png)

```c
gcc test.c -o test
```

위 명령을 사용해서 빌드하고 실행파일을 최종적으로 만드는데, 

실질적으로는 여러 단계의 작업이 이루어 진다.

<br>

### 전처리 단계 (전처리기)

```c
cpp0 test.c /tmp/test.i
-> 전처리기가 소스파일 처리, 이 결과는 tmp 디렉토리에 test.i라는 이름으로 임시저장되고 나중에 이 파일을 삭제됨
```

- 전처리기(Preprocessor)는 C문법이 아닌 것들을 전부 처리한다
    - #include, #define, #if / #ifdef
    - 주석 제거
- test.i 파일이 출력 된다.
- 매크로가 전부 펼쳐짐
- 헤더 파일 내용이 그대로 복사
    
    ```c
    전처리 전
    #define N 10
    int a[N];
    
    전처리 후
    int a[10];
    ```
    
<br>

### 컴파일 단계 (컴파일러)

컴파일러에 의해 어셈블리 코드를 생성한다

```c
cc1 /tmp/test.i -o /tmp/test.s
-> 전처리 수행 결과 파일인 tmp 디렉토리의 test.i를 컴파일하여 tmp 디렉토리에 test.s로 저장한다.
```

- C코드를 CPU용 어셈블리로 변환
    - 문법검사, 타입 체크, 최적화
    - 타겟 아키텍쳐 반영(ARM, RISC-V, x86등)
- 어셈블리 코드가 출력되는거지 아직 기계어는 아님
- CPU 명령을 여기서 확인할 수 있음
    
    ```c
    ldr r0, [r1]
    add r0, r0, #1
    ```
    
<br>

### 어셈블러 단계 (어셈블러)

컴파일러 단계를 통해 생긴 어셈블리 코드는 어셈블러에 의해 오브젝트 파일로 변환된다 

```c
as /tmp/test.s -o /tmp/test.o
```

- 어셈블리 코드 → 기계어
    - 명령어를 opcode로 변환
    - 섹션 생성 (.text, .data, .bss)
    - 심볼 테이블 생성
    - relocation 정보 생성
- ELF relocatable object생성
- 주소 미확정 상태

<br>

### 링크 단계 (링커)

여러개의 오브젝트 파일이나 사용되는 라이브러리 코드들이 결합되는 단계

```c
ld /tmp/test.o /tmp/test2.o -o test
```

코드 빌드를 할 때, 링커에서 최종 실행파일(elf)을 만든다. 

이때 링커가 사용하는 파일이 아래와 같다.

- linker.ld → 링커스크립트(.ld) : “.text/.data/.bss를 메모리 어디에 둘지” 결정하는 지도 같은 파일
- startup.o→ 어셈블리로 만든 스타트업 코드(startup.s) : 리셋 핸들러/벡터 테이블/초기화 루틴 같은 “실행 될 코드 그 자체”
- main.o, test.o → C 코드들

> 링커는 먼저 입력 오브젝트 파일들의 섹션을 병합하여 최종 섹션 크기를 결정한 후, 해당 섹션들의 메모리 주소를 배정하고 심볼 값을 확정한 뒤 재배치를 수행한다
> 

링커가 실질적으로 하는 작업 

1. **링커 시작 전 (.o 상태)**
    
    각 .o 파일 안에서는,
    
    - 심볼 테이블
    - 섹션 내부 offset
    - relocation 엔트리 정보가 분리되어 들어있음
    
    아래 코드를 예를 들어서 오브젝트 파일 내부를 설명해 보자면
    
    ```c
    // b.c
    void foo() {
        bar(); // bar()은 다른 파일에 있음
    }
    ```
    
    링커 전 단계의 .o(ET_REL)은 보통 아래와 같은 구조를 갖는다
    
    ```c
    [ ELF Header ]
    [ Section Header Table ]
    
    [ .text ]        ← 기계어 (주소 미확정)
    0x00: push lr
    0x04: bl 0x00000000   ← ⚠️ 아직 주소 모름 (“bl bar” 자리에 진짜 주소가 없어서 일단 0이나 더미 값이 들어감)
    0x08: pop lr
    0x0c: ret
    
    [ .data ]
    [ .bss ]
    
    [ .symtab ]      ← 심볼 테이블
    [ .strtab ]      ← 심볼 이름 문자열
    [ .rel.text ]    ← .text용 relocation
    [ .rel.data ]    ← .data용 relocation
    
    컴파일러 가 b.o를 만들 때 bar은 다른 파일에 있어서 실제 주소를 모름
    그래서 bar호출 자리는 아직 주소가 미확정임
    0x04 위치는 "여기서 bar로 분기 해야 한다는 의도만 있음"
    => .text (코드)안에는 주소 값이 없음
    ```
    
    위 내용에서 bar 함수를 불러야 한다 는 정보는 어디 있을까? (남이 정의한 함수 써야 할 때)
    
    다른 파일에 정의된 심볼이라 지금 주소 모름
    
    코드(.text) 어느 위치에 bar 주소 채워야 하는 지가 중요
    
    relocation 엔트리 (.rel.text / .rela.text)에 그 정보가 있음
    
    ```c
    offset: 0x04          ← .text + 0x04 위치
    type:   R_ARM_CALL    ← 함수 호출용 relocation
    symbol: bar
    addend: 0
    
    => ".text 섹션 + 0x04 위치에 나중에 bar의 실제 주소를 계산해서 패치해라"
    ```
    
    그럼 foo는 어디에 정의 되어 있을 까? (내가 정의 한 함수)
    
    이 오브젝트 파일 안에 정의된 심볼이라서 위치만 알면 됨 (.text + offset)
    
    심볼 테이블 (.symtab)
    
    ```c
    Symbol table entry:
    name: foo
    type: FUNC
    section: .text
    value: 0x00
    
    => foo = .text 섹션 시작 + 0x00
    ```
    
2. 섹션 병합
    
    그림과 같이 각  파일 안에 있는 섹션(.text, .data, .bss …)을 링커 스크립트 규칙에 따라 하나로 합치는 과정이 섹션 병합 단계라고 보면 된다. 
    
    이 단계에서는 아직 함수/변수가 연결되지 않았고, 메모리 주소도 없고, 가상 주소도 없고, 그냥 덩어리 합치기 정도라고 보면 될 것 같다.
    
    이때 스타트업 코드의 실행 코드 부분도 elf의 .text섹션에 포함된다.
    
    
![image.png](/assets/posts_image/Build_Upload/object_file_and_compile/image1.png)

<br>

<span stype="color:green"><strong>섹션 병합 전 심볼 수집 단계</strong></span>

섹션 병합 단계에서는 아래 내용의 정보들이 생성된다. 

```c
섹션 병합 단계에서 실제로 생기는 내용

[출력 ELF (구조 생성 중)]
 ├─ .text   ← 여기에 startup.s의 코드 + main() 코드 + 함수 코드 전부
 ├─ .rodata
 ├─ .data
 ├─ .bss
 ├─ 링커 스크립트에 정의된 다른 섹션들
 
[링커 내부 심볼 테이블] => relocation 처리 할 때까지 계속 유지됨 최종 elf 실제 만들땐 없앰
 .o파일들의 .symtab에서 수집한 내용들이 존재하는 심볼 테이블
 "foo, bar는 병합된 .text 안에서 각각 이 offset 위치에 있다" 
		foo → ELF .text + 최종 offset
		bar → ELF .text + 최종 offset
		
[링커 내부 relocation 리스트]
	".text + 0x04에 있는 더미 값 자리를 bar의 실제 주소 값으로 패치해라"
		패치 위치: ELF .text + 0x24
		참조 심볼: bar
		relocation 타입: R_ARM_CALL
```
<br>

링커 내부 심볼 테이블에는 어떤 정보들이 들어갈까???

- 심볼 테이블에는 각 .o 오브젝트 파일들에서  .symtab 정보들을 수집해서 넣어둔다.

```c
// b.c
void foo() {
    bar(); // bar()은 다른 파일에 있음
}

// c.c
void bar() {
}

b.o 의 .symtab
	foo : 정의됨
	bar : 미정의

b.o의 .rel.text
	.text + offset 위치에 bar 주소 채워

c.o의 .symtab
	bar : 정의됨
```

모든 .o파일들의 .symtab를 임시로 모아서 링커 메모리 안의 내부 심볼 테이블 생성해서 거기에 정리됨(개념적으로)

```c
foo → (정의됨, b.o, .text, offset)
bar → (정의됨, c.o, .text, offset)

여기서 수집되는 .text + offset은 파일 내부 기준 오프셋일 뿐 
병합된 .text 기준 주소가 아님
```

| 구분 | 링커 내부 | 출력 ELF |
| --- | --- | --- |
| 심볼 정보 | 있음 | 보통 없음 |
| 형태 | 메모리 구조체 | ELF 섹션 |
| 목적 | 주소 계산, relocation | 디버깅/분석용 |

주소 배정을 위해 이 섹션 병합 단계에서 레이아웃을 만드는데, 

링커 스크립트를 기반으로 섹션 시작 주소 결정

```c
[링커 스크립트 파일 내용]
.text 0x08000000 : { *(.text*) }
.data 0x20000000 : { *(.data*) }

이 내용으로 아래 내용들이 결정됨
.text 시작 주소 = 0x08000000
.data 시작 주소 = 0x20000000
=> 이게 VMA (실행 주소), 임베디드 면 LMA도 같이 계산
```

링커 스크립트 참조해서 결정한 섹션 시작 주소 내용은 ELF 파일의 헤더 영역에 저장된다.

elf 파일에는 헤더 내용이 아래와 같이 3개가 있다 

```c
파일 시작
┌──────────────────┐
│ ELF Header       │
├──────────────────┤
│ Program Headers  │  ← 로더가 사용
├──────────────────┤
│ .text            │
│ .rodata          │
│ .data            │
│ .bss (NOBITS)    │
│ ...              │
├──────────────────┤
│ Section Headers  │  ← 디버거/링커용
└──────────────────┘
파일 끝
```

ELF Header

program header (실행 용)

section header (정보 용 )

이 중 program header + section header에 저장된다. 

```c
ELF Header 
- 항상 파일의 맨 앞에 존재 (0x0)
- elf 파일 정보를 담고 있는 영역
- 이 헤더 영역에서 다른 두 헤더의 위치를 가리킴
		Start of program headers
		Start of section headers

root@lazymarmot:~# readelf -h hello_world.elf
ELF Header:
  Magic:   7f 45 4c 46 01 01 01 00 00 00 00 00 00 00 00 00
  Class:                             ELF32
  Data:                              2's complement, little endian
  Version:                           1 (current)
  OS/ABI:                            UNIX - System V
  ABI Version:                       0
  Type:                              EXEC (Executable file)
  Machine:                           ARM
  Version:                           0x1
  Entry point address:               0x100000
  Start of program headers:          52 (bytes into file)
  Start of section headers:          170852 (bytes into file)
  Flags:                             0x5000400, Version5 EABI, hard-float ABI
  Size of this header:               52 (bytes)
  Size of program headers:           32 (bytes)
  Number of program headers:         1
  Size of section headers:           40 (bytes)
  Number of section headers:         25
  Section header string table index: 22
root@lazymarmot:~#
```

```c
Program Header
- ELF Header가 가리키는 위치에 있음
- 보통 파일 앞쪽에 있긴 한데 필수 순서는 아님
- 실행 할 때 로더/부트로더가 참조하는 헤더

root@lazymarmot:~# readelf -l hello_world.elf

Elf file type is EXEC (Executable file)
Entry point 0x100000
There is 1 program header, starting at offset 52
=> 파일 오프셋 0x34(52)부터 

Program Headers:
  Type           Offset   VirtAddr   PhysAddr   FileSiz MemSiz  Flg Align
  LOAD           0x010000 0x00100000 0x00100000 0x08008 0x0d830 RWE 0x10000

 Section to Segment mapping:
  Segment Sections...
   00     .text .init .fini .rodata .data .eh_frame .mmu_tbl .init_array .fini_array .bss .heap .stack
=>.text는 이 LOAD 세그먼트 안에 포함

[ LOAD segment ]
  ├─ .text
  ├─ .init
  ├─ .fini
  ├─ .rodata
  ├─ .data
  └─ ...

root@lazymarmot:~#

---------------------------------------------------------------------
offset -> ELF 파일 안에 데이터가 있는 위치("LOAD 세그먼트 파일 시작위치")
virtAddr -> VMA (실행 시 기준 주소, 메모리에 어디에 올릴지) ".text의 시작주소"
physAddr -> LMA (임베디드에서 중요, 안쓰면 보통 VMA랑 같거나 0일 수도 있음)
FileSiz/MemSiz -> ROM에 들어있는 크기 / RAM에서 필요한 크기(bss 때문에 MemSiz 더 커질 수 있음)

파일[0x010000]부터 읽어서 메모리 0x00100000에 올려라 

링커는 RAM에 로드될 주소(VMA)를 배치하며, 어디에서 읽어올지(LMA)는 환경에 따라 다르다
실행 중 CPU가 접근 할 주소 -> VMA

```

```c
Section Header
- ELF Header에서 가리키는 위치에 존재
- 보통 파일의 맨 뒤에 있음
- 실행에는 필요 없음

root@lazymarmot:~# readelf -SW hello_world.elf
There are 25 section headers, starting at offset 0x29b64:
=> 파일의 뒷부분(거의 끝) (170852) 
(ELF Header에서 섹션 헤더 테이블 전체 크기 40 * 25 = 1000 bytes)
(1000을 16진수로 바꾸면 0x3E8)
시작: 0x29B64
끝: 0x29B64 + 0x3E8 = 0x29F4C

Section Headers:
  [Nr] Name              Type            Addr     Off    Size   ES Flg Lk Inf Al
  [ 0]                   NULL            00000000 000000 000000 00      0   0  0
  [ 1] .text             PROGBITS        00100000 010000 000b6c 00  AX  0   0  4
  [ 2] .init             PROGBITS        00100b6c 010b6c 000018 00  AX  0   0  4
  [ 3] .fini             PROGBITS        00100b84 010b84 000018 00  AX  0   0  4
  [ 4] .rodata           PROGBITS        00100b9c 010b9c 000018 00   A  0   0  4
  [ 5] .data             PROGBITS        00100bb8 010bb8 000474 00  WA  0   0  8
  [ 6] .eh_frame         PROGBITS        0010102c 01102c 000004 00   A  0   0  4
  [ 7] .mmu_tbl          PROGBITS        00104000 014000 004000 00   A  0   0  1
  [ 8] .init_array       INIT_ARRAY      00108000 018000 000004 04  WA  0   0  4
  [ 9] .fini_array       FINI_ARRAY      00108004 018004 000004 04  WA  0   0  4
  [10] .ARM.attributes   ARM_ATTRIBUTES  00108008 018008 000033 00      0   0  1
  [11] .bss              NOBITS          00108008 018008 000028 00  WA  0   0  4
  [12] .heap             NOBITS          00108030 018008 002000 00  WA  0   0  1
  [13] .stack            NOBITS          0010a030 018008 003800 00  WA  0   0  1
  [14] .comment          PROGBITS        00000000 01803b 000036 01  MS  0   0  1
  [15] .debug_info       PROGBITS        00000000 018071 000998 00      0   0  1
  [16] .debug_abbrev     PROGBITS        00000000 018a09 0001ff 00      0   0  1
  [17] .debug_aranges    PROGBITS        00000000 018c08 000040 00      0   0  1
  [18] .debug_macro      PROGBITS        00000000 018c48 002c0c 00      0   0  1
  [19] .debug_line       PROGBITS        00000000 01b854 0006b6 00      0   0  1
  [20] .debug_str        PROGBITS        00000000 01bf0a 00be62 01  MS  0   0  1
  [21] .debug_frame      PROGBITS        00000000 027d6c 0000d4 00      0   0  4
  [22] .shstrtab         STRTAB          00000000 029a79 0000eb 00      0   0  1
  [23] .symtab           SYMTAB          00000000 027e40 0010e0 10     24 150  4
  [24] .strtab           STRTAB          00000000 028f20 000b59 00      0   0  1
Key to Flags:
  W (write), A (alloc), X (execute), M (merge), S (strings), I (info),
  L (link order), O (extra OS processing required), G (group), T (TLS),
  C (compressed), x (unknown), o (OS specific), E (exclude),
  D (mbind), y (purecode), p (processor specific)
root@lazymarmot:~#
```


| 네 그림 | 확인 명령 |
| --- | --- |
| ELF Header | `readelf -h` |
| Program Headers | `readelf -l` |
| .text/.data 실제 데이터 | `readelf -l`의 LOAD + `objdump -d` |
| Section Headers | `readelf -S` |
| 파일 배치 감각 | `hexdump -C` |

startup.s에는 주로 .text만 있고 .data가 없는 경우도 있는데 .data가 있으면

```c
.section .data
.global boot_flag
boot_flag:
    .word 1
```

startup.o에 .data가 생기고, 링커가 main.o, test.o, startup.o의 .data를 전부 합쳐서 elf의 .data 하나로 만든다.

또한 보통 startup.s 안에는 Reset Handler / 벡터 테이블 / 기타 초기화 루틴 내용들이 존재한다. 

실행되는 명령어들은 → .text 섹션

벡터 테이블 내용은 → .isr_vector 섹션

요렇게 섹션들이 나뉘어져 있는 경우도 있는데

```c
.section .isr_vector
  .word _estack
  .word Reset_Handler

.section .text
Reset_Handler:
  ...
  bl main
```

이름이 .text랑 다르기 때문에 

링커 스크립트에서 위치를 강제해줘야 한다. 

```c
.isr_vector 0x00000000 : { *(.isr_vector) }
```

그러나 링커 스크립트가 아래처럼 작성되어 있으면, 

```c
.text :
{
  *(.isr_vector)
  *(.text*)
}
```

메모리 상에는 전부 .text 영역에 연속으로 배치되게 된다. 

1. 심볼 해석 + 심볼 값 확정
    
    “결정되지 않은 심볼을 찾아 값을 결정”하는 단계
    
    ```c
    // a.c
    extern int x;
    void foo() { x = 1; }
    
    // b.c
    int x;
    ```
    
    a.o에서 x는 미정
    
    b.o에서 x는 정의됨
    
    링커의 역할 : “이 x는 b.o에 있네” → 주소 연결
    
    **심볼 해석**
    
    각 오브젝트 파일에는 이미 .symtab 영역이 있었고, 이 안에 각 심볼마다 아래의 정보들이 들어있었다 
    
    | 심볼이름 |
    | --- |
    | 심볼타입 |
    | 심볼 바인딩 ( STB_*) |
    | 심볼이 속한 섹션 |
    | 섹션 내부 offset(st_value) |
    
    이러한 내용을 수집해서 만든,
    
    심볼 테이블에도 배치나 재배치를 할 때 필요한 “프로그램의 심볼 정의”나 “참조 값에 대한 정보” “심볼 바인딩 정보”가 들어있다. 
    
    **심볼 테이블 예시**

    | 항목 | 설명 |
    | --- | --- |
    | 이름 | foo, bar, g_var |
    | 바인딩 | STB_LOCAL / GLOBAL / WEAK |
    | 정의 여부 | defined / undefined |
    | 최종 섹션 | .text / .data / .bss |
    | 최종 주소 | 0x80001000 |
    | 가시성 | local/global |
    
    <br>
    
    심볼 바인딩을 기반으로 심볼 해석, 심볼 값 확정을 하게 되는데 이때,
    
    링커가 하는 질문들
    
    - foo 참조 발견
    - 여러 .o 중 정의된 foo 선택 및 바인딩 규칙 적용
    - STB_LOCAL? → 파일 내부에서만 찾고 끝
    - STB_GLOBAL? → 다른 .o 까지도 탐색
    - STB_WEAK? / GLOBAL → GLOBAL 이 있으면 양보
    
    이걸로 심볼 ↔ 정의 매칭이 결정된다.
    
    예를 들어)
    
    foo라는 함수 안에서 다른 파일의 bar 함수를 호출한다고 하자.
    
    - foo 안의 bar 참조를 보고
    - 어떤 오브젝트 파일의 bar 정의를 쓸지 결정
    - STB_LOCAL/GLOBAL/WEAK 바인딩 규칙 적용
    
    ⇒ “이 bar 호출은 이 정의(bar@b.o)를 사용한다” (결과)
    
    ---
    
    심볼 바인딩(STB_*)은 링크 과정에서 각 심볼을 참조할 수 있는 범위나 우선 순위 등을 결정하도록 도와준다.
    
    | 이름 | 설명 | 값 |
    | --- | --- | --- |
    | STB_LOCAL | 파일 내부에서만 보임<br>static 함수/변수<br>다른 .o 에서 절대 접근 불가<br>static int a;<br>static void f(void); | 0 |
    | STB_GLOBAL | 모든 .o에서 참조 가능<br>다른 파일에서 참조하는 글로벌 심볼 / 다른 파일에서 정의되고, 현재 파일에서 참조하는 글로벌 심볼로 나뉨<br>int g;<br>void foo(void); | 1 |
    | STB_WEAK | 글로벌이긴 한데 우선순위가 낮음<br>같은 이름의 GLOBAL이 있으면 자동으로 밀림<br>여러개 있어도 에러가 아님<br><br>// weak.c<br>int __attribute__((weak)) x;<br>// strong.c<br>int x;<br>⇒ x는 strong쪽을 선택하고 weak는 무시됨 | 2 |
    | STB_LOPROC / STB_HIPROC | 아키텍처/프로세서의 사용을 위해 예약된 값 | 13/15 |
    

    같은 이름의 글로벌 심볼 규칙
    
    ```c
    // a.c
    int x;
    
    // b.c
    int x;
    
    -> 링크 에러 발생
    
    // a.c
    int x;
    
    // b.c
    int __attribute__((weak)) x;
    -> 링크 에러 발생 안함
    ```
    
    컴파일러는 보통 초기화된 전역 변수와 함수들은 global 심볼로 만들고
    
    초기화 되지 않은 전역 변수들은 약한 심볼로 만든다.
    
    ```c
    초기화 안된 전역변수
    int x;   // 보통 weak 취급
    
    초기화 되지 않은 전역 변수는 약한 심볼로 만든다 
    common symbol 관행 때문(gcc 옵션 -fno-commin이랑 연결됨)
    ```
    
    ```c
    foo  = .text + 0x20
    gvar = .data + 0x04
    cnt  = .bss  + 0x00
    
    여기까지가 "심볼 값 확정"
    
    주소 배정에서 주소가 생겼으니
    foo = 0x80000000 + 0x10
    bar = 0x80000000 + 0x30
    심볼 바인딩 규칙 적용됨
    ```
    
    ---
    
    **심볼 확정**
    
    그리고 나서 심볼 값 확정 단계에서는 이 심볼의 값(st_value)을 최종 주소 숫자로 고정한다. 
    
    이 단계에서 링커는 숫자를 만든다.
    
    - 입력 상태
        - foo가 .text에 있음
        - foo의 섹션 내부 offset=0x20
        - .text 시작주소 = 0x80000000
    - 확정 단계에서 계산
        
        ```c
        foo.value = 0x80000000 + 0x20
                  = 0x80000020
        ```
        
    
    섹션 병합 하면서 확정된 섹션 시작 주소를 기준으로,
    
    섹션 내부 offset + 시작 주소 ⇒ 심볼 절대 주소 계산 및 확정한다. 
    
    이 단계 이후 링커 내부 심볼 테이블이 아래 처럼 변경됨
    
    ```c
    foo → 0x08000000 + offset_a
    bar → 0x08000000 + offset_b
    
    예를 들어
    	offset_a = 0x40
    	offset_b = 0x120
    	
    	이러면
    	
    	foo = 0x08000040
    	bar = 0x08000120
    	요렇게 변경됨 (처음으로 절대주소가 생김)
    ```
    
    섹션 병합 하면서 확정된 섹션 시작 주소를 기준으로,

    섹션 내부 offset + 시작 주소 ⇒ 심볼 절대 주소 계산 및 확정한다. 

    이 단계 이후 링커 내부 심볼 테이블이 아래 처럼 변경됨

    ```c
    foo → 0x08000000 + offset_a
    bar → 0x08000000 + offset_b

    예를 들어
	    offset_a = 0x40
	    offset_b = 0x120
	
	    이러면
	
	    foo = 0x08000040
	    bar = 0x08000120
	    요렇게 변경됨 (처음으로 절대주소가 생김)
    ```

    결론:

    이미 정해진 레이아웃 기준으로

    ```c
    bar_addr = .text_start + bar_offset
    ```

    - bar의 절대 주소 숫자를 계산/확정
    - 내부 심볼 테이블에 저장

<br>

3. 재배치 (Relocation)
    
    심볼 정의와 심볼 참조를 연결하는 과정
    
    프로그램에서 함수를 호출하면 함수 분기 명령을 적합한 목적 주소로 변환하는 작업을 하는 단계
    
    relocation 리스트 순회 하면서  위에 실제 패치 하는 단계
    
    “심볼테이블에서 심볼의 절대주소를 계산해두고 relocation 리스트를 순회하면서 계산된 심볼 주소를 코드/데이터에 실제로 패치 한다. 
    
    ```c
    (.text + 0x24) → bar
    
    ".text 섹션 시작 + 0x24 위치에 bar의 주소를 써라"
    
    패치 위치 = 0x08000000 + 0x24
    써야 할 값 = 0x08000120
    ```
    
    relocation 까지 다 끝나면 결국 .text 안의 호출/참조는 실제로 타겟 함수로 가도록 값이 채워진 상태가 된다.
    
    ```c
    함수 호출
    call foo
    ⬇️
    call 0x80000000   (또는 PC-relative offset)
    
    전역 변수 접근
    ldr r0, =g_var
    ⬇️
    ldr r0, =0x80010000
    
    ```
    
    재배치가 왜 필요할까?
    
    ```c
    컴파일러 입장에서 보면 
    void foo();
    void bar() {
        foo();
    }
    
    이런 코드를 보고 컴파일 할 때 컴파일러는 이렇게 생각함
    	"foo는 있긴한데 아직 어디 있는지는 모름"
    
    그래서 기계어 만들 때 
    call foo    ; 주소 모름
    주소 자리에 빈칸을 두고 나중에 채우라는 메모를 남기는데
    이 메모가 relocation entry임
    
    #########################
    relocation entry
    #########################
    (REL 구조 addend(보정값)이 기계어 안에 있음, x86에서 주로 사용)
    	typedef struct {
    	    Elf32_Addr r_offset; // 어디를 고칠지
    	    Elf32_Word r_info;   // 어떤 심볼을, 어떤 방식으로
    	} Elf32_Rel;
    
    r_offset : 섹션 안에서 몇 바이트 위치를 고칠지
    	ex) .text + 0x14 
    r_info : 두 정보가 합쳐져 있음
    	1. 어떤 심볼인지
    	2. 어떻게 고칠지 (절대주소?...)
    ```
    
    ---
    
    링커가 재배치 하는 실제 흐름을 보면
    
    ```c
    // b.c
    void foo() {
        bar();
    }
    ```
    
    ```c
    // c.c
    void bar() {
    }
    ```
    
    - `.text` 시작 주소 = `0x08000000`
    - `foo` offset = `0x40` → `foo = 0x08000040`
    - `bar` offset = `0x120` → `bar = 0x08000120`
    - relocation : (.text + 0x24) → bar
    
    relocation 적용 전의 .text파일
    
    ```c
    주소        기계어 / 의미
    --------------------------------
    0x08000040  push {lr}
    0x08000044  bl 0x00000000    ← 아직 bar 주소 모름 (더미값)
    0x08000048  pop {lr}
    0x0800004C  bx lr
    ```
    
    3 단계가 끝나고 나면
    
    relocation 정보 → 필요 없어짐 → 제거
    
    링커 내부 심볼 테이블 → 작업 끝 → 제거
    
    남는 것은 : 
    
    주소가 박힌 .text / .data / .bss
    
    진짜 실행 가능한 ELF 파일
    
    
이후 타겟 보드에 로딩

