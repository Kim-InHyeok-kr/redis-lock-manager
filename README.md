# O2O 연동 – 서버 간 주문 생성 & 클라이언트 실시간 반영

## 📘 개요
본 문서는 O2O 연동 환경에서 서버→서버로 주문이 생성될 때, Redis 분산락을 이용해 Redis ↔ DB 동기화 과정의 경쟁 상태를 제거하고,
생성된 주문을 클라이언트에 실시간 전달(SSE/WebSocket)하는 레퍼런스 구현을 제공합니다.

---

## ⚙️ 문제 원인
1. 4초 주기의 polling에서 “Redis ↔ DB 불일치”를 신규 주문으로 오인함  
2. 주문 생성 후 RabbitMQ 발행 전에 polling이 먼저 수행되어 중복 판단 발생  
3. polling이 준실시간 이벤트를 흉내내고 있어 타이밍 의존성이 큼

---

## 🧩 해결 방안
- Redis 분산락(Redisson)으로 DB↔Redis 동기화 임계영역을 보호.
- 주문 생성 트랜잭션 커밋 이후에만 메시지 발행(트랜잭션 동기화 / Outbox 패턴).
- 트랜잭션 커밋 이후 RabbitMQ 이벤트 발행 (Outbox 패턴 권장)  
- 라이언트는 SSE 또는 WebSocket으로 서버 이벤트를 구독 → 신규 주문 실시간 반영.
- polling은 fallback으로 축소(주기↑, 부하↓) 또는 단계적 제거.

---

## 🏗️ 구조 요약
[O2O Partner] → [주문 API]
↓
(분산락 획득)
↓
DB Upsert (멱등)
↓
Redis 동기화
↓
(커밋 이후) MQ 발행
↓
Client(SSE/WS) 반영
