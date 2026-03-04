---
layout: post
title:  "AXI3 Memory Attribute Signaling"
date:   2026-03-04 12:00:00 +0900
categories: AXI4 AMBA
show_on_home: false
---

# AXI3 memory attribute signaling

![image.png](/assets/posts_image/AXI/AXI3%20memory%20attribute%20signaling/image.png)

![image.png](/assets/posts_image/AXI/AXI3%20memory%20attribute%20signaling/image%201.png)

### **AxCACHE 비트 설명**

AxCACHE[0] — Bufferable (B) 비트

이 비트가 assert(1) 되면, 

인터커넥트 또는 다른 어떤 컴포넌트라도 이 트랜잭션이 최종 목적지에 도달하는 것을 임의의 사이클 동안 지연 시킬 수 있다.

<aside>
💡

참고

Bufferable 속성은 보통 쓰기(write) 트랜잭션에서만 의미가 있다.

</aside>

AxCACHE[1] — Cacheable (C) 비트

이 비트가 deassert(0) 되면,

해당 트랜잭션에 대해 캐시 할당(allocation)은 금지된다.

이 비트가 assert(1) 되면:

- 캐시 할당이 허용된다 (RA, WA 비트가 추가적인 힌트를 제공한다)
- 트랜잭션이 최종 목적지에서 처리되는 방식은 원래 트랜잭션의 특성과 동일할 필요가 없다

이 의미는 다음과 같다:

- 쓰기(write) → 여러 개의 서로 다른 write를 하나로 병합(merge) 할 수 있다
- 읽기(read) → 해당 위치의 데이터를 미리 가져올 수(prefetch) 있고, 한 번 가져온 값을 여러 read 트랜잭션에서 재 사용할 수 있다.

AxCACHE[2] — Read-allocate (RA) 비트

이 비트가 assert(1) 되면,

읽기(read) 트랜잭션에서 캐시 할당이 권장된다 (단, 필수는 아니다).

제약 조건

C 비트가 0일 때는

RA 비트를 assert해서는 안 된다.

AxCACHE[3] — Write-allocate (WA) 비트

이 비트가 assert(1) 되면,

쓰기(write) 트랜잭션에서 캐시 할당이 권장된다 (역시 필수는 아니다).

제약 조건

C 비트가 0일 때는

WA 비트를 assert해서는 안 된다.

[Cache Attribute](AXI3%20memory%20attribute%20signaling/Cache%20Attribute%202e96feb16a3e80219ca7d318097a2895.md)