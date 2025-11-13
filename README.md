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
O2O Partner → 주문 API  
↓  
(분산락 획득)  
↓  
DB Upsert (멱등)  
↓  
Redis 동기화  
↓  
(커밋 이후) MQ 발행  
↓  
Client (SSE / WS) 반영

---

## 📦 Maven 의존성
```xml
<dependencies>
  <!-- Redis & Redisson -->
  <dependency>
    <groupId>org.redisson</groupId>
    <artifactId>redisson-spring-boot-starter</artifactId>
    <version>3.36.0</version>
  </dependency>
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
  </dependency>
```

---

## 🔧 RedisConfig.java
```java
import org.redisson.Redisson;
import org.redisson.api.RedissonClient;
import org.redisson.config.Config;
import org.springframework.boot.autoconfigure.data.redis.RedisProperties;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class RedisConfig {
    private final RedisProperties redisProperties;

    public RedisConfig(RedisProperties redisProperties) {
        this.redisProperties = redisProperties;
    }
    
    @Bean
    public RedissonClient redissonClient() {
        Config config = new Config();

        config.useSingleServer()
                .setAddress(redisProperties.getUrl())
                .setDatabase(redisProperties.getDatabase());

        if (redisProperties.getPassword() != null) {
            config.useSingleServer().setPassword(redisProperties.getPassword());
        }

        return Redisson.create(config);
    }
}
```

---

## 🔧 RedisService.java
```java
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.redisson.api.RLock;
import org.redisson.api.RedissonClient;
import org.springframework.stereotype.Service;

import java.util.concurrent.TimeUnit;
import java.util.function.Supplier;

@Slf4j
@Service
@RequiredArgsConstructor
public class RedisService {
    private static final String ACTION_KEY_PREFIX = "action-lock-";
    
    private final RedissonClient redissonClient;

    // Redis 기반 분산 락으로, 특정 메소드 실행 단위에 대해 동시 실행을 제어하기 위한 락 처리 메소드
    public <T> T executeWithLock(String action, String shopCd, Supplier<T> supplier) {
        String key = ACTION_KEY_PREFIX + action + shopCd;
        RLock lock = redissonClient.getLock(key);
        boolean acquired = false;

        try {
            acquired = lock.tryLock(3, 10, TimeUnit.SECONDS); // 최대 3초간 락 획득을 시도하고, 성공 시 10초 동안 락을 유지

            if (!acquired) {
                log.error("RedisService::executeWithLock - info : {}", "진행 중인 서비스 입니다.");
                return supplier.get();
            }

            return supplier.get(); // ---- 임계영역 ----

        } catch (Exception e) {
            log.error("RedisService::executeWithLock - error : {}", e.getMessage(), e);
            return supplier.get();
        } finally {
            // 락을 성공적으로 획득했고, 현재 스레드가 해당 락을 소유 중인 경우에만 해제
            if (acquired && lock.isHeldByCurrentThread()) {
                lock.unlock();
            }
        }
    }
}
```
---

## 🎯 기대효과

- **중복 주문 방지**  
  - 주문 생성과 Redis 동기화 시점을 일원화하여 동일 주문이 2회 이상 생성되는 문제 해소
- **데이터 일관성 확보**  
  - DB ↔ Redis 간 불일치 최소화로 안정적인 캐시 및 상태 관리 가능
- **트랜잭션 안정성 강화**  
  - 커밋 이후 이벤트 발행을 통해 메시지 중복 및 유실 방지
- **실시간 반영성 향상**  
  - SSE 기반으로 클라이언트에 신규 주문 즉시 전달
- **시스템 부하 감소**  
  - 기존 4초 단위 polling 의존도를 낮추어 네트워크 및 DB 부하 감소
