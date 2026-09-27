# 石나가는 판단 · SeokPan

**실시간 투표형 오목 서비스를 직접 구현하고, 이를 운영할 온프레미스 Kubernetes 플랫폼을 구축한 4인 팀 프로젝트입니다.** 네트워크·서버 자동화부터 애플리케이션, 승인 기반 배포, 메트릭·로그 수집, 장애 복구까지 하나의 서비스로 연결했습니다.

참여자는 팀별로 착수할 좌표에 투표하고, 서버가 투표를 마감해 수를 결정합니다. 로비·대기방·게임·채팅·랭킹을 제공하는 서비스를 통해 여러 Backend가 상태를 공유하고, 배포와 장애 상황에서도 데이터를 관리하는 구조를 검증했습니다.

## 전체 구성

![1차 프로젝트의 Kubernetes 서비스와 외부 인프라 구성](images/first-project-architecture.svg)

VMware·CentOS Stream 9 기반 **물리 호스트 4대, VM 16대**에서 Control Plane 3대와 Worker 2대를 구성했습니다. Kubernetes 내부의 서비스와 별도 VM의 데이터베이스·Registry·스토리지를 구분해 관리합니다.

- **서비스 요청**: HAProxy를 거쳐 NGINX Gateway Fabric에 도달하며, 페이지 요청은 Frontend로, `/api/v1`·`/ws/v1`은 Backend로 전달됩니다. HTTPS/WSS와 인증서 공급 경로를 구성했습니다.
- **상태 저장**: MariaDB는 회원·공식 착수·결과·레이팅을, Redis는 세션·방·투표·접속 상태를 맡습니다. Backend 2개 Replica가 같은 저장소를 사용하고, DB 접속은 MaxScale과 TLS를 거칩니다.
- **구축과 배포**: Ansible이 서버·클러스터 기반을 구성하고, GitOps가 Kubernetes 배포 설정을 관리합니다. Jenkins가 검증된 이미지로 배포 PR을 생성하며, 팀원의 승인·Merge 후 Argo CD가 반영합니다.

## 주요 구현

| 영역 | 구현한 내용 | 확인할 코드·자료 |
|---|---|---|
| 인프라 자동화 | 라우팅·방화벽·HAProxy, kubeadm·containerd·Calico, DB·NFS 및 인증서 공급을 Ansible Role로 구성 | [Infra](https://github.com/seokpan/seokpan-infra) |
| 실시간 서비스 | React·TypeScript UI, FastAPI HTTP/WebSocket API, 투표 마감·게임 결과 저장, Replica 간 세션·방 상태 공유 | [Application](https://github.com/seokpan/seokpan-app) |
| 승인 기반 배포 | Jenkins 테스트·이미지 빌드·스캔, Harbor Digest 확정, 변경 컴포넌트의 GitOps PR 생성, Argo CD 동기화 | [이미지 파이프라인](https://github.com/seokpan/seokpan-app/blob/main/Jenkinsfile.image-pipeline) · [GitOps](https://github.com/seokpan/seokpan-gitops) |
| 관측성 | Prometheus 메트릭 수집, Alloy의 컨테이너 로그 수집과 Loki 조회, Grafana 대시보드·Alertmanager 알림 구성 | [Observability](https://github.com/seokpan/seokpan-gitops/tree/main/observability) |
| 데이터·복구 | MariaDB 복제와 백업 체인, etcd Snapshot 복구 자동화, Redis AOF/PVC와 복구 검증 | [복구 Playbook](https://github.com/seokpan/seokpan-infra/tree/main/ansible/playbooks) · [검증 기록](https://github.com/seokpan/seokpan-docs/tree/main/12_MVP_검증·측정_계획) |

## 팀과 담당

각 영역을 나누어 구현하고, 저장소 간 인터페이스와 변경 사항은 Issue·PR 리뷰로 조율했습니다. 아래는 주 담당 영역이며, 통합·리뷰·검증은 함께 진행했습니다.

| 팀원 | 주 담당 영역 |
|---|---|
| 정태훈 · 팀장 | Kubernetes·Calico·Gateway, Frontend/Backend 구현, Redis 상태 연동, 애플리케이션·플랫폼 통합 |
| 이유빈 | 네트워크·라우팅·방화벽·LB, Ansible 실행 환경과 안전 실행 도구 |
| 김상희 | MariaDB·MaxScale·NFS, 데이터 백업·복구, etcd·Redis 복구 검증 |
| 최유준 | Harbor·Jenkins·Argo CD, 이미지 배포 자동화, 인증서 기반 구성, 메트릭·로그·알림 |

담당 범위는 [Infra 역할표](https://github.com/seokpan/seokpan-infra#담당과-역할), [역할별 실행·통합 설계](https://github.com/seokpan/seokpan-docs/tree/main/09_MVP_실행·통합_실시설계)와 각 저장소의 PR·커밋에서 확인할 수 있습니다.

## 검증 결과와 1차 완료 범위

1차 프로젝트는 **서비스 구현·인프라 통합을 마쳤으며**, 종료 시점의 결과와 추가 검증 대상은 [2026-09-23 Current State](https://github.com/seokpan/seokpan-docs/blob/main/CURRENT_STATE.md)에 정리했습니다.

| 검증 영역 | 기록된 결과 |
|---|---|
| 서비스 통합 | Gateway HTTPS/API/WebSocket, Backend·Frontend 각 2개 Replica, 브라우저 게임 정상 흐름과 랭킹·통계 확인 |
| 배포 운영 | 실제 GitOps Promotion PR 생성, Argo CD Self-Heal, Git Revert 기반 Runtime Rollback 확인 |
| 관측성 | Backend 2개 Replica의 Prometheus Target `UP`, 앱 메트릭 조회, 구조화된 오류 로그의 Loki 조회 확인 |
| MariaDB 복구 | 양쪽 DB 유실 시나리오에서 단일 노드 서비스 재개 **1분 29초**, 이중화 정상화 **4분**, 미백업 데이터 **3건 손실** |
| etcd 복구 | Snapshot Restore부터 Quorum·API·Kubernetes Object 확인까지 **52초** — 전체 클러스터 재구축 시간과는 구분 |

복구 수치는 해당 실측 환경의 결과입니다. 절차·측정 범위·근거는 [역할별 검증·측정 문서](https://github.com/seokpan/seokpan-docs/tree/main/12_MVP_검증·측정_계획)에서 확인할 수 있습니다. 재접속·다중 Pod·장애 경계의 강화 검증은 [App #112](https://github.com/seokpan/seokpan-app/issues/112)로 이어집니다. HPA·AI 분석 Runtime·추가 LB/MaxScale 이중화는 1차 완료 범위에 포함하지 않습니다.

## 저장소 둘러보기

| 저장소 | 여기서 확인할 내용 |
|---|---|
| [seokpan-app](https://github.com/seokpan/seokpan-app) | 서비스 코드, API·도메인 계약, 테스트, 컨테이너·CI |
| [seokpan-infra](https://github.com/seokpan/seokpan-infra) | 서버·네트워크·클러스터 구축, DB·스토리지·복구 자동화 |
| [seokpan-gitops](https://github.com/seokpan/seokpan-gitops) | Kubernetes 배포 설정, Argo CD, 플랫폼·CI/CD·관측성 리소스 |
| [seokpan-docs](https://github.com/seokpan/seokpan-docs) | 설계, 변경 결정, Runbook, 검증 결과, 트러블슈팅 |

구현 과정에서 해결한 문제는 [트러블슈팅 모음](https://github.com/seokpan/seokpan-docs/tree/main/troubleshooting)에 원인·조치·재검증 근거와 함께 남겼습니다. 협업은 **Issue → Branch → PR → Review → Merge** 순서로 진행하며, 세부 기준은 [저장소 운영 문서](https://github.com/seokpan/seokpan-docs/tree/main/10_GitHub_협업_및_Repository_운영)를 따릅니다.
