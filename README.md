# [우리FISA 6기] 클라우드 엔지니어링 과정 1팀 — 매순간(MaeSoonGan)

## 1. 프로젝트 개요

* **주제**: 실제 증권사 시스템 아키텍처를 모방한 모의투자 거래 플랫폼 **매순간**
* **프로젝트 기획 배경**: 매순간은 실제 증권사가 채택하는 **채널계(고객 접점) / 계정계(원장) / 체결(매칭 엔진)** 의 분리 구조를 재현하고 이를 **AWS 클라우드(채널계) ↔ 온프레미스(계정계·체결) 하이브리드** 환경에 배치하는 것을 목표로 기획되었습니다.
* **기술 스택**
    * **Backend**: Java 17, Spring Boot 3.x (Gradle 멀티모듈 모노레포), Spring Security + JWT
    * **Data**: MySQL / MariaDB (온프레 Master·Slave, AWS RDS Aurora Replica), Redis (ElastiCache, 온프레 Redis Sentinel), Apache Kafka
    * **External**: 한국투자증권 Open API, WebSocket
    * **Cloud (AWS)**: EKS, RDS, ElastiCache, ECR, ALB/NLB, WAF, Route 53, S3
    * **On-Premise**: VMware ESXi / vCenter, Rocky Linux 9, Docker Compose
    * **IaC / CI·CD**: Terraform, GitHub Actions, GitLab CE + ArgoCD + Argo Rollouts
    * **Observability**: Prometheus, Grafana
---

## 2. 아키텍처

### 2-1. 시스템 아키텍처

<!-- 여기에 시스템 아키텍처 다이어그램 이미지를 첨부하세요 (preview 이미지 활용) -->
<!-- <img width="862" height="690" alt="시스템 아키텍처" src="이미지_URL" /> -->

### 설명

매순간은 **AWS 클라우드와 온프레미스를 Site-to-Site VPN으로 연결한 하이브리드 구조**로 설계되었습니다.

* **채널계 (AWS EKS)**: 회원, 시세, 주문, 거래, 자산, 알림, 실시간 시세, 사용자 실시간 이벤트 등을 서비스별 Pod로 배치했습니다. 외부 트래픽은 Route 53 → ALB로 유입됩니다.
* **계정계·체결 (온프레미스)**: 원장 DB와 주문 체결 엔진을 온프레미스에 배치했습니다. DC1 Active/DC2 Standby와 Keepalived VIP, MySQL Primary/Replica 복제로 가용성과 장애 복구 체계를 구성했습니다. 
* **연동**:  주문, 취소 요청과 체결 결과 등 비동기 연동은 VPN을 경유한 온프레미스 Kafka로 처리합니다. 회원 계정계 연동 등 일부 동기 요청은 HTTP API로 처리합니다. 
* **모니터링·알림**: EKS의 Grafana Alloy, Node Exporter, kube-state-metrics가 수집한 지표를 온프레미스 Prometheus로 remote_write하고 Grafana에서 통합 조회합니다.
### 2-2. 소프트웨어 아키텍처

<!-- 여기에 소프트웨어/도메인 아키텍처 다이어그램 이미지를 첨부하세요 -->
<!-- <img width="1148" height="705" alt="소프트웨어 아키텍처" src="이미지_URL" /> -->

### 설명

증권사 시스템의 채널계, 계정계, 체결계 분리 방식을 참고해 각 영역의 책임을 구분했습니다.

* **채널계**는 고객 요청을 검증하고 주문, 취소 요청은 채널 DB의 주문 스냅샷에 접수 상태를 기록한 뒤 Kafka로 발행합니다. 회원 관련 일부 명령은 온프레미스 계정계 HTTP API를 호출합니다. 조회는 RDS의 회원, 주문, 포트폴리오 스냅샷과 Redis 등 채널계 조회 모델을 사용합니다.
* **체결계**는 Order Book을 기반으로 주문을 매칭합니다. 체결 결과와 갱신된 잔고, 보유종목 정보를 Kafka 이벤트로 발행하며 채널계는 이를 소비해 조회용 스냅샷을 갱신합니다.
* **계정계(원장)** 는 잔고, 보유종목, 체결 내역의 최종 자산 변경을 담당합니다. 채널계 스냅샷은 원장의 복제본이므로 이벤트 처리 시점에 따라 일시적인 지연이 발생할 수 있습니다.
이 구조를 통해 조회 트래픽이 집중되는 채널계와 정합성이 중요한 원장을 분리했습니다. 채널 서비스는 독립적으로 확장할 수 있도록 구성했으며 원장은 온프레미스에서 일관된 자산 변경을 담당합니다. 
따라서 트래픽이 몰리는 채널계는 자유롭게 수평 확장하면서도 정합성이 핵심인 원장은 온프레미스에서 안정적으로 운영할 수 있습니다.

---

## 3. 주요 기능 소개

### 3-1. 핵심 기술 구성

<!-- 여기에 핵심 기술 구성 이미지를 첨부하세요 -->


매순간은 실제 금융권 시스템에서 중요하게 다루는 **확장성, 정합성, 장애 대응, 실시간성**을 프로젝트 구조에 반영하는 것을 목표로 했습니다. 특히 채널계와 원장을 분리하고 주문 체결 흐름을 이벤트 기반으로 처리하며 조회 부하는 스냅샷 모델로 분산하는 데 중점을 두었습니다.

#### 01. 채널계와 원장을 분리한 하이브리드 구조

트래픽 변동이 큰 채널계는 **AWS EKS**에 배치하고, 정합성이 중요한 계정계와 체결계는 **온프레미스**에 구성했습니다. 이를 통해 사용자의 요청을 처리하는 영역은 유연하게 확장하면서도, 자산 변경을 담당하는 원장은 통제 가능한 환경에서 안정적으로 운영할 수 있도록 설계했습니다.

#### 02. Kafka 기반 주문·체결 비동기 처리

주문 접수와 체결 처리를 직접 호출 방식으로 연결하지 않고, **Kafka 이벤트**를 통해 비동기로 전달했습니다. 채널계는 주문을 접수한 뒤 이벤트를 발행하고, 체결계는 이를 소비해 처리 결과를 다시 이벤트로 전달합니다. 이 구조를 통해 체결계 지연이나 장애가 사용자 응답 흐름에 직접 영향을 주지 않도록 했습니다.

#### 03. CQRS 기반 스냅샷 조회 모델

자산 변경의 기준 데이터는 원장이 관리하고, 채널계는 Kafka로 동기화된 **회원·주문·포트폴리오 스냅샷**을 조회합니다. 원장 DB를 직접 조회하지 않도록 읽기와 쓰기 책임을 분리하여, 조회 트래픽이 원장 부하로 이어지지 않도록 설계했습니다.

#### 04. 실시간 시세 및 사용자 이벤트 스트리밍

한국투자증권 Open API에서 수신한 실시간 시세를 **Redis에 캐싱**하고, 사용자 화면에는 **WebSocket과 SSE**를 통해 전달했습니다. 또한 사용자가 실제로 조회하는 종목만 동적으로 구독하도록 구성해 불필요한 외부 API 호출과 네트워크 사용을 줄였습니다.

#### 05. 고가용성 및 통합 관측 환경 구성

온프레미스는 **Active/Standby 구조와 DB 복제**를 통해 장애 대응이 가능하도록 구성하고, AWS EKS 기반 서비스는 MSA 구조로 배치했습니다. 또한 **Alloy, Prometheus, Grafana, OpenTelemetry**를 활용해 클라우드와 온프레미스 환경의 지표를 통합적으로 관측할 수 있도록 했습니다.


### 3-2. 통합 워크플로우 다이어그램

<!-- 여기에 주문→체결→알림 통합 워크플로우 이미지를 첨부하세요 -->

**주문 1건의 전체 흐름**

```
[사용자] 
   │ ① 매수/매도 주문
   ▼
[주문 API POD (AWS EKS)]
   │ ② JWT 인증 → Redis 잔고 예약 → order_snapshot(PENDING) 저장
   │ ③ 사용자에게 '접수됨' 즉시 응답
   │ ④ Kafka 'order.requested' 발행 (VPN 경유)
   ▼
[체결 엔진 (온프레미스)]
   │ ⑤ Order Book 매칭 → 체결
   │ ⑥ 원장 DB(MySQL Master) FOR UPDATE 최종 잔고 차감
   │ ⑦ Kafka 'order.completed' / 'execution.confirmed' 발행
   ▼
[채널계 멀티 Consumer]
   ├─ notification-group → 알림 발송 (Lambda → SNS/SES)
   ├─ ranking-group     → 랭킹 점수 집계 (Redis Sorted Set)
   └─ websocket-group   → 사용자 화면 실시간 갱신
```

### 3-3. 세부 기능 소개

#### [기능 1] Redis Lua Script를 이용한 주문 가능 금액 원자적 선점

* **기능 설명**: 동일 계좌에서 여러 매수 주문이 동시에 들어와도 잔액 검증과 차감을 하나의 **Redis Lua Script**로 실행합니다. Redis가 Script 전체를 원자적으로 처리하므로, 동시에 여러 주문이 들어오는 상황에서도 가용 금액을 초과한 중복 주문을 방지할 수 있습니다.
* **설계 의도**: 단순히 `GET`으로 잔액을 확인한 뒤 `DECR`로 차감하면 두 요청이 동시에 들어왔을 때 둘 다 잔액이 충분하다고 판단하는 경쟁 조건이 발생할 수 있습니다. 이를 방지하기 위해 잔액 조회, 주문 금액 비교, 차감 처리를 하나의 Lua Script 안에서 수행하도록 설계했습니다. 이를 통해 채널계에서 빠르게 주문 가능 금액을 선점하고, 원장 처리 이전 단계에서 과주문을 차단할 수 있습니다.
* **핵심 코드**:

```java
private static final String RESERVE_SCRIPT = """
    local balance = redis.call('GET', KEYS[1])
    if not balance then return -1 end

    local amount = tonumber(ARGV[1])
    if tonumber(balance) < amount then return 0 end

    redis.call('INCRBYFLOAT', KEYS[1], '-' .. ARGV[1])
    return 1
    """;

public ReserveResult reserve(long memberId, long contestId, BigDecimal amount) {
    Long result = redisTemplate.execute(
        reserveScript,
        List.of(balanceKey(memberId, contestId),
                pendingReleaseKey(memberId, contestId)),
        amount.toPlainString()
    );

    return result == 1 ? RESERVED : INSUFFICIENT;
}
```

* **코드 링크**: `BalanceCache.java`

#### [기능 2] Kafka 기반 비동기 주문·체결 파이프라인

* **기능 설명**: 채널계는 주문 요청을 검증하고 접수 상태를 저장한 뒤, 주문 ID를 Key로 Kafka 이벤트를 발행합니다. 온프레미스 체결계는 해당 이벤트를 소비해 주문을 처리하고, 체결 결과를 다시 이벤트로 전달합니다. 이를 통해 주문 접수와 체결 처리를 비동기 구조로 분리했습니다.
* **설계 의도**: 주문 접수와 체결을 동기 방식으로 강하게 결합하면 체결계 지연이나 장애가 사용자 응답 지연으로 이어질 수 있습니다. Kafka를 중심으로 주문 요청과 체결 결과를 이벤트로 전달하면 시스템 간 결합도를 낮추고, 체결계가 일시적으로 지연되더라도 주문 접수 흐름을 안정적으로 유지할 수 있습니다. 또한 주문 ID를 Key로 사용해 동일 주문에 대한 이벤트 처리 순서를 관리하기 쉽도록 구성했습니다.
* **핵심 코드**:

```java
orderRepository.insertOrder(command);

orderEventPublisher.publishOrderRequested(
    new OrderRequestedEvent(
        orderId,
        accountId(memberId),
        stockCode,
        side,
        orderType,
        orderPrice,
        quantity,
        now
    )
);

private void send(String topic, String key, Object event) {
    kafkaTemplate.send(topic, key, event)
        .get(sendTimeout.toMillis(), TimeUnit.MILLISECONDS);
}
```

* **코드 링크**: `OrderService.java`, `OrderEventPublisher.java`

#### [기능 3] 이벤트 멱등성 및 트랜잭션 기반 스냅샷 동기화

* **기능 설명**: 체결 이벤트의 `eventId`를 처리 이력에 기록해 이미 성공한 이벤트가 다시 들어와도 중복 처리되지 않도록 했습니다. 체결 내역 저장, 주문 상태 변경, 이벤트 처리 이력 기록을 하나의 DB 트랜잭션에서 수행하여 재전송에 강한 동기화 구조를 구현했습니다.
* **설계 의도**: Kafka 기반 이벤트 구조에서는 네트워크 지연, Consumer 재시작, 재시도 등의 이유로 동일 이벤트가 다시 전달될 수 있습니다. 이때 체결 내역이나 주문 상태가 중복 반영되면 잔고와 포트폴리오 정합성이 깨질 수 있습니다. 따라서 `eventId` 기준으로 처리 성공 여부를 먼저 확인하고, 실제 동기화 작업과 처리 이력 저장을 하나의 트랜잭션으로 묶어 중복 반영을 방지했습니다.
* **핵심 코드**:

```java
private SyncResult processEvent(String eventId, Runnable action) {
    if ("SUCCESS".equals(findProcessStatus(eventId))) {
        return skipped(eventId, "Already processed event");
    }

    transactionTemplate.executeWithoutResult(status -> {
        action.run();
        upsertSyncEventLog(eventId, "SUCCESS");
    });

    return success(eventId);
}

processEvent(eventId, () -> {
    upsertTradeHistory(request);
    updateOrderByTrade(request);
});
```

* **코드 링크**: `TradeSyncService.java`

#### [기능 4] 원장과 조회 모델을 분리한 CQRS 스냅샷 구조

* **기능 설명**: 잔고와 보유종목의 권위 있는 데이터는 온프레미스 원장이 관리하고, 채널계는 원장 DB를 직접 조회하지 않습니다. 대신 Kafka로 동기화된 `portfolio_snapshot`, `order_snapshot`, `trade_history`를 이용해 포트폴리오와 주문 내역을 빠르게 조회합니다.
* **설계 의도**: 원장 DB는 체결, 잔고 차감, 보유종목 변경처럼 정합성이 중요한 쓰기 작업에 집중해야 합니다. 반면 포트폴리오 조회, 주문 내역 조회, 체결 내역 조회는 사용자 요청이 빈번하게 발생하는 읽기 작업입니다. 읽기와 쓰기 책임을 분리하면 원장 부하를 줄이고, 채널계는 스냅샷 테이블을 통해 빠르게 응답할 수 있어 전체 시스템의 안정성과 확장성을 높일 수 있습니다.
* **핵심 코드**:

```java
public Optional<PortfolioRow> findPortfolio(
        long memberId,
        long contestId
) {
    return queryOne("""
        select cash_balance, available_cash,
               stock_evaluation_amount, total_asset,
               holdings_json, portfolio_version, synced_at
        from portfolio_snapshot
        where member_id = ? and contest_id = ?
        """, portfolioRowMapper, memberId, contestId);
}
```

```sql
insert into portfolio_snapshot (...)
values (...)
on duplicate key update
    cash_balance = values(cash_balance),
    holdings_json = values(holdings_json),
    portfolio_version = values(portfolio_version)
```

* **코드 링크**: `PortfolioRepository.java`, `TradeSyncService.java`

#### [기능 5] 구독 수 기반 실시간 시세 스트리밍 및 자동 복구

* **기능 설명**: 사용자가 요청한 종목만 한국투자증권 WebSocket에 동적으로 구독합니다. 종목별 구독자 수를 관리해 최초 구독자 발생 시에만 증권사에 등록하고, 마지막 구독자가 종료되면 구독을 해제합니다. 또한 연결 장애 후 복구되면 현재 활성화된 구독 종목을 자동으로 재등록합니다.
* **설계 의도**: 모든 종목을 상시 구독하면 불필요한 네트워크 사용량과 외부 API 부담이 커집니다. 따라서 실제 사용자가 보고 있는 종목만 구독하도록 구성해 리소스를 효율적으로 사용했습니다. 또한 WebSocket 연결은 네트워크 상황에 따라 끊길 수 있으므로, 복구 시 기존 활성 구독 목록을 다시 등록해 사용자가 시세 중단을 최소한으로 체감하도록 설계했습니다.
* **핵심 코드**:

```java
private boolean incrementGlobalSubscription(
        String stockCode,
        ConcurrentMap<String, AtomicInteger> counts
) {
    AtomicInteger count =
        counts.computeIfAbsent(stockCode, key -> new AtomicInteger());

    return count.incrementAndGet() == 1;
}

private void resubscribeActive() {
    List<String> prices =
        List.copyOf(priceSubscriberCounts.keySet());

    if (!prices.isEmpty()) {
        quoteIngestionService.subscribePrices(prices);
    }
}
```

* **코드 링크**: `MarketWebSocketHandler.java`

---

## 4. 팀 구성 및 역할

| 이름 | 담당 영역 |
| --- | --- |
| (팀원) | (역할 작성) |

<!-- 팀원별 역할을 작성하세요 -->
