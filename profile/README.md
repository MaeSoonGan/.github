# [우리FISA 6기] 클라우드 엔지니어링 1팀 — 매순간(MaeSoonGan)

> **장애 상황에서도 멈추지 않는** 고가용성 모의투자 거래 플랫폼
>
> EKS 오토스케일링 · 인프라 이중화 · DR 서버로 무중단 운영을 지향하며, 증권사 표준 **채널계 / 계정계 / 체결계 분리**를 <br>**AWS 클라우드 ↔ 온프레미스 하이브리드**로 구현

---

## 1. 프로젝트 개요

- **주제** : 한국투자증권 Open API의 실시간 시세로 가상 자산을 매매하고 모의투자 대회로 수익률을 겨루는 모의투자 거래 플랫폼입니다. 실제 증권사의 **채널계 / 계정계 / 체결계 분리 구조**를 클라우드 ↔ 온프레미스 하이브리드로 구현하고, **장애 상황에서도 멈추지 않는 고가용성** 확보를 최우선 목표로 설계했습니다.

- **프로젝트 기획 배경** : 증권 서비스는 장 운영 중 단 몇 분의 장애도 큰 손실로 이어지므로, **"장애가 발생해도 멈추지 않는"** 고가용성이 핵심 과제입니다. 이를 위해 다음과 같이 설계했습니다.
  - **오토스케일링** — 채널계(AWS EKS)는 장 개장·마감 트래픽 급증에 EKS 오토스케일링으로 탄력 대응
  - **이중화** — AZ 이중화, RDS Primary/Replica, Redis Multi-AZ, NAT/로드밸런서 이중화로 단일 장애점 제거
  - **DR** — 계정계(온프레미스)는 DC1 Active / DC2 Standby + Keepalived VIP + DB 복제로 데이터센터 장애 시에도 서비스 연속성 유지

- **기술 스택** :

| 영역                    | 기술                                                                                                              |
| ----------------------- | ----------------------------------------------------------------------------------------------------------------- |
| **Backend**       | Java 17, Spring Boot (Gradle 멀티모듈 모노레포), Spring Security + JWT                                            |
| **Data**          | MySQL / MariaDB (온프레 Master·Slave, AWS RDS Replica), Redis (ElastiCache, 온프레 Redis Sentinel), Apache Kafka |
| **External**      | 한국투자증권 Open API, WebSocket                                                                                  |
| **Cloud (AWS)**   | EKS, RDS, ElastiCache, ECR, ALB/NLB, WAF, Route 53, S3                                                            |
| **On-Premise**    | VMware ESXi / vCenter, Rocky Linux 9, Docker Compose, Keepalived, pfSense (Site-to-Site VPN 종단)                 |
| **IaC / CI·CD**  | Terraform, GitHub Actions, GitLab CE + ArgoCD + Argo Rollouts                                                     |
| **Observability** | Prometheus, Grafana, Loki, Jaeger, Grafana Alloy, OpenTelemetry                                                   |

---

## 2. 아키텍처

### 2-1. 시스템 아키텍처

<p align="center">
  <img width="2850" height="1780" alt="최종_전체_아키텍처" src="https://github.com/user-attachments/assets/89a51d35-79e8-4cfe-9f3a-c0aabf563187" />
</p>

### 설명

매순간은 AWS 클라우드와 온프레미스를 **Site-to-Site VPN**으로 연결한 하이브리드 구조입니다.

- **채널계 (AWS EKS)**: 회원·시세·주문·거래·자산·알림·실시간 시세·사용자 실시간 이벤트를 서비스별 Pod로 배치. 외부 트래픽은 Route 53 → NLB → WAF → ALB로 유입됩니다.
- **계정계·체결 (온프레미스)**: 원장 DB와 체결 엔진을 배치하고, DC1/DC2 이중화·DB 복제로 장애 복구 체계를 구성했습니다.
- **연동**: 비동기 연동(주문·체결)은 VPN 경유 Kafka로, 일부 동기 요청(회원)은 HTTP API로 처리합니다.
- **모니터링**: EKS 지표를 Grafana Alloy가 수집해 온프레미스 Prometheus로 `remote_write`하고 Grafana에서 통합 조회합니다.

### 2-2. 소프트웨어 아키텍처

<p align="center">
  <img src="docs/software-architecture.png" alt="소프트웨어 아키텍처" width="900">
  <br><em>※ 소프트웨어 아키텍처 다이어그램 — 이미지 삽입 예정</em>
</p>

### 설명

증권사의 채널계·계정계·체결계 분리를 참고해 **읽기/쓰기 책임을 분리(CQRS)** 했습니다.

- **계정계(원장)** 가 자산의 최종 정합성(잔고·보유·체결)을 관리하고, **채널계**는 Kafka로 동기화된 **스냅샷**(주문·포트폴리오)으로 조회를 처리합니다.
- **체결계**는 Order Book 기반으로 주문을 매칭해 결과를 Kafka로 발행합니다.
- → 조회가 집중되는 채널계는 **수평 확장**, 정합성이 중요한 원장은 온프레미스에서 **일관성 있게 운영**합니다. (서비스별 책임은 아래 MSA 표 참고)

#### 마이크로서비스 구성 (MSA)

채널계와 계정계 모두 **도메인별 독립 마이크로서비스**로 분해했습니다. Gradle 멀티모듈 모노레포로 관리하되 배포는 **서비스 단위 컨테이너**로 이루어져, 서비스별 독립 배포·확장·장애 격리가 가능합니다. EKS에서는 워크로드 성격별 노드그룹으로 격리 배치하여, 트래픽이 몰리는 서비스(예: 실시간 시세)만 선택적으로 스케일아웃합니다.

**채널계 (AWS EKS)**

| 서비스                      | 책임                                                   |
| --------------------------- | ------------------------------------------------------ |
| `auth-service`            | 회원가입·로그인·JWT 발급/재발급·이메일 인증         |
| `admin-service`           | 회원·대회·공지·시스템 관리, 대회 랭킹 스케줄러      |
| `contest-service`         | 모의투자 대회 조회·참가·탈퇴·랭킹                   |
| `market-service`          | 종목·차트·지수·관심종목, KIS 시세 스냅샷            |
| `order-service`           | 주문 접수·취소, 포트폴리오, 증거금(주문가능금액) 예약 |
| `notification-api`        | 알림·공지 발행/조회, 시세 알림 스케줄러               |
| `market-realtime-service` | KIS 실시간 시세 수집 → Redis 캐싱 → WebSocket 송출   |
| `user-realtime-service`   | 사용자별 실시간 이벤트 푸시(SSE)                       |
| `trade-sync-worker`       | Kafka 이벤트 소비 → 채널 DB 동기화(Read-Model)        |

**계정계 (On-Premises)**

| 서비스               | 책임                                                               |
| -------------------- | ------------------------------------------------------------------ |
| `execution-engine` | 주문 수신·상태 관리, 시장가/지정가 체결(매칭), 부분체결, 자동취소 |
| `ledger-service`   | 계좌 원장·잔고·보유종목·거래내역·손익(PnL), EOD 마감·대사     |
| `member-service`   | 회원 자격증명(비밀번호) 관리, 보안 명령 처리                       |

**MSA 채택 이유**

- **독립 배포** — 서비스별로 배포(GitOps)하여 장애 시 영향 범위 최소화
- **독립 확장** — 실시간 시세·주문 등 부하 큰 서비스만 선택적으로 스케일
- **장애 격리** — 한 서비스 장애가 전체로 전파되지 않음
- **느슨한 결합** — Kafka 이벤트 기반 비동기 연동으로 서비스 간 의존도 최소화

---

## 3. 주요 기능 소개

### 3-1. 핵심 기술 구성

<p align="center">
  <img src="docs/core-features-pentagon.png" alt="핵심 기술 구성" width="800">
  <br><em>※ 핵심 기술 구성(펜타곤) 다이어그램 — 이미지 삽입 예정</em>
</p>

매순간의 차별화된 핵심 기술 5가지입니다.

1. **채널계 / 계정계 / 체결계 분리 하이브리드** — 증권사 표준 3계 분리를 AWS(채널계) ↔ 온프레미스(계정계·체결) 하이브리드로 구현하여, 정합성이 중요한 원장과 확장성이 중요한 채널을 분리 운영합니다.
2. **이중화 + DR 기반 고가용성** — AZ 이중화·RDS/Redis 복제·DC1/DC2 Standby(Keepalived VIP)로 단일 장애점을 제거하고, 장애 상황에서도 무중단 운영을 유지합니다.
3. **Kafka 이벤트 기반 비동기 연동** — 주문→체결→정산을 Kafka로 비동기 연결하고, At-least-once + 멱등 처리로 결합도를 낮추면서 무손실·중복 방지를 보장합니다.
4. **CQRS 스냅샷 조회 모델** — 자산의 최종 정합성은 온프레미스 원장(쓰기)이, 조회는 Kafka로 동기화된 채널계 스냅샷(읽기)이 담당하여 조회 부하를 원장에서 격리합니다.
5. **통합 관측성(Observability)** — Metric(Prometheus)·Log(Loki)·Trace(Jaeger)를 Grafana Alloy로 클라우드↔온프레미스 통합 수집하여 장애를 조기 탐지합니다.

### 3-2. 통합 워크플로우 다이어그램

<p align="center">
  <img src="docs/workflow-diagram.png" alt="통합 워크플로우" width="900">
  <br><em>※ 통합 워크플로우 다이어그램 — 이미지 삽입 예정</em>
</p>

### 3-3. 세부 기능 소개

> 매순간은 **Redis · Kafka 미들웨어로 증권 거래의 정합성·신뢰성 문제를 해결**하는 데 중점을 두었습니다. 아래 5개 세부 기능은 그 핵심 구현입니다.

---

#### [기능 1] Kafka Consumer 멱등 처리 — 체결 이벤트 중복 반영 방지

**기능 설명**
At-least-once 메시징에서는 지연·재시작·재시도로 동일 이벤트가 다시 전달될 수 있고, 중복 반영되면 잔고·포트폴리오 정합성이 깨집니다. 이를 막기 위해 체결·동기화 이벤트의 `eventId`를 처리 이력에 기록하고, 처리 여부 확인·동기화 작업·이력 기록을 **하나의 트랜잭션**으로 묶어 재전송에도 중복 반영되지 않도록 했습니다.

**핵심 코드(스크립트)**

```java
private SyncResult processEvent(String eventId, String eventType, String aggregateType,
                                String aggregateId, Runnable action) {
    if ("SUCCESS".equals(findProcessStatus(eventId))) {                       // 멱등: 이미 처리 시 SKIP
        return new SyncResult(eventId, eventType, aggregateType, aggregateId,
                "SKIPPED", "Already processed event", LocalDateTime.now());
    }
    try {
        transactionTemplate.executeWithoutResult(status -> {
            action.run();                                                     // 스냅샷 동기화
            upsertSyncEventLog(eventId, eventType, aggregateType, aggregateId, // 처리 이력 기록
                    "SUCCESS", null, LocalDateTime.now());
        });
        return new SyncResult(eventId, eventType, aggregateType, aggregateId, "SUCCESS", "Sync completed", ...);
    } catch (RuntimeException exception) {
        upsertSyncEventLog(eventId, eventType, aggregateType, aggregateId, "FAILED", truncate(exception.getMessage()), ...);
        throw exception;
    }
}
```

**코드 링크(스크립트 링크)** : [TradeSyncService.java](https://github.com/MaeSoonGan/Back-End/blob/develop/apps/worker/trade-sync-worker/src/main/java/com/mock/maesoongan/tradesyncworker/sync/TradeSyncService.java#L594)

---

#### [기능 2] Kafka Producer 전송 보장 — 주문 이벤트 무손실 발행

**기능 설명**
발행이 유실되면 "채널계는 접수, 계정계는 모름"의 **채널-원장 불일치**가 생기고, 주문 1건도 유실되면 안 됩니다. 따라서 fire-and-forget 대신 **브로커 ack를 동기적으로 대기**(`get(timeout)`)하고, 실패하면 예외를 던져 상위 보상 흐름(증거금 예약 해제)을 트리거합니다.

**핵심 코드(스크립트)**

```java
private void send(String topic, String key, Object event) {
    if (!kafkaEnabled) { log.warn("Kafka disabled. Skip publish. topic={}, key={}", topic, key); return; }
    try {
        kafkaTemplate.send(topic, key, event)
                .get(sendTimeout.toMillis(), TimeUnit.MILLISECONDS);          // 브로커 ack 동기 대기
    } catch (Exception exception) {
        throw new BusinessException(HttpStatus.INTERNAL_SERVER_ERROR,
                "KAFKA_PUBLISH_FAILED", "Failed to publish order event.");    // 실패 → 보상 트리거
    }
}
```

**코드 링크(스크립트 링크)** : [OrderEventPublisher.java](https://github.com/MaeSoonGan/Back-End/blob/develop/apps/api/order-service/src/main/java/com/mock/maesoongan/orderservice/order/OrderEventPublisher.java)

---

#### [기능 3] 채널계 ↔ 계정계 Kafka 비동기 연동

**기능 설명**
동기 호출로 결합하면 체결계 지연·장애가 사용자 응답 지연으로 직결됩니다. 이를 피하기 위해 채널계 `order-service`가 **주문 ID를 파티션 Key**로 토픽에 발행하면, VPN을 경유해 계정계 `execution-engine`이 소비하고 체결 결과를 다시 이벤트로 회신합니다. Kafka 비동기 연동으로 결합도를 낮춰 체결계가 지연돼도 주문 접수는 유지되며, 주문 ID Key로 동일 주문의 처리 순서를 보장합니다.

**핵심 코드(스크립트)**

```java
// 채널계: 토픽 계약(KafkaTopics) — 채널↔계정 연동 인터페이스
public static final String ORDER_REQUEST = "order.request";
public static final String ORDER_CANCEL  = "order.cancel";
public static final String EXECUTION_RESULT    = "execution.result";
public static final String EXECUTION_CONFIRMED = "execution.confirmed";

// 채널계: 주문 ID를 Key로 발행 → 동일 주문 순서 보장
public void publishOrderRequested(OrderRequestedEvent event) {
    send(orderRequestTopic, String.valueOf(event.orderId()), event);
}

// 계정계(온프레): VPN 경유 Kafka 소비 → 체결 처리
@KafkaListener(topics = KafkaTopics.ORDER_REQUEST, groupId = "${spring.kafka.consumer.group-id}")
public void listenOrderRequest(@Payload OrderRequestEvent event) {
    orderEventProcessingService.processOrderRequest(event);
}
```

**코드 링크(스크립트 링크)** : [KafkaTopics.java](https://github.com/MaeSoonGan/Onpre-Back-End/blob/main/libs/event-contracts/src/main/java/com/maesungan/onprem/event/KafkaTopics.java) · [OrderEventPublisher.java](https://github.com/MaeSoonGan/Back-End/blob/develop/apps/api/order-service/src/main/java/com/mock/maesoongan/orderservice/order/OrderEventPublisher.java) · [OrderEventListener.java](https://github.com/MaeSoonGan/Onpre-Back-End/blob/main/apps/execution-engine/src/main/java/com/maesungan/onprem/execution/order/messaging/OrderEventListener.java)

---

#### [기능 4] Redis Lua Script 기반 원자적 증거금(주문가능금액) 선점

**기능 설명**
`GET`으로 확인 후 `DECR`로 차감하면 두 요청이 동시에 "잔액 충분"으로 판단하는 **경쟁 조건(race condition)** 이 생깁니다. 이를 막기 위해 동시 매수 주문에서도 **잔액 조회·비교·차감을 하나의 Redis Lua Script로 원자 실행**하여, 원장 처리 이전 단계인 채널계에서 증거금을 선점하고 가용 금액을 초과한 과주문을 차단합니다.

**핵심 코드(스크립트)**

```java
private static final String RESERVE_SCRIPT = """
        local balance = redis.call('GET', KEYS[1])
        if not balance then return -1 end
        local pending = redis.call('GET', KEYS[2])
        local amount = tonumber(ARGV[1])
        if tonumber(balance) + tonumber(pending or '0') < amount then return 0 end
        redis.call('INCRBYFLOAT', KEYS[1], '-' .. ARGV[1])      -- 원자적 차감
        return 1
        """;

public ReserveResult reserve(long memberId, long contestId, BigDecimal amount) {
    Long result = redisTemplate.execute(reserveScript,
            List.of(balanceKey(memberId, contestId), pendingReleaseKey(memberId, contestId)),
            amount.toPlainString());
    if (result == null || result < 0) return ReserveResult.MISSING;
    if (result == 0)                  return ReserveResult.INSUFFICIENT;
    return ReserveResult.RESERVED;
}
```

**코드 링크(스크립트 링크)** : [BalanceCache.java](https://github.com/MaeSoonGan/Back-End/blob/develop/apps/api/order-service/src/main/java/com/mock/maesoongan/orderservice/order/BalanceCache.java)

---

#### [기능 5] Redis TTL 기반 주문 취소 예약 보상

**기능 설명**
취소는 "요청 → Kafka → 계정계 확정"의 비동기 흐름이라 그 사이에 **시간 갭**이 있고, 미리 복구하면 미확정 금액이 재주문에 쓰여 정합성이 깨집니다. 따라서 취소 요청 직후 잔고를 바로 복구하지 않고 **반환 예정 금액을 TTL과 함께 Redis에 기록**해 보류했다가, 계정계에서 취소가 확정될 때 Lua Script로 복원·pending 차감·키 정리를 원자 처리합니다.

**핵심 코드(스크립트)**

```java
// ① 취소 요청 시: 반환 예정 금액을 TTL과 함께 pending으로 기록 (즉시 복구 X)
public void markCancelPending(long memberId, long contestId, long orderId, BigDecimal pendingAmount, Duration ttl) {
    redisTemplate.opsForValue().set("cancel:pending:" + orderId, "true", ttl);
    if (pendingAmount.compareTo(BigDecimal.ZERO) > 0) {
        String key = pendingReleaseKey(memberId, contestId);
        redisTemplate.opsForValue().increment(key, pendingAmount.doubleValue());
        redisTemplate.expire(key, ttl);
    }
}
```

```lua
-- ② 취소 확정 시: 예약 복원 · pending 차감 · 키 정리를 원자적으로 처리
local balance = redis.call('GET', KEYS[1])
if tonumber(ARGV[1]) > 0 and balance then
    redis.call('INCRBYFLOAT', KEYS[1], ARGV[1])                 -- 잔고 복원
end
if tonumber(ARGV[2]) > 0 then
    local pending = redis.call('GET', KEYS[2])
    if pending then
        local nextPending = tonumber(pending) - tonumber(ARGV[2])
        if nextPending > 0 then redis.call('SET', KEYS[2], tostring(nextPending))
        else redis.call('DEL', KEYS[2]) end
    end
end
redis.call('DEL', KEYS[3])
return 1
```

**코드 링크(스크립트 링크)** : [BalanceCache.java](https://github.com/MaeSoonGan/Back-End/blob/develop/apps/api/order-service/src/main/java/com/mock/maesoongan/orderservice/order/BalanceCache.java) · [TradeSyncService.java](https://github.com/MaeSoonGan/Back-End/blob/develop/apps/worker/trade-sync-worker/src/main/java/com/mock/maesoongan/tradesyncworker/sync/TradeSyncService.java)

---
