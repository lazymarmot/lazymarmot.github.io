---
layout: post
title: "메모리 최적화"
date: 2026-03-05 12:00:00 +0900
categories: Code Optimization
show_on_home: true
---

# 메모리 최적화

![image.png](/assets/posts_image/Code_Optimization/Memory%20Optimization/image.png)

## **※ ROM 최적화**

ROM 최적화는 실행 파일을 줄이는 노력을 해야 함

1. 데드 코드 제거
    
    사용되지 않는 계산, 도달하지 않는 코드 같은 것들을 제거하는 것
    
    데드 코드 제거는 컴파일러의 최적화 옵션에 의해 수행됨 
    
    ⇒ 컴파일러가 알아서 필요 없는 코드 선별해서 제거한다는 의미
    
    ```jsx
    for(i=0; i<8; i++)
    {
    		display(i);
    		for(j=0; j<10000; j++);
    }
    ```
    
    최적화 옵션 -O2 로 컴파일하면 	”for(j=0; j<10000; j++);” 요 부분은 컴파일러가 제거 해버리는데, 
    
    이걸 예방하려면 j를 volatile을 사용하면 최적화 하지 않음
    
2. 매크로나 인라인 사용하지 않기
    
    “속도 최적화”에서는 분기가 성능을떨어트리니까, 간단한 함수는 매크로나 인라인 함수로 대체하면 좋음 
    
     ROM 메모리가 부족한 경우는 매크로, 인라인 말고 함수 사용하는게 좋음
    
3. 전역 변수 초기화 하지 말기
    
    전역/정적 변수 초기화해서 사용하면 ROM, RAM 두 메모리에 모두 저장되기 때문에 ROM을 절약 하려면 전역/정적 변수의 초기화를 생략해서 bss 세그먼트에 저장되도록 하는게 좋음
    
4. 상수나 전역 변수 대신 지역 변수 사용 
    
    ROM 절약을 위해 전역 변수를 지역 변수로 대체하는게 좋고,  상수 역시 코드 크기를 늘리기 때문에 변수로 대체하는 편이 좋음
    
5. 표현은 간결하하고 불필요한 중간 과정은 생략

## **※ RAM 최적화**

스택을 얼마나 효율적으로 사용하는 지가 RAM 최적화의 관건

1. 함수 호출은 깊지 않게
    
    ![stack_frame추가.jpg](/assets/posts_image/Code_Optimization/Memory%20Optimization/stack_frame%EC%B6%94%EA%B0%80.jpg)
    
    스택 프레임 참고 
    
    https://www.tcpschool.com/c/c_memory_stackframe
    
    https://justicehui.github.io/unifox/2018/09/30/Unifox_Basic/
    
    스택프레임 구조에 따라 함수에서 다른 함수를 호출하면 새로운 함수가 사용할 메모리를 스택에서 또 할당 받아야 되고, 복귀주소, 사용중인 레지스터의 내용들, 매개변수, 지역변수 등 많은 내용을 스택에 저장해야 하니 RAM 메모리의 사용량이 늘어난다.
    
    그러니 함수의 호출 깊이가 깊어질수록 오버헤드가 증가하고 스택 사용량이 늘어나기 때문에 깊이를 깊게 가져가지 않는 편이 좋다.
    
2. 함수는 매크로나 인라인을 대체
    
    함수는 스택 사용량이 크기 때문에 RAM이 부족한 경우 혹시 ROM이 넉넉하다면 함수 대신 매크로나 인라인을 사용해서 RAM을 좀 절약 할 수 있다.
    
      
    
3. 구조체나 배열 대신 포인터 사용
    
    구조체는 대부분 크기가 큰 편이다.
    
    함수의 인자나 리턴 값이 구조체 일 때 값에 의한 호출을 한다면, 메모리나 속도 손실이 크다
    
     포인터를 인자로 넘겨 스택 메모리 사용을 줄이는 편이 좋다.
    
4. 비트플래그 사용
    
    인터럽트 발생 여부 / 응용프로그램에서 데이터 검색 시 검색 여부를 나타내는 플래그..
    
    이런 플래그를 만들 때 int flag; 이렇게 만들지 말고 1비트로 충분히 표현 가능하기 때문에 보통 비트 플래그를 사용하면 된다.    
    
    ```c
    #include <stdio.h>
    
    struct SR
    {
    		unsigned int CF : 1,       // unsiged int를 1비트만 쓰겠다는 의미
    						PF : 1,
    						AF : 1,
    						ZF : 1,
    						SF : 1,
    						IF : 1,
    						MOD : 4;
    };
    
    void main(void)
    {
    		struct SR flag = {1, 1, 0, 1, 0, 1, 0xF};
    		
    		if(flag.CF)
    				printf("carry\n");
    		if(flag.PF)
    				printf("parity\n");
    		if(flag.AF)
    				printf("assistant carry\n");
    		if(flag.ZF)
    				printf("zero\n");		
    }
    ```
    
    ```c
    | CF | PF | AF | ZF | SF | IF | MOD(4bit) | 남는 비트들 |
    
    1+1+1+1+1+1+4 = 10비트
    
    근데 unsigned int 가 32비트니까, 실제 메모리는 4바이트 하나를 통째로 잡고 그 안에서 10비트만 사용
    
    	unsigned int (32bit)
    	[CF][PF][AF][ZF][SF][IF][MOD(4bit)][unused 22bit]
    
    ```
    
    ```c
    unsigned int CF;
    unsigned int PF;
    이건 진짜 4byte + 4byte = 8byte
    
    unsigned int CF : 1;  
    unsigned int PF : 1;
    이건 4byte안에서 2bit 만 씀.
    
    unsigned int CF : 1 
    이거는, "unsigned int" 타입 변수 하나 생성이 아니라
    "unsigned int"를 저장 단위(container)로 쓰겠다는 의미로
    컴파일러가 
    unsigned int (32bit짜리 저장공간 하나 만들고)
    그 안에 비트를 순서대로 채워 넣음
    	bit0  → CF
    	bit1  → PF
    	bit2~31 → unused
    
    왜 합쳐지냐면
    C 표준 규칙으로 
    	같은 기반 타입(unsigned int)이고 아직 남는 비트가 있으면 같은 저장 단위 안에 
    	이어서 채운다
    
    만일 아래와 같으면?
    unsigned int CF : 31;
    unsigned int PF : 2;
    31비트 쓰고 1비트 남았는데 PF는 2비트 필요하니까 -> 새로운 unsigned int 컨테이너 생성 -> 그래서 8바이트 됨
    ```
    
    이런 방식은 하드웨어 설정에서 유용하게 사용될 수 있으며, 특히 메모리를 절약 하는데 효과적이다.  
    
    하지만 개별 비트를 처리해야 하기 때문에 내부적으로 shift나 masking연산에 의한 오버헤드가 발생하여 속도가 조금 떨어질 수도 있다.  
    
5. 값이 변하지 않는 전역 변수의 상수 화 
    
    값이 바뀌지 않는 전역 변수라면 const 키워드를 사용해 상수화 하는게 좋다.
    
    전역 변수를 상수화 하면 RAM에 복사하는 것을 막을 수 있기 때문이다.