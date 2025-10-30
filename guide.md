## MongoDB StatefulSet 배포 가이드 (dev)

본 문서는 Kubernetes 상에 MongoDB Replica Set을 구성하기 위한 전체 과정을 설명합니다. 포함 내용: 각 리소스 역할, 필요한 Secret/ConfigMap, 선행 준비사항, 배포 순서, 검증/운영 명령어.

### 아키텍처 개요
- **StatefulSet `mongodb`**: 3개 레플리카로 MongoDB Replica Set 구성. 각 Pod는 영구 스토리지(PVC)를 가짐.
- **Headless Service `mongodb-headless`**: Stable DNS 제공. Pod 간 통신/Replica Set 디스커버리 용도.
- **NodePort Service `mongodb-primary-nodeport`**: 클러스터 외부(노드 IP)에서 PRIMARY(예상: `mongodb-0`)에 접근하기 위한 포트 개방.
- **Job `mongodb-ensure-admin`**: PRIMARY 준비 후 `admin` 사용자 보장(없으면 생성).
- **Secret `scof-dev-secret`(필요)**: 루트 계정 정보와 Replica Set keyFile 포함.
- **ConfigMap `scof-dev-config`(필요)**: Replica Set 이름 제공.
- **StorageClass `scof-dev-sc`(필요)**: PVC 바인딩용 스토리지 클래스.
- **Namespace `scof-dev`(필요)**: 모든 리소스가 배포될 네임스페이스.

### 매니페스트 개요 및 역할
- `k8s_resources/dev/service-headless.yaml`
  - Headless Service (`clusterIP: None`)
  - selector `app: mongodb` → StatefulSet Pod들을 묶어 내부 DNS(`mongodb-0.mongodb-headless.scof-dev.svc.cluster.local`) 제공
- `k8s_resources/dev/statefulset.yaml`
  - StatefulSet `mongodb`, `replicas: 3`, `serviceName: mongodb-headless`
  - `volumeClaimTemplates`로 `data` PVC 생성 (`storageClassName: scof-dev-sc`, 10Gi)
  - `initContainer: fix-key-perms`가 Secret의 keyFile을 `emptyDir`로 복사/권한 설정(UID/GID=999, 0400)
  - 컨테이너 `mongo`는 `mongod --replSet ${REPLICA_SET_NAME} --keyFile /etc/mongodb.key/mongodb.key --bind_ip_all`
  - Env:
    - `MONGO_INITDB_ROOT_USERNAME`, `MONGO_INITDB_ROOT_PASSWORD` → Secret `scof-dev-secret`
    - `REPLICA_SET_NAME` → ConfigMap `scof-dev-config`의 `MONGODB_REPLICA_SET_NAME`
  - Volumes:
    - Secret `scof-dev-secret`의 `MONGODB_KEY` → `/secret/mongodb.key`로 마운트 후 복사
    - `emptyDir`(`keyfile`) → `/etc/mongodb.key`로 마운트하여 mongod keyFile 사용
- `k8s_resources/dev/service-primary.yaml`
  - NodePort Service로 PRIMARY 예상 Pod(`statefulset.kubernetes.io/pod-name: mongodb-0`) 타겟팅
  - 외부 접근: 노드 IP:30002 → 27017
- `k8s_resources/dev/job-ensure-admin.yaml`
  - PRIMARY 준비 대기(`db.hello().isWritablePrimary == true`) 후 `admin` DB에 루트 사용자 생성(없으면 생성)
  - 접속 대상: `mongodb-0.mongodb-headless.scof-dev.svc.cluster.local:27017`

### 선행 준비사항
1. Namespace 존재 확인/생성: `scof-dev`
2. StorageClass 존재 확인: `scof-dev-sc`
3. Secret 준비: `scof-dev-secret`
   - 키들: `MONGO_INITDB_ROOT_USERNAME`, `MONGO_INITDB_ROOT_PASSWORD`, `MONGODB_KEY`
   - `MONGODB_KEY`는 Replica Set 멤버 간 인증용 keyFile 내용(임의 시크릿 문자열, 권장 512비트 이상)
4. ConfigMap 준비: `scof-dev-config`
   - 키: `MONGODB_REPLICA_SET_NAME` (예: `rs0`)

### 리소스 생성 명령어
아래 명령은 macOS/zsh 기준 예시입니다. 필요 시 값 변경.

```bash
# 0) 컨텍스트 확인
kubectl config current-context

# 1) 네임스페이스
kubectl get ns scof-dev || kubectl create ns scof-dev

# 2) StorageClass 확인 (이미 있어야 함)
kubectl get storageclass | grep scof-dev-sc || echo "[경고] StorageClass scof-dev-sc가 필요합니다"

# 3) Secret (루트 계정 + keyFile)
# 방법 A: literal 로 생성
kubectl -n scof-dev create secret generic scof-dev-secret \
  --from-literal=MONGO_INITDB_ROOT_USERNAME=admin \
  --from-literal=MONGO_INITDB_ROOT_PASSWORD='change-me' \
  --from-literal=MONGODB_KEY="$(openssl rand -base64 756)" \
  --dry-run=client -o yaml | kubectl apply -f -

# 방법 B: 파일에서 keyFile 로드 (예: ./keyfile)
# kubectl -n scof-dev create secret generic scof-dev-secret \
#   --from-literal=MONGO_INITDB_ROOT_USERNAME=admin \
#   --from-literal=MONGO_INITDB_ROOT_PASSWORD='change-me' \
#   --from-file=MONGODB_KEY=./keyfile \
#   --dry-run=client -o yaml | kubectl apply -f -

# 4) ConfigMap (Replica Set 이름)
kubectl -n scof-dev create configmap scof-dev-config \
  --from-literal=MONGODB_REPLICA_SET_NAME=rs0 \
  --dry-run=client -o yaml | kubectl apply -f -

# 5) 서비스 및 StatefulSet 적용
kubectl -n scof-dev apply -f k8s_resources/dev/service-headless.yaml
kubectl -n scof-dev apply -f k8s_resources/dev/statefulset.yaml

# 6) Pod 생성 확인
kubectl -n scof-dev get pods -l app=mongodb -w
```

### Replica Set 초기화(필요 시)
일부 환경에서는 최초 PRIMARY 선출 및 멤버 설정을 수동으로 초기화해야 할 수 있습니다. 아래는 `mongodb-0`에 접속해 rs.initiate 수행 예시입니다.

```bash
# mongodb-0에 접속
kubectl -n scof-dev exec -it statefulset/mongodb -c mongo -- bash -lc "mongosh --host mongodb-0.mongodb-headless.scof-dev.svc.cluster.local:27017"
```

```javascript
// mongosh 내부에서 수행 (예시: rs0, 멤버 3개)
rs.initiate({
  _id: "rs0",
  members: [
    { _id: 0, host: "mongodb-0.mongodb-headless.scof-dev.svc.cluster.local:27017" },
    { _id: 1, host: "mongodb-1.mongodb-headless.scof-dev.svc.cluster.local:27017" },
    { _id: 2, host: "mongodb-2.mongodb-headless.scof-dev.svc.cluster.local:27017" }
  ]
})

rs.status()
```

참고: 클러스터가 자동 선출을 완료하면 `job-ensure-admin`이 PRIMARY 준비를 폴링한 뒤 admin 생성 로직을 실행합니다.

### admin 사용자 보장 Job 실행
매니페스트 적용으로 Job은 필요 시 수동 적용 가능합니다.

```bash
kubectl -n scof-dev apply -f k8s_resources/dev/job-ensure-admin.yaml
kubectl -n scof-dev logs job/mongodb-ensure-admin -f
```

### 외부 접근(개발 편의)
- NodePort Service 사용: 노드 IP의 30002 포트 → PRIMARY(`mongodb-0`) 27017로 전달

```bash
# 노드 IP 확인
kubectl get nodes -o wide
# 예) mongodb 쉘로 외부에서 접속
mongosh "mongodb://<NODE_IP>:30002/admin" -u "$MONGO_INITDB_ROOT_USERNAME" -p "$MONGO_INITDB_ROOT_PASSWORD"
```

개발 환경에서만 사용 권장. 보안/네트워크 정책에 맞춰 Ingress/LoadBalancer/Port-Forward 등으로 대체 가능.

### 점검 및 트러블슈팅 명령어
```bash
# 리소스 조회
kubectl -n scof-dev get all -l app=mongodb
kubectl -n scof-dev get sts mongodb -o yaml | less
kubectl -n scof-dev get svc mongodb-headless mongodb-primary-nodeport

# Pod 상태/로그
kubectl -n scof-dev describe pod -l app=mongodb
kubectl -n scof-dev logs -l app=mongodb -c mongo --tail=200
kubectl -n scof-dev logs -l app=mongodb -c fix-key-perms --tail=200

# PVC/PV 확인
kubectl -n scof-dev get pvc
kubectl get pv | grep scof-dev

# Secret/ConfigMap 확인
kubectl -n scof-dev get secret scof-dev-secret -o yaml | kubectl neat || kubectl -n scof-dev get secret scof-dev-secret -o yaml | less
kubectl -n scof-dev get configmap scof-dev-config -o yaml

# Replica Set 상태 점검
kubectl -n scof-dev exec -it statefulset/mongodb -c mongo -- bash -lc "mongosh --eval 'rs.status()'"
```

### 보안 및 운영 고려사항
- keyFile(`MONGODB_KEY`)은 외부 유출 금지. Secret 접근 권한 최소화.
- NodePort는 개발 용도. 운영은 내부 서비스/포워딩/전용 게이트웨이 사용 고려.
- 백업/복구 전략 수립(스냅샷/옵스툴/백업 에이전트 등).
- 모니터링 도입(Prometheus Exporter, 로그 수집 등).
- 리소스 제한/요청 설정, PodDisruptionBudget, PodAntiAffinity 등 가용성 정책 검토.

### 전체 배포 순서 요약
1) Namespace/StorageClass 확인 → 2) Secret/ConfigMap 생성 → 3) Headless Service 적용 → 4) StatefulSet 적용 → 5) Replica Set 초기화(필요 시) → 6) ensure-admin Job 실행/확인 → 7) NodePort로 외부 접속(선택)

### 부록: 파일 경로
- `k8s_resources/dev/statefulset.yaml`
- `k8s_resources/dev/service-headless.yaml`
- `k8s_resources/dev/service-primary.yaml`
- `k8s_resources/dev/job-ensure-admin.yaml`

