---
layout: post
title: "코드 최적화"
date: 2026-03-05 12:00:00 +0900
categories: Code Optimization
show_on_home: false
---

# 코드 최적화

1. 포인터 활용과 최적화
    
    ```c
    #include <stdio.h>
    
    void main()
    {
    		char a[] = "apple";
    		char *p = "grace";
    		a[1] = 'k';
    		p[1] = 'k';
    		printf("%s\n", a);
    		printf("%s\n", p);
    }
    ```
    
    ```nasm
    5:           a[1] = 'k';
    00401041     mov       byte ptr  [ebp-7],6Bh
    6:           p[1] = 'k';
    00401045     mov       eax,dword ptr  [ebp-0Ch]
    00401048     mov       byte ptr  [eax+1],6Bh
    ```
    
    5, 6 라인 코드로 봐서는 같은 일을 수행하는데 역어셈을 보면 
    
    배열의 1번째 인덱스에 접근하여 값을 쓸 때는 mov 명령 하나로 처리되는데 포인터 p의 1번째 인덱스에 접근하여 값을 쓸 때는 mov 명령이 2개 사용되었다. 
    
    배열에 접근 : 그 주소에 직접 접근해서 값을 씀
    
    a[1] 에 접근 ⇒ acc[0x104]
    
    포인터 접근 : 포인터가 가진 주소를 읽어와 1번째 인덱스이기 때문에 그 주소에서 한 단위를 증가시킨 위치에 값 씀
    
    p[1]에 접근 ⇒ r0 ← p
    
                       acc ← [r0, #4]
    
    a[1]에 접근 할 때에는 그 주소를 바로 acc(계산용 레지스터)로 가져올 수 있는데, p[1]에 접근할 때에는 r0 레지스터에 p의 값을 로드 하고 p 값은 배열의 시작 주소니까, 주소에 4byte를 더한 1번 인덱스 위치를 acc에 저장한다. 
    
    ⇒ 배열의 특정 위치에 접근 하는건 명령어 하나로 처리 되는데, 포인터를 이용하여 메모리의 특정 위치에 접근 하는 것은 명령어 2개로 처리가 됨 (포인터보다는 배열이 빠르다고 볼 수 있음)
    
    배열이 속도가 빠르긴 하지만, 메모리 사용에 있어서 배열보다 포인터가 좀 더 효과적이다.
    
    아래 그림을 참조 해서 코드를 다시 본다면, 
    
    배열 a는 스택에 문자열의 크기만큼 메모리를 할당 받고 상수 영역의 “apple” 문자열을 복사하지만, 포인터 p는 상수 영역에 있는 문자열의 시작 주소 만을 저장한다.
    
    ⇒ 문자열의 길이가 다양한 여러개의 문자열을 저장해야 하는 경우
    
    파일에 접근해서 데이터를 읽어와 처리해야 하는 경우 
    
    데이터의 크기, 개수가 일정하지 않아 배열보단 포인터를 사용하는게 효과적이다. 
    
    ⇒ 배열은 사용전 크기를 지정해야 하며 크기를 정확히 알지 못할 때에는 메모리 낭비를 가져올 수 있음, 파일을 열어보기 전 몇 라인이 들어있는지 알 수 없는 부분도 있음 
    
    ![image.png](/assets/posts_image/Code_Optimization/Code%20Optimization/image.png)
    
    ※ 문자열 저장 방법
    
    apple랑 grace는 문자열 상수이다. 
    
    ⇒ 상수는 text 세그먼트에 저장 되기 때문에 문자열 상수 또한 text 세그먼트에 저장된다. 
    
    이 영역은 값이 바뀔 수 없는 영역으로 문자열이 저장되면 시작 주소를 이용해 접근할 수 있다.
    
    우선 apple라는 문자열이 텍스트 세그먼트에 저장되고, 
    
    그 시작 주소를 이용해 문자열을 불러와 스택에 만들어진 배열 a에 다시 복사 된다.  
    
    (만일 apple라는 문자열이 이전에 사용되었다면, 동일한 문자열을 text세그먼트에 두번 저장하는게 아니라 이미 존재하는 문자열 상수의 시작 주소를 넘긴다.)
    
    다음 grace라는 문자열을 text 세그먼트에 저장하고, 그 시작 주소를 포인터 p에 저장한다.
    
    a, p 모두 1번 인덱스에 접근해 글자를 바꾼다고 할 때, 
    
    a 배열은 문자열을 복사해서 스택에 저장하기 때문에 문제가 없는데, 
    
    포인터의 주소를 이용해서 글자 바꾸는건 text 세그먼트는 수정이 불가능한 메모리 영역이라 실행 시 오류가 발생될 것이다. 실행이 된다 쳐도 안 바뀔거다.
    
    %s는 문자열의 시작 주소를 넘겨 받아 한 문자 씩 널 문자(\0)를 만날 때까지 출력하는 서식이다.
    
2. 포인터와 구조체 사용
    
    구조체처럼 부피가 큰 내용들을 함수의 인자로 넘기는건 프로그램의 성능을 떨어트린다. (속도, 메모리 효율 떨어짐)
    
    그래서 이런 경우 주로 포인터 활용한다. 
    
    ```c
    <값에 의한 호출>
    #include <stdio.h>
    void f1(struct Test x);
    
    struct Test{
    	int a;
    	float b;
    	char c;
    };
    
    void main(void)
    {
    	struct Test t1;
    	t1.a = 3;
    	t1.b = 3.3;
    	t1.c = 'x';
    	f1(t1);
    }
    
    void f1(struct Test x)
    {
    	printf("%d, %f, %c \n", x.a, x.b, x.c);
    }
    ```
    
    ```nasm
    15:        f1(t1);
    0040104A     sub            esp,0Ch
    0040104D     mov            eax,esp
    0040104F     mov            ecx,dword ptr  [ebp-0Ch]
    00401052     mov            dword ptr [eax],ecx
    00401054     mov            edx,dword ptr  [ebp-8]
    00401057     mov            dword ptr [eax+4],edx
    0040105A     mov            ecx,dword ptr  [ebp-4]
    0040105D     mov            dword ptr [eax+8],ecx
    00401060     call           @ILT+15(f1)  (00401014)
    00401065     add            esp,0Ch
    
    ```
    
    ```c
    <참조에 의한 호출>
    #include <stdio.h>
    void f1(struct Test *x);
    
    struct Test{
    	int a;
    	float b;
    	char c;
    };
    
    void main(void)
    {
    	struct Test t1, *p;
    	p = &t1;
    	t1.a = 3;
    	t1.b = 3.3;
    	t1.c = 'x';
    	f1(p);
    }
    
    void f1(struct Test *x)
    {
    	printf("%d, %f, %c \n", x->a, x->b, x->c);
    }
    ```
    
    ```nasm
    16:        f1(p);
    00401050     mov            ecx,dword ptr  [ebp-10h]
    00401053     push           ecx
    00401054     call           @ILT+10(f1)  (0040100f)
    00401059     add            esp,4
    ```
    
    “값에 의한 호출”은 함수에 인자를 넘길 때 값을 복사해서 넘긴다.
    
    : 바로 아래의 역어셈을 보면 구조체 멤버의 값들을 스택에 저장하는 것을 알 수 있다
    
    : 구조체의 크기가 12byte 이므로 4byte씩 데이터를 읽어와 스택에 저장하는 동작을 3번 수행한다. 
    
    : main 함수가 사용하는 스택에서 데이터를 읽어오는 명령, 읽어온 데이터를 f1()함수가 사용하는 스택에 저장하는 명령이 필요해서 4byte의 데이터를 복사하는데 2개의 명령이 필요하다.
    
    : 그래서 12byte를 인자로 넘기려고 지금 6개 이상의 명령을 사용하고 있다
    
    “참조에 의한 호출”을 보면 
    
    : 포인터는 4byte이고, 포인터 값을 읽어 스택에 저장하는 데에는 두 개의 명령어만 필요하다. 
    
    결과적으로, 크기가 12 byte인 데이터를 함수의 인자로 넘기는 것은 포인터를 인자로 넘기는 거에 비해 3개 이상의 명령이 필요하고 
    
    → 재수 없으면 3 배 이상의 시간이 더 걸릴 수도 있다는 말이 된다. (극단적으로 말해서)
    
    ⇒ 부피 큰 데이터를 함수 인자로 넘길 때는 포인터를 사용하는게 속도/메모리에서 모두 효과적이다.
    
    만일 “참조에 의한 호출” 예제에서 f1() 함수가 포인터를 이용해 main의 변수 t1의 값을 변경하지 않고, 값을 읽어가는 것만 하고 싶은 경우  ⇒ const 키워드 사용
    
    ```c
    #include <stdio.h>
    void f1(const struct Test *x);
    
    struct Test{
    	int a;
    	float b;
    	char c;
    };
    
    void main(void)
    {
    	struct Test t1, *p;
    	p = &t1;
    	t1.a = 3;
    	t1.b = 3.3;
    	t1.c = 'x';
    	f1(p);
    }
    
    void f1(const struct Test *x)
    {
    	printf("%d, %f, %c \n", x->a, x->b, x->c);
    }
    ```
    
    <aside>
    💡
    
    const는 변수를 상수화 시키는 키워드라서
    
    const int *p 라는 표현은 포인터 p가 가리키는 메모리의 값을 변경 할 수 없다는 의미이다.
    
    그러므로 f1함수는 포인터 x를 통해 구조체  t1의 값은 읽어올 수 있는데 그 값을 변경 시킬 수는 없다.
    
    이런 방법은 주로 라이브러리 함수에서 많이 사용해, 외부 함수에서는 읽기만 가능하게 하고 외부에서 값이 변경되는 사고를 막아준다. 
    
    </aside>
    
3. heap 영역에 접근하기
    
    임베디드 환경에서 heap 영역에 접근하려면 주소 이용해서 직접 접근 하거나, malloc함수를 사용하거나, malloc이 없으면 직접 만들거나..
    
    < malloc 함수 >
    
    malloc 함수를 이용해 메모리를 할당 받는 것을 동적 할당이라 한다. 
    
    할당 받는 메모리의 크기가 컴파일 시 결정 되는게 아니라, 실행 시 결정 된다.
    
    주로 저장할 데이터가 몇 개 인지 예측 어렵거나, 배열 크기 지정이 어려울 때 동적 할당을 한다. 
    
    ```c
    <test.c>
    
    #include <stdio.h>
    #include <string.h>
    #include <stdlib.h>
    
    #define MAX 256
    
    //연결 리스트에 사용할 구조체 정의
    struct sample
    {
    	char line[MAX];  // 데이터를 저장할 멤버
    	struct sample *next;  // 다음 노드의 주소를 저장할 멤버
    };
    
    void main(void)
    {
      FILE *fp; // 파일구조체 포인터 선언
      char buf[MAX];
      
      struct sample *head, *tmp, *tail;
      
      fp = fopen("test.c", "r");
      
      if(fp == NULL)
      {
         printf("file not found");
         exit(1);
      }
      
      // 연결 리스트 생성 전 초기화. 노드가 하나도 없음 나타냄
      head = NULL;
      
      // 파일의 끝을 만날 때까지 한 라인 씩 읽어 buf 배열에 저장
      while(fgets(buf, MAX, fp) != NULL)
      {
    		  // malloc 이용해 struct sample크기 만큼의 메모리 할당 받아 
    		  // tmp에 주소 저장 => 연결 리스트 노드 생성
    			tmp = (struct sample*)malloc(sizeof(struct sample));		
    			
    			// 파일로부터 읽어들인 한 라인을 tmp의 멤버인 line에 복사
    			strcpy(tmp->line, buf);
    			
    			if(head == NULL)
    				head = tmp;
    			else
    				tail->next = tmp;
    				
    			tail = tmp;
    			tail->next = NULL;			
      }
      
      close(fp);
      
      tmp = head;
      while(tmp->next != NULL)
      {
    		  printf("%s", tmp->line);
    		  tmp = tmp->next;
      }
      
      tmp = head;
      while(tmp != NULL)
      {
    		  head = head->next;
    		  free(tmp);
    		  tmp = head;
      }
    }
    ```
    
    만일 파일에서 텍스트를 한 라인씩 읽어서 저장하는 프로그램을 만들 때, 
    
    각 라인을 배열에 저장한다고 하면, 사용하기 전에 배열을 선언해야 된다.
    
    근데, 파일 안에 텍스트가 몇 라인인지는 파일을 열어보기 전에는 알 수가 없고, 파일을 열어서 일일이 라인을 세어서 배열 크기를 결정 하는걸 누가 하겠는가??
    
    배열은 컴파일 단계에서 이미 정해져야 하는 건데, 지금같이 런타임 시 크기가 결정 될 때는 배열을 사용할 수 없고, 메모리를 동적으로 할당 받아야 한다. 
    
    ⇒ 동적으로 메모리 할당 받으려고 할 때 메모리가 배열처럼 연속적으로 할당되지 않아서, 할당 받은 메모리 주소를 몽땅 저장해 두어야 다시 그 메모리에 접근 할 수 있다
    
    ⇒ 자료구조 알고리즘 중 연결 리스트 알고리즘 사용 하는게 좋다
    
4. 포인터
    
    
    <포인터 체인을 제거해서 포인터를 빠르게 할 수 있다>
    
    구조체 포인터를 사용하다 보면 구조체 멤버의 멤버의 멤버에 접근하는 것처럼 포인터가 체인을 형성하는 일이 빈번하게 발생한다. 
    
    포인터 체인(a→p1→x)은 CPU가 매번 메모리를 더 많이 읽어야 해서 비효율적이라서, 로컬 포인터로 끊어주면 더 빠르고 안전하다.
    
    | 포인터 체인 | 포인터 체인 제거 |
    | --- | --- |
    | struct Point 
    {
         int x, y, z;
    };
    
    struct Obj 
    {
        Point *p1, *d;
    };
    
    void draw(struct Obj *a)
    {
        a→p1→x = 0;
        a→p1→y = 0;
        a→p1→z = 0;
    } | struct Point 
    {
         int x, y, z;
    };
    
    struct Obj 
    {
        Point *p1, *d;
    };
    
    void draw(struct Obj *a)
    {
        struct Point *k = a→p1;
        k→p1→x = 0;
        k→p1→y = 0;
        k→p1→z = 0;
    } |
    |   • a주소 읽기
      • a→p1 주소 읽기
      • p1이 가리키는 Point 구조체 주소 읽기
      • 거기서 x 오프셋 계산
      • 그 위치에 0 저장 |   • a→p1 한번만 읽어서 
      • 로컬 변수 k에 저장
     |
    | 포인터 역참조가 2번 들어감
    이걸 x,y,z 마다 반복 
    매번 a→p1을 다시 타고 들어감 | k→x, k→y, k→z 로컬 변수로 읽음
    포인터 체인이 1단계로 줄음
    메모리 접근 횟수 감소 |