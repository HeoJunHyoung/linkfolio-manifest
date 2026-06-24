# Linkfolio Manifest (GitOps with ArgoCD)

본 레포지토리는 **Linkfolio** 프로젝트의 쿠버네티스 리소스 관리를 위한 GitOps 매니페스트 전용 저장소입니다. `ArgoCD`와 `Kustomize`, 그리고 `App of Apps` 패턴을 활용하여 대규모 마이크로서비스(MSA) 인프라 및 애플리케이션을 선언적으로 배포하고 관리합니다.

## 🔗 1. 링크폴리오 관련 주소
* **Backend Repository (Dev):** [linkfolio-backend](https://github.com/heojunhyoung/linkfolio-backend)
* **Manifest Repository (Ops):** [linkfolio-manifest (dev branch)](https://github.com/heojunhyoung/linkfolio-manifest/tree/dev)
* **Frontend Service:** [https://linkfolio-three.vercel.app/](https://linkfolio-three.vercel.app/)

---

## 🏗 2. 저장소 아키텍처 및 GitOps 분석

본 프로젝트는 애플리케이션 소스 코드(Backend)와 배포 매니페스트(Manifest)를 완전히 분리하는 **GitOps의 Best Practice**를 따르고 있습니다. 이를 통해 개발자(Dev)는 비즈니스 로직에만 집중하고, 인프라(Ops)는 이 저장소를 '단일 진실 공급원(SSOT)'으로 삼아 클러스터의 상태를 관리합니다.

### 🧩 2.1. App of Apps 패턴 (`applications/`)
수많은 마이크로서비스와 인프라 리소스를 개별적으로 배포하는 대신, **ArgoCD의 App of Apps 패턴**을 도입하여 클러스터 전체를 하나의 시스템으로 관리합니다.
* **`root-app.yaml`**: 최상위 부모 애플리케이션입니다. 이 파일 하나만 클러스터에 배포하면, 하위의 모든 애플리케이션이 연쇄적으로 동기화됩니다.
* **`development-apps/`**: `auth-service`, `user-service`, `infra-database`, `monitoring` 등 각각의 서비스를 정의한 ArgoCD Application 매니페스트들이 모여 있는 폴더입니다. 이를 통해 클러스터의 전체 아키텍처를 한눈에 파악하고, 일괄 생성 및 삭제가 가능합니다.

### 🛠 2.2. Kustomize 기반의 환경 분리 (`base/` vs `overlays/`)
코드의 중복을 막고 여러 환경(Dev, Prod 등)을 효율적으로 관리하기 위해 **Kustomize**를 적극 활용했습니다.
* **`base/` (공통 리소스):**
  * 모든 환경에서 변하지 않는 애플리케이션의 뼈대(Deployment, Service, ConfigMap, SealedSecret 등)가 정의되어 있습니다.
  * DB 리소스(`database-commons`), 스토리지 리소스(`storage`), 모니터링 스택(`monitoring`) 등 공통 인프라도 이곳에서 관리됩니다.
* **`overlays/development/` (환경별 패치):**
  * 개발 환경(`dev`)에만 적용되는 특화된 설정들을 오버라이딩(Overriding)합니다.
  * 예: 외부 접속을 위한 `apigateway/ingress.yaml` 추가, Ngrok 로컬 테스트를 위한 `patch-auth-configmap-ngrok.yaml` 환경변수 패치 등 베이스 코드를 수정하지 않고 환경에 맞게 리소스를 변형합니다.

### 🛡 2.3. 주요 구성 요소 및 보안 (Sealed Secrets)
* **마이크로서비스:** `apigateway`, `auth`, `user`, `chat`, `community`, `portfolio`, `support` 등 7개의 서비스 모듈 구성
* **데이터베이스:** 공유 DB(`shared-mysql`, `shared-redis`)와 격리 DB(`chat-mongodb`, StatefulSet)의 분리 운영
* **모니터링 스택:** Prometheus, Grafana, Loki, Promtail, Node-exporter, Kube-state-metrics를 통합한 PLG + Prometheus 옵저버빌리티 구축
* **GitOps 보안:** DB 패스워드나 API 키와 같은 민감 정보는 평문 ConfigMap이 아닌 **Bitnami Sealed Secrets**(`sealed-secret.yaml`)를 사용하여 암호화된 상태로 Git에 안전하게 업로드 및 관리됩니다.

---

## 🚀 3. 실행 가이드

### [1] VirtualBox 실행
* 1개의 Master Node와 3개의 Worker Node(Ubuntu)를 실행합니다.
* **주의:** 반드시 Master Node의 부팅이 완료된 후 나머지 Worker Node들을 실행해야 합니다.

### [2] Ingress 포트포워딩 확인
Master Node에서 아래 명령어를 통해 `ingress-nginx`가 어느 노드에 떠있는지 확인합니다.
```bash
kubectl get pods -n ingress-nginx -o wide
```
* `NODE`가 `k8s-worker1`인 경우: 포트포워딩을 `10.0.2.7`로 설정
* `NODE`가 `k8s-worker2`인 경우: 포트포워딩을 `10.0.2.8`로 설정

### [3] NFS Server 실행 (사전 작업)
현재 k8s on-premise 환경에서는 PV, PVC 기반으로 외부 스토리지(NFS)를 사용하고 있습니다. NFS Server 담당이 **worker node2**이므로, **worker node2**에서 아래 명령어를 실행해야 합니다. (ArgoCD를 통해 DB Pod들이 정상적으로 뜨기 위한 필수 선행 작업입니다.)

```bash
# nfs_start2.sh 내용 실행
sudo mkdir -p /tmp/k8s-pv/shared-mysql
sudo mkdir -p /tmp/k8s-pv/shared-redis
sudo mkdir -p /tmp/k8s-pv/chat-mongodb

sudo chown -R 999:999 /tmp/k8s-pv/shared-mysql
sudo chown -R 999:999 /tmp/k8s-pv/shared-redis
sudo chown -R 999:999 /tmp/k8s-pv/chat-mongodb

sudo chmod -R 777 /tmp/k8s-pv

sudo systemctl restart nfs-server
sudo systemctl status nfs-server
```

### [4] Kafka 실행
Kafka 노드에 접속하여 아래 명령어를 순차적으로 실행합니다.
```bash
kafka@kafka:~$ sudo ./start_all.sh
kafka@kafka:~$ sudo jps
kafka@kafka:~$ ./publish_topics.sh
```

### [5] ArgoCD 콘솔 접속
ArgoCD의 포트포워딩이 `127.0.0.1:8443` → `10.0.2.7:30277`로 되어있습니다.
* **주소:** [https://127.0.0.1:8443/](https://127.0.0.1:8443/) (반드시 HTTPS로 접속)
* **ID:** `admin`
* **PW:** `ZGE3oiMAC5wPJJYq`

### [6] ArgoCD GitHub Manifest Repository 연동
ArgoCD의 `Settings > Repositories`에서 다음 정보로 연동합니다.
* **Choose your connection method:** VIA HTTPS
* **Type:** git
* **Name (optional):** linkfolio-repo
* **Project:** default
* **Repository URL:** `https://github.com/heojunhyoung/linkfolio-manifest.git`
* **Username:** HeoJunHyoung
* **Password:** GitHub Personal Access Token (PAT)

### [7] ArgoCD Application 실행 (App of Apps)
`New App`을 클릭하고 아래 정보를 입력하여 실행합니다.
*(실행 후 리소스가 뜨지 않는다면 클러스터에 `linkfolio` namespace를 생성해주세요.)*

* **Application Name:** `linkfolio-root-development`
* **Project:** `default`
* **Sync Policy:** `Manual`
* **Repository URL:** `https://github.com/heojunhyoung/linkfolio-manifest.git`
* **Revision:** `feature/unified-infra-app-of-apps`
* **Path:** `applications/development-apps`
* **Cluster URL:** `https://kubernetes.default.svc`
* **Namespace:** `argocd`

### [8] Ngrok 실행
로컬 API Gateway 연동을 위해 Ngrok을 실행합니다.
```bash
ngrok http 80 --host-header="linkfolio.127.0.0.1.nip.io"
```
* 실행 후 생성된 Swagger 주소로 접속하여 상태를 확인합니다. (예: `https://impressionless-connaturally-jonie.ngrok-free.dev/swagger-ui.html`)

### [9] Kafka Connector 연결
카프카 커넥터를 등록합니다.
```bash
kafka@kafka:~$ cd /opt/kafka/connect-configs

# 1. User Profile Connector 등록 (기존)
curl -i -X POST -H "Accept:application/json" -H "Content-Type:application/json" \
http://localhost:8083/connectors \
-d @user-profile-connector.json

# 2. Auth Outbox Connector 등록 (신규)
curl -i -X PUT -H "Accept:application/json" -H "Content-Type:application/json" \
http://localhost:8083/connectors/auth-outbox-connector/config \
-d "$(jq -c '.config' auth-outbox-connector.json)"

# 상태 확인 (둘 다 RUNNING 이어야 함)
curl -X GET [http://127.0.1.1:8083/connectors/user-profile-connector/status](http://127.0.1.1:8083/connectors/user-profile-connector/status) | jq
curl -X GET [http://127.0.1.1:8083/connectors/auth-outbox-connector/status](http://127.0.1.1:8083/connectors/auth-outbox-connector/status) | jq
```

---

## 🧹 4. 리소스 릴리즈

### [0] 카프카 리소스 정리
```bash
kafka@kafka:~$ ./stop_all.sh
```

### [1] ArgoCD 애플리케이션 종료
ArgoCD GUI에서 `DELETE`를 클릭하거나 아래 명령어를 수행합니다.
```bash
kubectl delete application linkfolio-root-development -n argocd --cascade=foreground
```

### [2] PVC 및 PV 제거
반드시 `kubectl get pods -n linkfolio`를 통해 namespace에 리소스가 없음을 확인한 뒤, Master Node에 접속해서 아래 명령어를 수행합니다.
```bash
ubuntu@k8s-master:~$ kubectl delete pvc -n linkfolio --all
ubuntu@k8s-master:~$ kubectl delete pv --all
```

### [3] NFS Server 리소스 제거
Worker Node2에 접속해서 아래 명령어를 수행합니다.
```bash
sudo rm -rf /tmp/k8s-pv/shared-mysql
sudo rm -rf /tmp/k8s-pv/shared-redis
sudo rm -rf /tmp/k8s-pv/chat-mongodb
```

---

## 🛠 5. 트러블 슈팅

### 📌 포트폴리오 모니터링 화면 접속
* `http://localhost:30000/login`

### 📌 Kafka Connector 재시작 필요 시
커넥터 상태 확인 후 필요시 재시작합니다.
```bash
# 커넥터 상태 확인
curl -X GET [http://127.0.1.1:8083/connectors/user-profile-connector/status](http://127.0.1.1:8083/connectors/user-profile-connector/status) | jq
curl -X GET [http://127.0.1.1:8083/connectors/auth-outbox-connector/status](http://127.0.1.1:8083/connectors/auth-outbox-connector/status) | jq

# 커넥터 재시작
curl -X POST [http://127.0.1.1:8083/connectors/auth-outbox-connector/restart](http://127.0.1.1:8083/connectors/auth-outbox-connector/restart)
```

### 📌 ArgoCD 다운 현상 해결 (OOM 방지 패치)
리소스 부족으로 ArgoCD가 다운될 경우 아래 패치를 적용합니다.

**1. ArgoCD Server 패치**
```bash
kubectl patch deployment argo-cd-1758713185-argocd-server -n argocd -p '{"spec":{"template":{"spec":{"containers":[{"name":"server","resources":{"requests":{"cpu":"50m"}},"livenessProbe":{"timeoutSeconds":5},"readinessProbe":{"timeoutSeconds":5}}]}}}}'
```

**2. ArgoCD Repo Server 패치**
```bash
kubectl patch deployment argo-cd-1758713185-argocd-repo-server -n argocd -p '{"spec":{"template":{"spec":{"containers":[{"name":"server","resources":{"requests":{"cpu":"50m"}},"livenessProbe":{"timeoutSeconds":5},"readinessProbe":{"timeoutSeconds":5}}]}}}}'
```
