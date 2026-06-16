# [우리FISA 6기] 클라우드 엔지니어링 1팀 - 매순간(MaeSoonGan)

## 1. 프로젝트 개요

- **주제:** 클라우드 기반 고가용성 인프라 환경의 모의투자 대회 플랫폼
- **프로젝트 기획 배경:** 증권 서비스는 짧은 장애나 응답 지연만으로도 주문 실패, 체결 지연, 자산 정보 불일치가 발생할 수 있어 **높은 안정성**이 요구됩니다. 특히 모의투자 대회 서비스는 다수 사용자의 동시 주문과 실시간 랭킹 확인이 이루어지므로, **트래픽 증가와 서버 장애에 대비한 고가용성** 인프라가 필요합니다.또한 Main 서버 장애가 장기화될 경우 대회 현황, 주문 내역, 보유 자산 조회가 중단될 수 있으므로, **Veeam 기반 DR 서버 복구** 구조를 통해 핵심 서비스를 빠르게 재개하고 서비스 연속성을 확보하고자 했습니다.
- **기술 스택:**

    | 영역                         | 기술                                                        |
    | -------------------------- | --------------------------------------------------------- |
    | **Frontend**               | React, JavaScript                                         |
    | **Backend**                | Java, Spring Boot                                         |
    | **Database**               | MariaDB, Redis                                            |
    | **Message Queue**          | Kafka                                                     |
    | **Cloud & Container**      | AWS EKS, ECR, ALB, WAF, Route53, Docker, Kubernetes       |
    | **On-Prem Infrastructure** | VMware ESXi, vSphere, Ubuntu, Rocky Linux                 |
    | **Network**                | VyOS, pfSense, VPN Site-to-Site                           |
    | **CI/CD & IaC**            | GitHub, Jenkins, ArgoCD, Terraform, Ansible, Shell Script |
    | **Observability**          | Prometheus, Grafana, Loki, Tempo, Alloy, Alertmanager     |
    | **DR**                     | Veeam Backup & Replication                                |

## 2. 아키텍처

### 2-1. 시스템 아키텍처

<p align="center">
  <img width="2850" height="1780" alt="최종_전체_아키텍처" src="https://github.com/user-attachments/assets/89a51d35-79e8-4cfe-9f3a-c0aabf563187" />
</p>

### 설명

본 시스템은 **AWS 클라우드**와 **On-Premise 환경**을 **Site-to-Site VPN**으로 연결한 **하이브리드 클라우드 기반 고가용성 인프라 구조**입니다. 사용자의 요청은 **Route53, WAF, ALB를 거쳐 AWS EKS 클러스터**의 사용자 서비스, 관리자 서비스, 실시간 시세 서비스, 비동기 처리 서비스로 전달되며, 각 서비스는 **Kubernetes 기반 Pod로 분산 배치**되어 안정적인 서비스 운영과 확장성을 확보합니다.

AWS 내부에는 M**ariaDB와 Redis를 Primary/Replica 구조로 구성**하여 데이터 저장과 캐시 처리를 지원하고, **S3와 CloudWatch를 통해 파일 저장 및 클라우드 리소스 모니터링**을 수행합니다. On-Premise 영역에는 원장 서버, 체결 엔진, Kafka로 구성된 **Core Cluster**와 Master/Slave **DB Cluster**를 배치하여 주문 검증, 체결 처리, 이벤트 기반 데이터 처리를 담당합니다.

또한 Main 서버 장애에 대비하여 **DR Cluster**를 별도로 구성하고, **Veeam 기반 복제 및 복구 구조**를 통해 장애 발생 시 핵심 서비스를 빠르게 재개할 수 있도록 설계했습니다. Monitoring 서버에는 Prometheus, Alertmanager, Grafana, Loki, Jaeger 등을 구성하여 **AWS와 On-Premise 서버의 Metric, Log, Trace를 수집하고 시각화**하며, **장애 발생 시 알림**을 통해 운영자가 신속하게 대응할 수 있도록 지원합니다.


### 2-2. 소프트웨어 아키텍처

<img width="1920" height="1080" alt="소프트웨어아키텍처" src="https://github.com/user-attachments/assets/06244c31-0078-49ac-8b9c-8458b035a203" />


### 설명

해당 아키텍처는 **Front-End부터 Database 및 External Interface**까지 계층적으로 구성된 Layered 구조로, 각 레이어가 역할에 따라 분리되어 있습니다. 사용자의 요청은 React 기반 Presentation Layer에서 시작되어 API/Gateway Layer를 거쳐 Controller, Service, Domain/Component, Persistence/Integration Layer로 순차적으로 전달되며, 회원, 주문, 계좌, 대회, 랭킹, 알림 등의 비즈니스 로직이 처리됩니다.

또한 On-Prem 원장 시스템, 체결 엔진, Kafka와 연동되는 **Domain/Component 영역**을 별도로 구성하여 **핵심 거래 처리 로직을 분리**하고, Persistence/Integration Layer에서 MariaDB, Redis, Kafka Topic, Private API, Slack Webhook 등 외부 시스템과의 연결을 담당하도록 설계했습니다. 이를 통해 서비스 기능의 확장성과 유지보수성을 높이고, 각 기능 간 의존도를 낮춘 구조를 구현했습니다.

운영 측면에서는 AWS EKS, On-Prem Private, DB/Cache, CI/CD, Observability 영역을 별도로 표현하여 **애플리케이션 계층과 인프라 운영 구성을 구분**했습니다. Prometheus, Grafana, Loki, Tempo, Alloy, Alertmanager를 활용한 Metric, Log, Trace 수집 및 알림 구조를 통해 **서비스 상태를 관측하고 장애 대응이 가능**하도록 설계했습니다.


## 3. 주요 기능 소개

### 3-1. 핵심 기술 구성

<img width="1676" height="939" alt="핵심기술이미지" src="https://github.com/user-attachments/assets/386132fd-7a89-4c1a-8707-bd78084eff59" />

### 3-2. 통합 워크플로우 다이어그램

<img width="1672" height="941" alt="통합워크플로우다이어그램" src="https://github.com/user-attachments/assets/c7555c4e-1c7b-46c4-85e1-cf5ed5b047f9" />


### 3-3. 세부 기능 소개

#### [기능 1] AWS EKS 기반 오토스케일링 및 안정적인 서비스 운영

- **기능 설명:**
사용자 접속 증가나 주문 요청 증가 상황에서도 서비스가 안정적으로 동작할 수 있도록 AWS EKS 기반 Kubernetes 환경에 주요 API 서비스를 배포했습니다. HPA를 통해 Pod 리소스 사용량에 따라 자동 확장되도록 구성하고, ALB와 Ingress를 통해 외부 요청을 각 서비스로 전달하여 트래픽 증가 상황에 대응할 수 있도록 했습니다.

- **핵심 코드(스크립트):**
```YAML
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: maesoongan-ingress
  namespace: maesoongan
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP":80}]'
spec:
  rules:
    - host: maesoongan.xyz
      http:
        paths:
          - path: /api/auth
            pathType: Prefix
            backend:
              service:
                name: auth-service
                port:
                  number: 8080

          - path: /api/orders
            pathType: Prefix
            backend:
              service:
                name: order-service
                port:
                  number: 8080

          - path: /api/contests
            pathType: Prefix
            backend:
              service:
                name: contest-service
                port:
                  number: 8080

          - path: /api/markets
            pathType: Prefix
            backend:
              service:
                name: market-service
                port:
                  number: 8080
```

- **코드 링크(스크립트 링크):**
[maesoongan-ingress.yaml](https://github.com/user-attachments/files/29013803/maesoongan-ingress.yaml), [frontend-deploy-files.zip](https://github.com/user-attachments/files/29012371/frontend-deploy-files.zip)

---

#### [기능 2] Observability 기반 장애 감지 및 Slack 알림

- **기능 설명:**
장애 발생 시 운영자가 빠르게 상태를 파악할 수 있도록 Prometheus, Grafana, Loki, Tempo, Alloy, Alertmanager 기반 Observability 환경을 구성했습니다. 서버와 애플리케이션의 Metric, Log, Trace를 수집하고 Grafana에서 시각화하며, 장애 조건이 감지되면 Alertmanager를 통해 Slack으로 알림을 전송하도록 구성했습니다.

- **핵심 코드(스크립트):**
```YAML
route:
  receiver: slack-warning
  group_by:
    - alertname
    - area
    - host
    - service
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 1h

  routes:
    - matchers:
        - severity="critical"
      receiver: slack-critical
      continue: false

    - matchers:
        - severity="warning"
      receiver: slack-warning
      continue: false

inhibit_rules:
  - source_matchers:
      - severity="critical"
    target_matchers:
      - severity="warning"
    equal:
      - alertname
      - area
      - host
      - service

```

- **코드 링크(스크립트 링크):**
[observability-configs.zip](https://github.com/user-attachments/files/29012838/observability-configs.zip)

---

#### [기능 3] Veeam 기반 DR 서버 복구 및 서비스 재개

- **기능 설명:**
Main 서버 장애가 장기화될 경우에도 핵심 서비스를 빠르게 재개할 수 있도록 Veeam 기반 백업 및 DR 복구 구조를 구성했습니다. Main 서버의 VM 백업을 기반으로 DR 서버를 복구하고, 복구 이후 서비스 상태와 모니터링 수집 설정을 점검하여 장애 상황에서도 서비스 연속성을 확보할 수 있도록 했습니다.

- **핵심 코드(스크립트):**
```ps1
$parts = @(
    'set -e;',
    'CONN=$(nmcli -t -f NAME,DEVICE con show --active | grep -v ":lo$" | head -n1 | cut -d: -f1);',
    'test -n "$CONN" || { echo "No active NetworkManager connection found"; nmcli -t -f NAME,DEVICE con show --active; exit 1; };',
    'echo "=== Before ===";',
    'ip -br addr;',
    'ip route;',
    'hostname;',
    'sudo -n hostnamectl set-hostname "' + $NewHostname + '";',
    'sudo -n nmcli con mod "$CONN" ipv4.addresses "' + $IpCidr + '" ipv4.gateway "' + $Gateway + '" ipv4.dns "' + $Dns + '" ipv4.method manual;',
    'sudo -n nmcli con up "$CONN";',
    'echo "=== After ===";',
    'ip -br addr;',
    'ip route;',
    'hostname;'
)

$bash = ($parts -join ' ')

$result = Invoke-VMScript `
    -VM $vm `
    -GuestCredential $guestCred `
    -ScriptType Bash `
    -ScriptText $bash `
    -ToolsWaitSecs 180
```


- **코드 링크(스크립트 링크):**
[Veeam_VM_IP바꾸는스크립트.zip](https://github.com/user-attachments/files/29013033/Veeam_VM_IP.zip)


---

#### [기능 4] Kafka·Redis 기반 거래 정합성 보장

- **기능 설명:**
주문, 체결, 알림, 랭킹 갱신 과정에서 데이터 정합성이 깨지지 않도록 Kafka와 Redis를 활용했습니다. Kafka는 주문 요청과 체결 결과를 비동기 이벤트로 전달하여 채널계와 계정계 간 결합도를 낮추고, Consumer 멱등 처리를 통해 동일 이벤트가 재전송되어도 중복 반영되지 않도록 했습니다. 또한 Redis Lua Script를 활용해 증거금 선점과 취소 보상을 원자적으로 처리하여 동시 주문 상황에서도 잔고 불일치를 방지했습니다.

- **핵심 코드(스크립트):**
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

- **코드 링크(스크립트 링크):**
[BalanceCache.java](https://github.com/MaeSoonGan/Back-End/blob/develop/apps/api/order-service/src/main/java/com/mock/maesoongan/orderservice/order/BalanceCache.java)

---

#### [기능 5] On-Prem 클러스터링 및 HA/FT 기반 서버 이중화

- **기능 설명:**
On-Premise 환경에서 단일 서버 장애로 인해 핵심 거래 시스템이 중단되지 않도록 클러스터링 및 HA/FT 기반 이중화 구조를 구성했습니다. Main 서버, 원장 시스템, 체결 엔진 등 주요 구성 요소가 장애 상황에서도 복구 가능하도록 설계하고, 서버 장애 발생 시 대체 자원으로 전환될 수 있는 구조를 마련했습니다. 이를 통해 온프레미스 내부 장애에 대한 복원력을 높이고 안정적인 운영 기반을 확보했습니다.

- **핵심 코드(스크립트):**
```shell script
#!/bin/bash
# Orchestrator PostFailover 훅 — 신 master에서 실행할 후처리
# 인자: $1 = 신 master host, $2 = 신 master port

NEW_MASTER_HOST="$1"
NEW_MASTER_PORT="$2"
CNF="/etc/orchestrator-mysql.cnf"
LOG="/tmp/recovery.log"

echo "$(date '+%Y-%m-%d %H:%M:%S') post-failover START on ${NEW_MASTER_HOST}:${NEW_MASTER_PORT}" >> "$LOG"

# 1) 신 master semi-sync source 재활성화 (토글로 내부 동작 깨우기)
mysql --defaults-extra-file="$CNF" -h"$NEW_MASTER_HOST" -P"$NEW_MASTER_PORT" \
  -e "SET GLOBAL rpl_semi_sync_source_enabled=OFF; SET GLOBAL rpl_semi_sync_source_enabled=ON; SET GLOBAL rpl_semi_sync_source_timeout=10000;" \
  >> "$LOG" 2>&1

if [ $? -eq 0 ]; then
  echo "$(date '+%Y-%m-%d %H:%M:%S') semi-sync source re-enabled on ${NEW_MASTER_HOST}" >> "$LOG"
else
  echo "$(date '+%Y-%m-%d %H:%M:%S') ERROR: semi-sync toggle failed on ${NEW_MASTER_HOST}" >> "$LOG"
fi

# --- 향후 확장 지점 ---
# 2) ProxySQL writer 호스트그룹 전환 (ProxySQL 도입 후)
# 3) DNS 레코드 갱신 (nsupdate, DR 단계에서)
# 4) 정합성 동기화 API 호출 (채널계 RDS/Redis 재구성)

echo "$(date '+%Y-%m-%d %H:%M:%S') post-failover DONE on ${NEW_MASTER_HOST}:${NEW_MASTER_PORT}" >> "$LOG"
```

- **코드 링크(스크립트 링크):**
[orch-post-failover.sh](https://github.com/user-attachments/files/29013191/orch-post-failover.sh)


