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

| 영역 | 핵심 기술 | 적용 내용 |
| --- | --- | --- |
| 인증 | Spring Security + JWT | Access(1h) / Refresh(7d) 토큰, Refresh는 Redis 관리, 로그아웃 시 블랙리스트 처리 |
| 주문 처리 | Kafka 비동기 파이프라인 | 주문 접수 → `order.requested` 발행 → 체결 → `order.completed` 발행 |
| 동시성 제어 | Redis 잔고 예약 + MySQL `FOR UPDATE` | 1차 Redis 검증, 2차 원장 비관적 락으로 잔고 이중 차감 방지 |
| 실시간 전송 | WebSocket + Redis Pub/Sub | 체결 결과·호가창 실시간 Push (지연 1초 이내) |
| 시세 | 한국투자증권 Open API + Redis Cache | 실시간 시세·호가 스트리밍, 캔들 데이터 캐싱 |
| 관측성 | OpenTelemetry E2E Tracing | 클라우드↔온프레 단일 TraceID로 주문 전 구간 추적 |

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

#### [기능 1] 채널계–계정계 분리 하이브리드 아키텍처

* **기능 설명**: 실제 증권사처럼 고객 접점(채널계)과 원장(계정계)을 물리적으로 분리하여, 트래픽 변동이 큰 채널계는 AWS EKS에서 수평 확장하고, 정합성이 중요한 원장은 온프레미스에서 운영합니다. 두 환경은 Site-to-Site VPN으로 연결되며, 비동기 이벤트는 온프레미스 Kafka로, 동기 조회는 VPN을 경유한 HTTP로 처리합니다.
* **설계 의도**: 채널계와 계정계를 분리함으로써 ① 채널계 장애가 원장으로 전파되지 않고, ② 원장이 단일 진실 공급원으로서 잔고 정합성을 책임지며, ③ 클라우드의 탄력성과 온프레미스의 통제력을 동시에 확보할 수 있습니다.

#### [기능 2] 2단계 잔고 검증을 통한 동시성 제어

* **기능 설명**: 동일 사용자가 동시에 여러 주문을 시도하거나 한 종목에 주문이 몰릴 때, 잔고가 중복 차감되지 않도록 **2단계 검증**을 적용했습니다. 1차로 채널계에서 Redis에 잔고를 예약(`balance:{memberId}:{contestId}`)하여 빠르게 차단하고, 2차로 체결 시점에 원장 DB에서 `SELECT ... FOR UPDATE` 비관적 락으로 최종 잔고를 차감합니다.
* **설계 의도**: Redis만으로는 장애 시 정합성을 보장하기 어렵고, DB 락만으로는 대량 트래픽을 감당하기 어렵습니다. 빠른 1차 필터(Redis)와 정합성 보장 2차 확정(DB 락)을 분리하여 성능과 정확성을 모두 확보했습니다.
* **핵심 코드(예시)**:

```java
// 1차: 채널계 — Redis 잔고 예약 (주문 접수 시)
public void reserveBalance(Long memberId, Long contestId, long amount) {
    String key = "balance:" + memberId + ":" + contestId;
    Long remaining = redisTemplate.opsForValue().decrement(key, amount);
    if (remaining == null || remaining < 0) {
        redisTemplate.opsForValue().increment(key, amount); // 롤백
        throw new BusinessException("주문 가능 금액 부족");
    }
}

// 2차: 계정계 — 원장 DB 비관적 락 최종 차감 (체결 시)
@Transactional
public void confirmBalance(Long accountId, long amount) {
    Account account = accountRepository.findByIdForUpdate(accountId) // SELECT ... FOR UPDATE
            .orElseThrow();
    account.withdraw(amount);
}
```

* **코드(스크립트) 링크**: `OrderService.java`, `BalanceCache.java` <!-- 실제 파일 경로/링크로 교체 -->

#### [기능 3] 주문 취소 시 잔고 이중 사용 방지 (TTL 기반 설계)

* **기능 설명**: 주문 취소 시 잔고를 **즉시 복구하지 않고**, `cancel:pending:{orderId}` 키에 TTL을 부여하여 일정 시간 보류합니다. 체결 엔진의 취소 확정 이벤트를 받은 뒤에 잔고를 복구하도록 설계해, "취소 직후 동일 잔고로 즉시 재주문"하는 이중 사용을 차단합니다.
* **설계 의도**: 취소 요청과 실제 체결 엔진의 취소 확정 사이에는 시간 차가 존재합니다. 이 구간에서 잔고를 곧바로 풀어주면 동일 금액이 두 주문에 동시에 쓰일 수 있습니다. 취소 상태를 `CANCEL_REQUESTED`로 두고 TTL로 보류하는 방식은 이 정합성 공백을 설계 단계에서 선제적으로 막기 위한 결정입니다.
* **코드(스크립트) 링크**: `BalanceCache.java` <!-- 실제 파일 경로/링크로 교체 -->

#### [기능 4] Kafka 기반 비동기 주문 처리와 멀티 Consumer Group

* **기능 설명**: 주문 접수는 사용자에게 즉시 '접수됨'을 응답하고, 실제 체결은 Kafka 이벤트로 비동기 처리합니다. 체결 완료(`order.completed`)는 하나의 토픽을 **알림·랭킹·WebSocket 세 개의 독립 Consumer Group**이 각자 소비하여, 한 번의 체결로 푸시 알림·랭킹 집계·실시간 화면 갱신이 동시에 일어납니다.
* **설계 의도**: 동기 처리 시 체결 지연이 사용자 응답 지연으로 직결됩니다. 이벤트 기반으로 분리하면 응답 속도를 보장하면서, Consumer Group별 Offset을 독립 관리해 한 소비자의 지연이 다른 소비자에게 영향을 주지 않습니다.

```yaml
# application-onprem.yml (발췌)
spring:
  kafka:
    bootstrap-servers: 10.4.10.33:9092   # 온프레미스 Kafka (VPN 경유)
    consumer:
      # 알림 / 랭킹 / WebSocket 각 그룹이 동일 토픽을 독립 소비
      group-id: notification-group
```

#### [기능 5] 스냅샷 기반 조회 분리 (읽기/쓰기 책임 분리)

* **기능 설명**: 포트폴리오·잔고·보유종목 조회는 원장 DB에 직접 접근하지 않고, **별도의 스냅샷 테이블** 기준으로만 조회합니다. 쓰기(체결·차감)는 원장이, 읽기(조회)는 스냅샷이 담당하여 조회 트래픽이 원장 성능에 영향을 주지 않습니다.
* **설계 의도**: 조회 요청은 주문/체결보다 훨씬 빈번합니다. 읽기와 쓰기 경로를 분리하면 원장은 정합성에만 집중하고, 조회는 스냅샷에서 빠르게 응답할 수 있어 시스템 전체의 안정성과 확장성이 높아집니다.

---

## 4. 팀 구성 및 역할

| 이름 | 담당 영역 |
| --- | --- |
| (팀원) | (역할 작성) |

<!-- 팀원별 역할을 작성하세요 -->
