# OpenMetadata 온프레미스 배포 가이드

> **대상 환경**: 전사 규모 (500+ 사용자, 50+ 데이터 소스)
> **기술 스택**: PostgreSQL + Elasticsearch + OIDC(Keycloak) + Kubernetes
> **데이터 소스**: RDB (MySQL, PostgreSQL, Oracle 등)

---

## 목차

1. [아키텍처 개요](#1-아키텍처-개요)
2. [물리적 서버 구성](#2-물리적-서버-구성)
3. [사전 준비](#3-사전-준비)
4. [배포 방법 A: Docker Compose (검증/스테이징용)](#4-배포-방법-a-docker-compose-검증스테이징용)
5. [배포 방법 B: Kubernetes + Helm (운영용)](#5-배포-방법-b-kubernetes--helm-운영용)
6. [OIDC 인증 설정 (Keycloak)](#6-oidc-인증-설정-keycloak)
7. [데이터 소스 연결 (RDB)](#7-데이터-소스-연결-rdb)
8. [운영 및 모니터링](#8-운영-및-모니터링)
9. [백업 및 복구](#9-백업-및-복구)
10. [트러블슈팅](#10-트러블슈팅)
11. [Kubernetes 핵심 개념 요약](#11-kubernetes-핵심-개념-요약)

---

## 1. 아키텍처 개요

### 1.1 시스템 구성도

```
                        ┌─────────────────────┐
                        │   Load Balancer /    │
                        │   Reverse Proxy      │
                        │   (Nginx/HAProxy)    │
                        │   Port: 443 (HTTPS)  │
                        └──────────┬──────────┘
                                   │
                    ┌──────────────┼──────────────┐
                    │              │              │
           ┌────────▼───────┐     │     ┌────────▼───────┐
           │  OpenMetadata  │     │     │  OpenMetadata  │
           │  Server #1     │     │     │  Server #2     │
           │  Port: 8585    │     │     │  Port: 8585    │
           │  (API + UI)    │     │     │  (API + UI)    │
           └────────┬───────┘     │     └────────┬───────┘
                    │              │              │
        ┌───────────┴──────────────┴──────────────┘
        │
   ┌────▼─────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
   │PostgreSQL │  │Elasticsearch │  │   Airflow     │  │  Keycloak    │
   │  (Primary │  │  Cluster     │  │  (Ingestion)  │  │  (SSO/OIDC)  │
   │  +Standby)│  │  (3 nodes)   │  │  Port: 8080   │  │  Port: 8443  │
   │  Port:5432│  │  Port: 9200  │  │               │  │              │
   └───────────┘  └──────────────┘  └──────────────┘  └──────────────┘
```

### 1.2 필수 구성 요소

| 구성 요소 | 역할 | 포트 |
|----------|------|------|
| **OpenMetadata Server** | 메타데이터 API + 웹 UI | 8585 (HTTP), 8586 (Admin) |
| **PostgreSQL** | 메타데이터 저장소 | 5432 |
| **Elasticsearch** | 검색 인덱스 | 9200, 9300 |
| **Airflow (Ingestion)** | 메타데이터 수집 파이프라인 오케스트레이션 | 8080 |
| **Keycloak** | OIDC 인증 서버 (SSO) | 8443 |

---

## 2. 물리적 서버 구성

### 2.1 권장 서버 사양 (전사 규모, 500+ 사용자)

#### 옵션 A: 분리 배포 (권장)

| 서버 역할 | CPU | RAM | 디스크 | 수량 | 비고 |
|----------|-----|-----|--------|------|------|
| **OpenMetadata Server** | 8 cores | 16 GB | 100 GB SSD | 2대 | HA 구성, LB 뒤에 배치 |
| **PostgreSQL** | 8 cores | 32 GB | 500 GB SSD | 2대 | Primary + Standby (Streaming Replication) |
| **Elasticsearch** | 8 cores | 32 GB | 500 GB SSD | 3대 | 클러스터 구성 (Master-eligible 3대) |
| **Airflow (Ingestion)** | 4 cores | 8 GB | 100 GB SSD | 1대 | 수집 워커 |
| **Load Balancer** | 2 cores | 4 GB | 50 GB | 1대 | Nginx 또는 HAProxy |
| **Keycloak (SSO)** | 4 cores | 8 GB | 50 GB | 1대 | 기존 SSO 서버 활용 가능 |
| **합계** | | | | **10대** | |

#### 옵션 B: 통합 배포 (최소 구성)

| 서버 역할 | CPU | RAM | 디스크 | 수량 | 비고 |
|----------|-----|-----|--------|------|------|
| **App 서버** (OM + Airflow) | 16 cores | 32 GB | 200 GB SSD | 2대 | Docker Compose 또는 K8s Worker |
| **DB 서버** (PostgreSQL + ES) | 16 cores | 64 GB | 1 TB SSD | 2대 | 데이터 계층 |
| **합계** | | | | **4대** | |

#### 옵션 C: Kubernetes 클러스터 활용 (기존 K8s 운영 중인 경우)

| 구성 요소 | 리소스 요청 | 리소스 제한 | 레플리카 | 비고 |
|----------|-----------|-----------|---------|------|
| **OpenMetadata Server** | 1 CPU / 2 Gi | 4 CPU / 8 Gi | 2 | Deployment |
| **PostgreSQL** | 2 CPU / 4 Gi | 4 CPU / 16 Gi | 1 (+ Standby) | StatefulSet 또는 외부 DB |
| **Elasticsearch** | 2 CPU / 4 Gi | 4 CPU / 16 Gi | 3 | StatefulSet |
| **Airflow** | 500m CPU / 1 Gi | 2 CPU / 4 Gi | 1 | Deployment |
| **Ingestion Job** | 500m CPU / 1 Gi | 2 CPU / 4 Gi | - | K8s Job (온디맨드) |

> **참고**: PostgreSQL과 Elasticsearch는 K8s 내부보다 외부 전용 서버에 배치하는 것이 운영 안정성 면에서 유리합니다.

### 2.2 네트워크 요구사항

```
┌─────────────────────────────────────────────────────────────────┐
│                    사내 네트워크                                  │
│                                                                 │
│  ┌──────────────┐     ┌──────────────┐     ┌──────────────┐     │
│  │ 사용자 PC     │────▶│ Load Balancer│────▶│ OpenMetadata │     │
│  │              │     │ :443 (HTTPS) │     │ :8585        │     │
│  └──────────────┘     └──────────────┘     └──────┬───────┘     │
│                                                    │             │
│  ┌──────────────┐                                  │             │
│  │ 데이터 소스   │◀──── Airflow Ingestion ◀─────────┘             │
│  │ Oracle :1521 │     (메타데이터 수집)                            │
│  │ MySQL  :3306 │                                                │
│  │ PG     :5432 │                                                │
│  └──────────────┘                                                │
└─────────────────────────────────────────────────────────────────┘
```

**필수 방화벽 규칙**:

| From | To | Port | Protocol | 용도 |
|------|-----|------|----------|------|
| 사용자 PC | Load Balancer | 443 | TCP | 웹 UI 접근 |
| Load Balancer | OpenMetadata Server | 8585 | TCP | API 프록시 |
| OpenMetadata Server | PostgreSQL | 5432 | TCP | DB 접근 |
| OpenMetadata Server | Elasticsearch | 9200 | TCP | 검색 인덱스 |
| OpenMetadata Server | Airflow | 8080 | TCP | 파이프라인 관리 |
| OpenMetadata Server | Keycloak | 8443 | TCP | OIDC 인증 |
| Airflow | 데이터 소스 | 각 포트 | TCP | 메타데이터 수집 |
| Airflow | OpenMetadata Server | 8585 | TCP | API 호출 |
| Airflow | PostgreSQL | 5432 | TCP | Airflow 메타DB |

---

## 3. 사전 준비

### 3.1 필수 소프트웨어

```bash
# Docker & Docker Compose (모든 서버)
docker --version        # 20.10+ 필요
docker compose version  # v2.0+ 필요

# Kubernetes (K8s 배포 시)
kubectl version         # 1.25+ 권장
helm version            # 3.10+ 필요

# Java (소스 빌드 시)
java --version          # Java 21+ 필요
mvn --version           # Maven 3.8+ 필요
```

### 3.2 소스 코드 빌드 (선택)

공식 Docker 이미지를 사용하면 빌드가 필요 없습니다. 소스를 커스터마이징한 경우에만 빌드합니다.

```bash
# 1. 소스 클론
git clone https://github.com/open-metadata/OpenMetadata.git
cd OpenMetadata

# 2. 백엔드 빌드 (UI 포함)
mvn clean package -DskipTests

# 3. 백엔드 빌드 (UI 제외, 빠른 빌드)
mvn clean package -DonlyBackend -pl !openmetadata-ui -DskipTests

# 4. 빌드 결과 확인
ls openmetadata-dist/target/openmetadata-*.tar.gz

# 5. Docker 이미지 빌드
docker build -t my-company/openmetadata-server:1.12.0 \
  -f docker/development/Dockerfile .
```

### 3.3 PostgreSQL 초기화 스크립트

아래 SQL로 OpenMetadata + Airflow 데이터베이스를 생성합니다.

```sql
-- PostgreSQL에 접속하여 실행
CREATE DATABASE openmetadata_db;
CREATE DATABASE airflow_db;

CREATE USER openmetadata_user WITH PASSWORD '<강력한_비밀번호_입력>';
CREATE USER airflow_user WITH PASSWORD '<강력한_비밀번호_입력>';

ALTER DATABASE openmetadata_db OWNER TO openmetadata_user;
ALTER DATABASE airflow_db OWNER TO airflow_user;
ALTER USER airflow_user SET search_path = public;
```

---

## 4. 배포 방법 A: Docker Compose (검증/스테이징용)

Docker Compose는 **운영 전 검증 단계** 또는 **소규모 운영**에 적합합니다. 전사 운영에는 방법 B(Kubernetes)를 권장합니다.

### 4.1 환경 변수 파일 작성

```bash
mkdir -p /opt/openmetadata
cd /opt/openmetadata
```

`.env` 파일 생성:

```bash
cat > .env << 'EOF'
# === PostgreSQL ===
DB_DRIVER_CLASS=org.postgresql.Driver
DB_SCHEME=postgresql
DB_HOST=postgresql
DB_PORT=5432
DB_USER=openmetadata_user
DB_USER_PASSWORD=<강력한_비밀번호_입력>
OM_DATABASE=openmetadata_db
DB_PARAMS=allowPublicKeyRetrieval=true&useSSL=false&serverTimezone=UTC

# === Elasticsearch ===
ELASTICSEARCH_HOST=elasticsearch
ELASTICSEARCH_PORT=9200
ELASTICSEARCH_SCHEME=http
SEARCH_TYPE=elasticsearch

# === 서버 설정 ===
SERVER_PORT=8585
SERVER_ADMIN_PORT=8586
LOG_LEVEL=INFO
OPENMETADATA_HEAP_OPTS=-Xmx4G -Xms4G

# === OIDC 인증 (Keycloak 예시) ===
AUTHENTICATION_PROVIDER=custom-oidc
AUTHENTICATION_CLIENT_TYPE=confidential
CUSTOM_OIDC_AUTHENTICATION_PROVIDER_NAME=Keycloak
AUTHENTICATION_PUBLIC_KEYS=[http://keycloak.company.com:8443/realms/openmetadata/protocol/openid-connect/certs,http://localhost:8585/api/v1/system/config/jwks]
AUTHENTICATION_AUTHORITY=http://keycloak.company.com:8443/realms/openmetadata
AUTHENTICATION_CLIENT_ID=openmetadata
AUTHENTICATION_CALLBACK_URL=http://openmetadata.company.com/callback
AUTHENTICATION_JWT_PRINCIPAL_CLAIMS=[email,preferred_username,sub]
AUTHENTICATION_ENABLE_SELF_SIGNUP=true

OIDC_CLIENT_ID=openmetadata
OIDC_CLIENT_SECRET=<Keycloak_Client_Secret>
OIDC_TYPE=custom
OIDC_DISCOVERY_URI=http://keycloak.company.com:8443/realms/openmetadata/.well-known/openid-configuration
OIDC_CALLBACK=http://openmetadata.company.com/callback
OIDC_SERVER_URL=http://openmetadata.company.com
OIDC_CLIENT_AUTH_METHOD=client_secret_post
OIDC_SCOPE=openid email profile

# === Authorizer ===
AUTHORIZER_ADMIN_PRINCIPALS=[admin]
AUTHORIZER_PRINCIPAL_DOMAIN=company.com
AUTHORIZER_ALLOWED_REGISTRATION_DOMAIN=["company.com"]
AUTHORIZER_ENFORCE_PRINCIPAL_DOMAIN=true

# === Airflow ===
AIRFLOW_USERNAME=admin
AIRFLOW_PASSWORD=<Airflow_비밀번호>
PIPELINE_SERVICE_CLIENT_ENDPOINT=http://ingestion:8080
SERVER_HOST_API_URL=http://openmetadata-server:8585/api

# === JWT ===
RSA_PUBLIC_KEY_FILE_PATH=./conf/public_key.der
RSA_PRIVATE_KEY_FILE_PATH=./conf/private_key.der
JWT_ISSUER=open-metadata.org
JWT_KEY_ID=Gb389a-9f76-gdjs-a92j-0242bk94356
EOF
```

### 4.2 Docker Compose 파일

프로젝트에 포함된 `docker/docker-compose-quickstart/docker-compose-postgres.yml`을 기반으로 합니다.

```bash
# 기본 제공 파일로 바로 실행 (검증용)
cd /path/to/OpenMetadata
docker compose -f docker/docker-compose-quickstart/docker-compose-postgres.yml \
  --env-file /opt/openmetadata/.env \
  up -d
```

또는 커스텀 compose 파일을 작성합니다:

```yaml
# /opt/openmetadata/docker-compose.yml
version: "3.9"

volumes:
  pg-data:
  es-data:
  ingestion-dag-airflow:
  ingestion-dags:
  ingestion-tmp:

services:
  # ──────────────────────────────────────────────
  # PostgreSQL
  # ──────────────────────────────────────────────
  postgresql:
    image: postgres:15
    restart: always
    command: >
      -c work_mem=10MB
      -c max_connections=200
      -c shared_buffers=2GB
      -c effective_cache_size=6GB
      -c maintenance_work_mem=512MB
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: <postgres_root_비밀번호>
    ports:
      - "5432:5432"
    volumes:
      - pg-data:/var/lib/postgresql/data
      - ./init-db.sql:/docker-entrypoint-initdb.d/init.sql
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 15s
      timeout: 10s
      retries: 10

  # ──────────────────────────────────────────────
  # Elasticsearch
  # ──────────────────────────────────────────────
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.11.0
    restart: always
    environment:
      - discovery.type=single-node
      - ES_JAVA_OPTS=-Xms2g -Xmx2g
      - xpack.security.enabled=false
    ports:
      - "9200:9200"
      - "9300:9300"
    volumes:
      - es-data:/usr/share/elasticsearch/data
    healthcheck:
      test: >
        curl -s http://localhost:9200/_cluster/health?pretty
        | grep status | grep -qE 'green|yellow' || exit 1
      interval: 15s
      timeout: 10s
      retries: 10

  # ──────────────────────────────────────────────
  # DB 마이그레이션 (1회 실행)
  # ──────────────────────────────────────────────
  execute-migrate-all:
    image: docker.getcollate.io/openmetadata/server:1.6.1
    command: "./bootstrap/openmetadata-ops.sh migrate"
    env_file: .env
    depends_on:
      elasticsearch:
        condition: service_healthy
      postgresql:
        condition: service_healthy

  # ──────────────────────────────────────────────
  # OpenMetadata Server
  # ──────────────────────────────────────────────
  openmetadata-server:
    image: docker.getcollate.io/openmetadata/server:1.6.1
    restart: always
    env_file: .env
    ports:
      - "8585:8585"
      - "8586:8586"
    depends_on:
      elasticsearch:
        condition: service_healthy
      postgresql:
        condition: service_healthy
      execute-migrate-all:
        condition: service_completed_successfully
    healthcheck:
      test: ["CMD", "wget", "-q", "--spider", "http://localhost:8586/healthcheck"]
      interval: 30s
      timeout: 10s
      retries: 5

  # ──────────────────────────────────────────────
  # Airflow (Ingestion)
  # ──────────────────────────────────────────────
  ingestion:
    image: docker.getcollate.io/openmetadata/ingestion:1.6.1
    restart: always
    environment:
      AIRFLOW__API__AUTH_BACKENDS: "airflow.api.auth.backend.basic_auth,airflow.api.auth.backend.session"
      AIRFLOW__CORE__EXECUTOR: LocalExecutor
      AIRFLOW__OPENMETADATA_AIRFLOW_APIS__DAG_GENERATED_CONFIGS: "/opt/airflow/dag_generated_configs"
      DB_HOST: postgresql
      DB_PORT: 5432
      AIRFLOW_DB: airflow_db
      DB_USER: airflow_user
      DB_SCHEME: postgresql+psycopg2
      DB_PASSWORD: <Airflow_DB_비밀번호>
    entrypoint: /bin/bash
    command: ["/opt/airflow/ingestion_dependency.sh"]
    ports:
      - "8080:8080"
    volumes:
      - ingestion-dag-airflow:/opt/airflow/dag_generated_configs
      - ingestion-dags:/opt/airflow/dags
      - ingestion-tmp:/tmp
    depends_on:
      postgresql:
        condition: service_healthy
      openmetadata-server:
        condition: service_started
```

DB 초기화 SQL 파일:

```bash
# /opt/openmetadata/init-db.sql
cat > init-db.sql << 'EOF'
CREATE DATABASE openmetadata_db;
CREATE DATABASE airflow_db;
CREATE USER openmetadata_user WITH PASSWORD '<강력한_비밀번호_입력>';
CREATE USER airflow_user WITH PASSWORD '<Airflow_DB_비밀번호>';
ALTER DATABASE openmetadata_db OWNER TO openmetadata_user;
ALTER DATABASE airflow_db OWNER TO airflow_user;
ALTER USER airflow_user SET search_path = public;
EOF
```

### 4.3 실행 및 확인

```bash
cd /opt/openmetadata

# 1. 실행
docker compose up -d

# 2. 상태 확인
docker compose ps

# 3. 로그 확인
docker compose logs -f openmetadata-server

# 4. 헬스체크
curl http://localhost:8586/healthcheck
curl http://localhost:8585/api/v1/system/health

# 5. Elasticsearch 상태 확인
curl http://localhost:9200/_cluster/health?pretty

# 6. 웹 UI 접속
# 브라우저에서 http://<서버IP>:8585 접속
# 기본 계정: admin / admin (OIDC 설정 전)

# 7. 중지
docker compose down

# 8. 데이터 포함 완전 삭제
docker compose down -v
```

---

## 5. 배포 방법 B: Kubernetes + Helm (운영용)

> **Kubernetes를 처음 접하는 분을 위해**: 11장에 K8s 핵심 개념 요약이 있습니다.

### 5.1 Helm Chart 설치

OpenMetadata는 공식 Helm Chart를 별도 저장소에서 관리합니다.

```bash
# 1. Helm 저장소 추가
helm repo add openmetadata https://helm.open-metadata.org
helm repo update

# 2. 차트 확인
helm search repo openmetadata

# 3. 기본 values 파일 다운로드 (커스터마이징 기준)
helm show values openmetadata/openmetadata > values-custom.yaml
```

### 5.2 Namespace 및 Secret 생성

```bash
# 1. Namespace 생성
kubectl create namespace openmetadata

# 2. PostgreSQL 시크릿
kubectl create secret generic postgres-secrets \
  --namespace openmetadata \
  --from-literal=openmetadata-postgres-password='<강력한_비밀번호_입력>'

# 3. Airflow 시크릿
kubectl create secret generic airflow-secrets \
  --namespace openmetadata \
  --from-literal=openmetadata-airflow-password='<Airflow_비밀번호>'

# 4. OIDC 시크릿
kubectl create secret generic oidc-secrets \
  --namespace openmetadata \
  --from-literal=oidc-client-secret='<Keycloak_Client_Secret>'

# 시크릿 확인
kubectl get secrets -n openmetadata
```

### 5.3 Helm Values 파일 (운영용)

```yaml
# /opt/openmetadata/values-production.yaml

# === OpenMetadata Server 설정 ===
openmetadata:
  config:
    logLevel: INFO

    # PostgreSQL 설정
    database:
      host: postgresql.openmetadata.svc.cluster.local  # K8s 내부 서비스 또는 외부 DB IP
      port: 5432
      driverClass: org.postgresql.Driver
      dbScheme: postgresql
      auth:
        username: openmetadata_user
        password:
          secretRef: postgres-secrets
          secretKey: openmetadata-postgres-password

    # Elasticsearch 설정
    elasticsearch:
      host: elasticsearch.openmetadata.svc.cluster.local  # K8s 내부 서비스 또는 외부 ES IP
      port: 9200
      searchType: elasticsearch
      scheme: http

    # OIDC 인증 (Keycloak)
    authentication:
      provider: custom-oidc
      clientType: confidential
      publicKeys:
        - http://keycloak.company.com:8443/realms/openmetadata/protocol/openid-connect/certs
        - http://openmetadata.company.com/api/v1/system/config/jwks
      authority: http://keycloak.company.com:8443/realms/openmetadata
      clientId: openmetadata
      callbackUrl: https://openmetadata.company.com/callback
      jwtPrincipalClaims:
        - email
        - preferred_username
        - sub
      enableSelfSignup: true

    authorizer:
      className: org.openmetadata.service.security.DefaultAuthorizer
      containerRequestFilter: org.openmetadata.service.security.JwtFilter
      adminPrincipals:
        - admin
      principalDomain: company.com
      allowedEmailRegistrationDomains:
        - company.com
      enforcePrincipalDomain: true

    # Airflow Pipeline Client
    pipelineServiceClientConfig:
      type: airflow
      apiEndpoint: http://openmetadata-dependencies-api-server:8080
      metadataApiEndpoint: http://openmetadata:8585/api
      airflow:
        username: admin
        password:
          secretRef: airflow-secrets
          secretKey: openmetadata-airflow-password

    # JVM 힙 설정
    openmetadataHeapOpts: "-Xmx4G -Xms4G"

# === Pod 리소스 ===
resources:
  requests:
    cpu: "2"
    memory: 4Gi
  limits:
    cpu: "4"
    memory: 8Gi

# === 레플리카 ===
replicaCount: 2

# === Ingress 설정 ===
ingress:
  enabled: true
  className: nginx
  annotations:
    nginx.ingress.kubernetes.io/proxy-body-size: "50m"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "600"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "600"
    cert-manager.io/cluster-issuer: letsencrypt-prod  # TLS 인증서 (선택)
  hosts:
    - host: openmetadata.company.com
      paths:
        - path: /
          pathType: Prefix
  tls:
    - secretName: openmetadata-tls
      hosts:
        - openmetadata.company.com

# === Health Check ===
livenessProbe:
  initialDelaySeconds: 120
  periodSeconds: 30
readinessProbe:
  initialDelaySeconds: 120
  periodSeconds: 30
startupProbe:
  failureThreshold: 15
  periodSeconds: 10

# === Service ===
service:
  type: ClusterIP
  port: 8585
```

### 5.4 배포 실행

```bash
# 1. (선택) 외부 DB/ES를 사용하지 않는 경우 — 의존 서비스 먼저 설치
helm install openmetadata-dependencies openmetadata/openmetadata-dependencies \
  --namespace openmetadata \
  --values values-production.yaml

# 2. OpenMetadata 서버 설치
helm install openmetadata openmetadata/openmetadata \
  --namespace openmetadata \
  --values values-production.yaml

# 3. 배포 상태 확인
kubectl get pods -n openmetadata -w

# 4. 서비스 확인
kubectl get svc -n openmetadata

# 5. 로그 확인
kubectl logs -f deployment/openmetadata -n openmetadata

# 6. 헬스체크
kubectl port-forward svc/openmetadata 8585:8585 -n openmetadata &
curl http://localhost:8585/api/v1/system/health
```

### 5.5 업그레이드

```bash
# Helm 차트 업데이트
helm repo update

# 업그레이드 (values 파일 유지)
helm upgrade openmetadata openmetadata/openmetadata \
  --namespace openmetadata \
  --values values-production.yaml

# 롤백 (문제 발생 시)
helm rollback openmetadata 1 -n openmetadata

# 배포 이력 확인
helm history openmetadata -n openmetadata
```

---

## 6. OIDC 인증 설정 (Keycloak)

### 6.1 Keycloak에서 클라이언트 생성

```
1. Keycloak 관리 콘솔 접속
2. Realm 생성: openmetadata
3. Client 생성:
   - Client ID: openmetadata
   - Client Protocol: openid-connect
   - Access Type: confidential
   - Valid Redirect URIs: https://openmetadata.company.com/callback
   - Web Origins: https://openmetadata.company.com
4. Client Secret 복사 → 환경 변수에 설정
```

### 6.2 환경 변수 설정 (Docker Compose)

```bash
# .env 파일에 아래 값 설정
AUTHENTICATION_PROVIDER=custom-oidc
AUTHENTICATION_CLIENT_TYPE=confidential
CUSTOM_OIDC_AUTHENTICATION_PROVIDER_NAME=Keycloak

# Keycloak의 JWKS 엔드포인트 + OpenMetadata 자체 JWKS
AUTHENTICATION_PUBLIC_KEYS=[http://keycloak.company.com:8443/realms/openmetadata/protocol/openid-connect/certs,http://localhost:8585/api/v1/system/config/jwks]

AUTHENTICATION_AUTHORITY=http://keycloak.company.com:8443/realms/openmetadata
AUTHENTICATION_CLIENT_ID=openmetadata
AUTHENTICATION_CALLBACK_URL=https://openmetadata.company.com/callback

# OIDC 상세 설정
OIDC_CLIENT_ID=openmetadata
OIDC_CLIENT_SECRET=<Keycloak에서_복사한_Client_Secret>
OIDC_TYPE=custom
OIDC_DISCOVERY_URI=http://keycloak.company.com:8443/realms/openmetadata/.well-known/openid-configuration
OIDC_CALLBACK=https://openmetadata.company.com/callback
OIDC_SERVER_URL=https://openmetadata.company.com
OIDC_CLIENT_AUTH_METHOD=client_secret_post
OIDC_SCOPE=openid email profile
```

### 6.3 인증 흐름 검증

```bash
# 1. OpenMetadata 서버 재시작
docker compose restart openmetadata-server

# 2. 브라우저에서 접속 → Keycloak 로그인 페이지로 리다이렉트 확인
# https://openmetadata.company.com

# 3. 로그인 후 콜백 URL로 돌아오는지 확인
# https://openmetadata.company.com/callback

# 4. API로 현재 사용자 확인
curl -H "Authorization: Bearer <JWT_TOKEN>" \
  http://localhost:8585/api/v1/users/loggedInUser
```

---

## 7. 데이터 소스 연결 (RDB)

### 7.1 웹 UI에서 서비스 추가

```
1. 로그인 → Settings → Services → Databases → Add New Service
2. 서비스 타입 선택 (MySQL, PostgreSQL, Oracle 등)
3. 연결 정보 입력
4. Test Connection 클릭
5. 성공 시 → Save
```

### 7.2 API로 서비스 추가 (자동화)

#### MySQL 데이터 소스 등록

```bash
# JWT 토큰 발급 (Bot 계정 사용)
export TOKEN=$(curl -s -X POST http://localhost:8585/api/v1/users/login \
  -H "Content-Type: application/json" \
  -d '{"email":"admin@company.com","password":"admin"}' | jq -r '.accessToken')

# MySQL 서비스 등록
curl -X POST http://localhost:8585/api/v1/services/databaseServices \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "production-mysql",
    "serviceType": "Mysql",
    "connection": {
      "config": {
        "type": "Mysql",
        "scheme": "mysql+pymysql",
        "hostPort": "mysql-server.company.com:3306",
        "username": "readonly_user",
        "authType": {
          "password": "<DB_비밀번호>"
        }
      }
    }
  }'
```

#### PostgreSQL 데이터 소스 등록

```bash
curl -X POST http://localhost:8585/api/v1/services/databaseServices \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "production-postgres",
    "serviceType": "Postgres",
    "connection": {
      "config": {
        "type": "Postgres",
        "scheme": "postgresql+psycopg2",
        "hostPort": "postgres-server.company.com:5432",
        "username": "readonly_user",
        "authType": {
          "password": "<DB_비밀번호>"
        },
        "database": "production_db"
      }
    }
  }'
```

#### Oracle 데이터 소스 등록

```bash
curl -X POST http://localhost:8585/api/v1/services/databaseServices \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "production-oracle",
    "serviceType": "Oracle",
    "connection": {
      "config": {
        "type": "Oracle",
        "hostPort": "oracle-server.company.com:1521",
        "username": "readonly_user",
        "authType": {
          "password": "<DB_비밀번호>"
        },
        "oracleConnectionType": {
          "oracleServiceName": "ORCL"
        }
      }
    }
  }'
```

### 7.3 메타데이터 수집 파이프라인 생성

```bash
# Ingestion Pipeline 생성 (MySQL 예시)
curl -X POST http://localhost:8585/api/v1/services/ingestionPipelines \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "production-mysql-metadata",
    "pipelineType": "metadata",
    "service": {
      "type": "databaseService",
      "id": "<위에서_생성한_서비스_ID>"
    },
    "sourceConfig": {
      "config": {
        "type": "DatabaseMetadata",
        "markDeletedTables": true,
        "includeTables": true,
        "includeViews": true
      }
    },
    "airflowConfig": {
      "scheduleInterval": "0 2 * * *"
    }
  }'
```

> **스케줄**: `0 2 * * *` = 매일 새벽 2시에 메타데이터 수집 실행

---

## 8. 운영 및 모니터링

### 8.1 헬스체크 엔드포인트

```bash
# OpenMetadata 서버 상태
curl http://localhost:8586/healthcheck

# 시스템 헬스 API
curl http://localhost:8585/api/v1/system/health

# Elasticsearch 클러스터 상태
curl http://localhost:9200/_cluster/health?pretty

# PostgreSQL 연결 확인
pg_isready -h localhost -p 5432 -U openmetadata_user

# Airflow 상태
curl http://localhost:8080/api/v1/health
```

### 8.2 모니터링 스크립트

```bash
#!/bin/bash
# /opt/openmetadata/healthcheck.sh

check_service() {
  local name=$1
  local url=$2
  local status=$(curl -s -o /dev/null -w "%{http_code}" "$url" 2>/dev/null)
  if [ "$status" = "200" ]; then
    echo "[OK] $name"
  else
    echo "[FAIL] $name (HTTP $status)"
    # 여기에 알림 로직 추가 (Slack, Email 등)
  fi
}

echo "=== OpenMetadata Health Check $(date) ==="
check_service "OpenMetadata Server" "http://localhost:8586/healthcheck"
check_service "Elasticsearch"      "http://localhost:9200/_cluster/health"
check_service "Airflow"            "http://localhost:8080/api/v1/health"

# PostgreSQL
if pg_isready -h localhost -p 5432 -U openmetadata_user -q; then
  echo "[OK] PostgreSQL"
else
  echo "[FAIL] PostgreSQL"
fi
```

```bash
# crontab에 등록 (5분마다 실행)
chmod +x /opt/openmetadata/healthcheck.sh
crontab -e
# 추가: */5 * * * * /opt/openmetadata/healthcheck.sh >> /var/log/om-healthcheck.log 2>&1
```

### 8.3 로그 관리

```bash
# Docker Compose 로그
docker compose logs --tail=100 openmetadata-server
docker compose logs --tail=100 ingestion
docker compose logs --tail=100 postgresql
docker compose logs --tail=100 elasticsearch

# Kubernetes 로그
kubectl logs -f deployment/openmetadata -n openmetadata --tail=100
kubectl logs -f deployment/openmetadata -n openmetadata -p  # 이전 Pod 로그

# 서버 내부 로그 파일 (컨테이너 내)
docker exec openmetadata_server cat /opt/openmetadata/logs/openmetadata.log
docker exec openmetadata_server cat /opt/openmetadata/logs/audit.log
```

### 8.4 Prometheus 메트릭 (선택)

```bash
# openmetadata.yaml 또는 환경 변수에서 설정
EVENT_MONITOR=prometheus
EVENT_MONITOR_BATCH_SIZE=10
EVENT_MONITOR_PATH_PATTERN=["/api/v1/tables/*", "/api/v1/health-check"]
```

---

## 9. 백업 및 복구

### 9.1 PostgreSQL 백업

```bash
#!/bin/bash
# /opt/openmetadata/backup.sh

BACKUP_DIR=/opt/openmetadata/backups
DATE=$(date +%Y%m%d_%H%M%S)
mkdir -p $BACKUP_DIR

# OpenMetadata DB 백업
pg_dump -h localhost -U openmetadata_user -d openmetadata_db \
  -F c -f "$BACKUP_DIR/openmetadata_db_$DATE.dump"

# Airflow DB 백업
pg_dump -h localhost -U airflow_user -d airflow_db \
  -F c -f "$BACKUP_DIR/airflow_db_$DATE.dump"

# 30일 이상 된 백업 삭제
find $BACKUP_DIR -name "*.dump" -mtime +30 -delete

echo "Backup completed: $DATE"
```

```bash
# crontab 등록 (매일 새벽 1시)
chmod +x /opt/openmetadata/backup.sh
crontab -e
# 추가: 0 1 * * * /opt/openmetadata/backup.sh >> /var/log/om-backup.log 2>&1
```

### 9.2 PostgreSQL 복구

```bash
# 복구 (주의: 기존 데이터 덮어씀)
pg_restore -h localhost -U openmetadata_user -d openmetadata_db \
  --clean --if-exists /opt/openmetadata/backups/openmetadata_db_20260309.dump
```

### 9.3 Elasticsearch 인덱스 재생성

DB 복구 후 검색 인덱스가 동기화되지 않은 경우:

```bash
# OpenMetadata UI에서: Settings → Database → Re-index All
# 또는 API로 실행:
curl -X POST http://localhost:8585/api/v1/search/reindex/all \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"recreateIndex": true}'
```

---

## 10. 트러블슈팅

### 10.1 자주 발생하는 문제

#### OpenMetadata 서버가 시작되지 않음

```bash
# 로그 확인
docker compose logs openmetadata-server | grep -i error

# 일반적 원인:
# 1. DB 연결 실패 → DB_HOST, DB_PORT, DB_USER, DB_USER_PASSWORD 확인
# 2. ES 연결 실패 → ELASTICSEARCH_HOST, ELASTICSEARCH_PORT 확인
# 3. 마이그레이션 미실행 → execute-migrate-all 컨테이너 로그 확인
# 4. 힙 메모리 부족 → OPENMETADATA_HEAP_OPTS 조정

# 마이그레이션 수동 실행
docker compose run --rm execute-migrate-all
```

#### OIDC 로그인 실패

```bash
# 확인 항목:
# 1. Discovery URI 접근 가능한지 확인
curl http://keycloak.company.com:8443/realms/openmetadata/.well-known/openid-configuration

# 2. Callback URL이 Keycloak Redirect URI에 등록되어 있는지 확인
# 3. Client Secret이 올바른지 확인
# 4. AUTHENTICATION_PUBLIC_KEYS에 Keycloak JWKS URL이 포함되어 있는지 확인
```

#### Elasticsearch 인덱스 오류

```bash
# 클러스터 상태 확인
curl http://localhost:9200/_cluster/health?pretty

# 인덱스 목록 확인
curl http://localhost:9200/_cat/indices?v

# 디스크 공간 확인 (디스크 90% 이상 시 read-only 전환)
curl http://localhost:9200/_cat/allocation?v

# read-only 해제
curl -X PUT http://localhost:9200/_all/_settings \
  -H "Content-Type: application/json" \
  -d '{"index.blocks.read_only_allow_delete": null}'
```

#### Airflow Ingestion 실패

```bash
# Airflow 로그 확인
docker compose logs ingestion | grep -i error

# Airflow 웹 UI에서 DAG 실행 상태 확인
# http://<서버IP>:8080 (admin / <비밀번호>)

# OpenMetadata ↔ Airflow 연결 확인
docker exec openmetadata_ingestion curl -s http://openmetadata-server:8585/api/v1/system/health
```

### 10.2 성능 튜닝

```bash
# === PostgreSQL 튜닝 (postgresql.conf) ===
# shared_buffers = RAM의 25% (예: 8GB)
# effective_cache_size = RAM의 75% (예: 24GB)
# work_mem = 10MB (복잡한 쿼리용)
# maintenance_work_mem = 512MB (VACUUM, CREATE INDEX용)
# max_connections = 200
# wal_buffers = 64MB
# checkpoint_completion_target = 0.9

# === Elasticsearch 튜닝 ===
# ES_JAVA_OPTS=-Xms4g -Xmx4g (RAM의 50%, 최대 32GB 미만)
# indices.memory.index_buffer_size = 20%
# thread_pool.search.queue_size = 1000

# === OpenMetadata Server 튜닝 ===
# OPENMETADATA_HEAP_OPTS=-Xmx4G -Xms4G (전사 규모 최소 4GB)
# DB_CONNECTION_POOL_MAX_SIZE=100
# SERVER_MAX_THREADS=150
```

---

## 11. Kubernetes 핵심 개념 요약

K8s를 처음 접하는 분을 위한 OpenMetadata 배포에 필요한 최소 개념입니다.

### 11.1 주요 용어

| 용어 | 설명 | OpenMetadata에서의 역할 |
|------|------|----------------------|
| **Pod** | 컨테이너를 실행하는 최소 단위 | OpenMetadata 서버 1개 = Pod 1개 |
| **Deployment** | Pod의 개수와 업데이트 전략을 관리 | OM 서버 2개 레플리카 유지 |
| **Service** | Pod에 접근하기 위한 네트워크 주소 | `openmetadata:8585`로 접근 |
| **Namespace** | 리소스를 논리적으로 격리 | `openmetadata` 네임스페이스에 모든 리소스 배치 |
| **Secret** | 비밀번호 등 민감 정보 저장 | DB 비밀번호, OIDC 시크릿 |
| **Ingress** | 외부 트래픽을 내부 서비스로 라우팅 | `openmetadata.company.com` → OM 서버 |
| **Helm** | K8s 앱 패키지 관리 도구 | OpenMetadata 설치/업그레이드/롤백 |
| **StatefulSet** | 상태가 있는 앱 관리 (DB 등) | PostgreSQL, Elasticsearch |
| **PersistentVolumeClaim** | 영구 저장소 요청 | DB 데이터, ES 인덱스 |

### 11.2 자주 사용하는 명령어

```bash
# Pod 상태 확인
kubectl get pods -n openmetadata

# Pod 상세 정보 (에러 원인 파악)
kubectl describe pod <pod-name> -n openmetadata

# Pod 로그 확인
kubectl logs <pod-name> -n openmetadata

# Pod 재시작 (Deployment 단위)
kubectl rollout restart deployment/openmetadata -n openmetadata

# 포트 포워딩 (로컬에서 직접 접속)
kubectl port-forward svc/openmetadata 8585:8585 -n openmetadata

# 현재 배포된 Helm 릴리스 확인
helm list -n openmetadata

# 리소스 사용량 확인
kubectl top pods -n openmetadata
```

### 11.3 K8s 팀에 요청할 사항 체크리스트

K8s 인프라 팀에 아래 항목을 요청하세요:

- [ ] `openmetadata` Namespace 생성
- [ ] Ingress Controller 설정 (Nginx Ingress 권장)
- [ ] `openmetadata.company.com` DNS → Ingress Controller IP 매핑
- [ ] TLS 인증서 (cert-manager 또는 수동 등록)
- [ ] PersistentVolume 프로비저너 (PostgreSQL, ES 데이터용)
- [ ] 외부 DB 서버 접근 가능한 네트워크 정책 (NetworkPolicy)
- [ ] 데이터 소스 서버(Oracle, MySQL 등)로의 아웃바운드 허용
- [ ] 리소스 쿼터 (최소: CPU 12 cores, Memory 32 Gi)
