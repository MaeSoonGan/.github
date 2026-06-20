# 🤝MaeSoonGan | 클라우드 기반 고가용성 모의투자 플랫폼

## ✍️프로젝트 한 줄 소개

- VMware 기반 On-Premise 인프라와 AWS EKS 클라우드 환경을 Site-to-Site VPN으로 연계하여, 트래픽 증가와 장애 상황에서도 안정적으로 서비스를 제공할 수 있도록 설계한 하이브리드 클라우드 기반 고가용성 증권 투자 플랫폼입니다.

<br />

## 🛫프로젝트 개요

- MaeSoonGan은 증권 투자 서비스를 안정적으로 운영하기 위한 클라우드 엔지니어링 프로젝트입니다.

- 증권 서비스는 짧은 장애나 응답 지연만으로도 주문 실패, 체결 지연, 자산 정보 불일치가 발생할 수 있습니다. 특히 다수 사용자의 동시 주문, 실시간 체결, 잔고 조회, 랭킹 갱신이 함께 이루어지므로 트래픽 증가와 서버 장애에 대비한 고가용성 인프라가 필요합니다.

- 본 프로젝트는 AWS Public Cloud와 VMware 기반 On-Premise 환경을 결합하여 실제 금융권에서 활용되는 하이브리드 인프라 구조를 모사했습니다. 사용자 서비스는 AWS EKS 기반 Kubernetes 환경에서 운영하고, 주문 정합성이 중요한 원장 시스템과 체결 엔진은 On-Premise Main 서버에 배치했습니다. 또한 Main 서버 장애 상황에 대비하여 DR 서버와 Veeam 기반 복구 구조를 구성하고, Prometheus, Grafana, Loki, Tempo, Alloy, Alertmanager를 활용해 AWS와 On-Premise 전 영역의 Metric, Log, Trace를 통합 수집하도록 설계했습니다.

- 이를 통해 단순한 웹 서비스 배포가 아니라, 고가용성, 장애 대응, DR 복구, 네트워크 분리, CI/CD 자동화, 통합 모니터링까지 고려한 운영 중심 인프라를 구축하는 것을 목표로 했습니다. 

<br />

## 📅프로젝트 기간

- 2026.04.23 ~ 2026.06.17

<br />

## 🧑‍💻팀원 소개 및 역할

| <img src="https://avatars.githubusercontent.com/u/127292182?v=4" width="150" height="150"/> | <img src="https://avatars.githubusercontent.com/u/180767288?v=4" width="150" height="150"/> | <img src="https://avatars.githubusercontent.com/u/178015712?v=4" width="150" height="150"/> | <img src="https://avatars.githubusercontent.com/u/107402806?v=4" width="150" height="150"/> | <img src="https://avatars.githubusercontent.com/u/181299322?v=4" width="150" height="150"/> |
| :----------------------------------------------------------------------------------------: | :-----------------------------------------------------------------------------------------: | :-----------------------------------------------------------------------------------------: | :-----------------------------------------------------------------------------------------: | :-----------------------------------------------------------------------------------------: |
|                     배기영<br/>[@bbky323](https://github.com/bbky323)                       |                      조민우<br/>[@minwoo-00](https://github.com/minwoo-00)                     |                        권순재<br/>[@Soooonnn](https://github.com/Soooonnn)                      |                   김소연<br/>[@thdus](https://github.com/thdus)                 |                     신성혁<br/>[@ssh221](https://github.com/ssh221)                   |
|                                           **PM / FE / ObservabilityServer**                                           |                                            **PL / BE / MainServer**                                           |                                            **BE / Network / DRServer**                                           |                                            **BE / AWS / DevOps**                                           |                                            **FE / BE / AWS**                                           |


<br />

## 🛖전체 서비스 아키텍처
<img width="11092" height="7200" alt="아키텍처1" src="https://github.com/user-attachments/assets/ef65fccc-567b-4892-9806-76a5f847f2ba" />


- 본 시스템은 AWS 클라우드와 On-Premise 환경을 Site-to-Site VPN으로 연결한 하이브리드 클라우드 기반 고가용성 인프라 구조입니다.

- 사용자의 요청은 Route53, WAF, ALB를 거쳐 AWS EKS 클러스터의 사용자 서비스, 관리자 서비스, 주문 서비스, 대회 서비스, 알림 서비스, 실시간 시세 서비스로 전달됩니다. 각 서비스는 Kubernetes 기반 Pod로 분산 배치되어 트래픽 증가 상황에 대응할 수 있도록 구성했습니다.

- On-Premise 영역에는 원장 시스템, 체결 엔진, Kafka, DB Cluster를 배치하여 주문 검증, 체결 처리, 이벤트 기반 데이터 처리를 담당하도록 설계했습니다. Main 서버 장애에 대비하여 DR 서버를 별도로 구성하고, Veeam 기반 VM 복구와 DR 전환 시나리오를 통해 핵심 서비스를 재개할 수 있도록 했습니다.

<br />

## 🌐전체 네트워크
<img width="1897" height="882" alt="아키텍처네트워크" src="https://github.com/user-attachments/assets/c3be4d95-fe82-4caf-9d98-fbfa47efe098" />

- 본 프로젝트는 AWS VPC와 On-Premise 네트워크를 Site-to-Site VPN으로 연결하여 사설 통신 기반의 하이브리드 클라우드 환경을 구성했습니다.

- 메인 서버와 DR서버의 경우 서비스망, DB Cluster망, CI/CD 관리망 등 역할에 따라 VLAN을 이용하여 네트워크를 분리했습니다. 이를 통해 장애 전파 범위를 줄이고, 서비스 트래픽, 데이터베이스 복제 트래픽, 관리 트래픽이 서로 영향을 최소화하도록 설계했습니다.

- 또한 각 물리 서버, AWS 서버의 사설망을 다르게 설정하여 독립적으로 운영할 수 있게 구성하였습니다.

<br />

## 🔗물리적 인프라 구조
<img width="952" height="627" alt="물리네트워크구성도" src="https://github.com/user-attachments/assets/14e5f524-0f6f-478c-8033-facf11aa61c9" />

- 실습 환경 특성상, 공유 자원을 이용해야 했으며, 해당 구조는 개인 물리 서버 1대, 공유 물리 서버 2대, 방화벽 및 라우터 서버 한 대를 이용하여 구성하였습니다.

- 개인 물리 서버 1대는 메인 서버, 공유 물리 서버 2대 중 1대는 DR 서버, 나머지 1대는 모니터링 서버로 구성하였으며, 방화벽 및 라우터 서버는 제공 노트북을 이용하였습니다.

- 각 물리 장치들은 Aruba 스위치를 이용하여 연결하였습니다.

<br />

## 👤소프트웨어 아키텍처
<img width="1920" height="1080" alt="소프트웨어아키텍처" src="https://github.com/user-attachments/assets/f468afc2-5841-4e69-9105-e843bec2dedb" />

- 소프트웨어 아키텍처는 Front-End부터 Database 및 External Interface까지 계층적으로 구성했습니다.

- 사용자 요청은 React 기반 Presentation Layer에서 시작되어 API Gateway, Controller, Service, Domain/Component, Persistence/Integration Layer로 전달됩니다. 회원, 주문, 계좌, 대회, 랭킹, 알림 등의 비즈니스 로직은 각 서비스 단위로 분리하고, On-Premise 원장 시스템과 체결 엔진은 핵심 거래 처리 영역으로 분리했습니다.

- 또한 Kafka Topic, MariaDB, Redis, Private API, Slack Webhook 등 외부 시스템과의 연결은 Persistence/Integration Layer에서 담당하도록 설계하여, 서비스 기능의 확장성과 유지보수성을 높였습니다.

<br />

## 👥통합 워크플로우
<img width="1672" height="941" alt="통합워크플로우다이어그램" src="https://github.com/user-attachments/assets/c546d94b-215c-4c6a-91d5-d1eaefd70437" />

<br />

## 🛠️핵심 기술 구성
<img width="1676" height="939" alt="핵심기술이미지" src="https://github.com/user-attachments/assets/ba06848c-b051-436b-a71c-3c1bcc342b87" />

- 본 프로젝트의 핵심 기술은 AWS Public Cloud, VMware 기반 On-Premise, Kubernetes, CI/CD, Observability, DR로 구성됩니다.

<br />

## 📚핵심 기술 스택

| 영역                        | 기술                                                        |
| ------------------------- | --------------------------------------------------------- |
| Frontend                  | React, JavaScript, PWA                                    |
| Backend                   | Java, Spring Boot                                         |
| Database                  | MariaDB, Redis                                            |
| Message Queue             | Kafka                                                     |
| Cloud & Container         | AWS EKS, ECR, ALB, WAF, Route53, Docker, Kubernetes       |
| On-Premise Infrastructure | VMware ESXi, vSphere, Ubuntu, Rocky Linux                 |
| Network                   | VyOS, pfSense, Site-to-Site VPN, VLAN, Routing, Firewall  |
| CI/CD & IaC               | GitHub, ArgoCD, Terraform, Ansible, Shell Script          |
| Observability             | Prometheus, Grafana, Loki, Tempo, Alloy, Alertmanager     |
| DR                        | Veeam Backup & Replication                                |

<br />

## 📋Repository 목차

### 애플리케이션

* [Frontend](https://github.com/MaeSoonGan/Front-End)
* [Backend](https://github.com/MaeSoonGan/Back-End)
* [Backend On-Prem](https://github.com/MaeSoonGan/Onprem-Back-End)

### 클라우드 인프라

* [AWS Infra](https://github.com/MaeSoonGan/Infra-AWS)
* [GitOps CI/CD](https://github.com/MaeSoonGan/Gitops-CI-CD)

### 온프레미스 인프라

* [On-Prem Main Server](https://github.com/MaeSoonGan/Onprem-Main-Server)
* [On-Prem DR Server](https://github.com/MaeSoonGan/Onprem-DR-Server)
* [On-Prem Observability Server](https://github.com/MaeSoonGan/Onprem-Observability-Server)

### 네트워크 및 운영

* [Network](https://github.com/MaeSoonGan/Network)

<br />

## 📽️주요 구현 기능 시연 영상

### AWS EKS 부하 테스트
https://youtu.be/Lh16cILNrZ8

### DR 테스트
https://youtu.be/DJYtDmqX4dU
