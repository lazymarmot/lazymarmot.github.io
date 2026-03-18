---
layout: post
title:  "Code Optimization Notes"
date:   2026-03-05 11:00:00 +0900
categories: Code Optimization
---

C 코드 개발 시 최적화를 위한 메모리 사용, 포인터, 동적 할당, 구조체/포인터 활용 등 상세 내용을 다룬다.
근데 주로 레거시 최적화 기법들을 설명한 부분도 많아서 최신 CPU나 고급 최적화 컴파일러에서는 옵션을넣으면 자동적으로 처리되는 부분도 있다. 
단순히 최적화가 왜필요한지, 어떻게 이루어지는지를 염두해두고 보면 될 것 같다.
(Blog2Book의 "임베디드 프로그래밍 C 코드 최적화" 라는 책을 기반으로 정리한 내용들이다. )

- [코드 최적화]({% post_url Code_Optimization/2026-03-05-1-code-optimization %})
- [C의 메모리 사용]({% post_url Code_Optimization/2026-03-05-2-c-memory-usage %})
- [변수 타입 사용]({% post_url Code_Optimization/2026-03-05-3-c-variable-types %})
- [데이터를 효율적으로 사용]({% post_url Code_Optimization/2026-03-05-4-efficient-data-usage %})
- [메모리 최적화]({% post_url Code_Optimization/2026-03-05-5-memory-optimization %})
- [함수는 어떻게 사용할까?]({% post_url Code_Optimization/2026-03-05-8-c-functions-usage %})
- [분기문은 어떻게 쓰면 좋나]({% post_url Code_Optimization/2026-03-05-7-conditional-statements-usage %})
- [루프는 어떻게 쓰면 좋을까]({% post_url Code_Optimization/2026-03-05-6-c-loop-usage %})
- [최적화 하기 위한 표현들]({% post_url Code_Optimization/2026-03-05-9-optimization-expressions %})
