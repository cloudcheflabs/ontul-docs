# Install — standing the stack up from release tarballs

Four cloudcheflabs products make up this demo. **No official Docker images are
published**, so the install path is to download each product's release tarball
and build the image from it. You do not need a source checkout for any of it.

| Product | Role here | Release |
|---|---|---|
| ShannonStore | S3 object storage | `cloudcheflabs/shannonstore-pack` |
| NeorunBase | vector · Korean FTS · graph · OLTP serving | `cloudcheflabs/neorunbase-pack` |
| Ontul | federation · semantic layer · ontology · IAM · Flow | `cloudcheflabs/ontul-pack` |
| kiok | batch scheduler | `cloudcheflabs/kiok-pack` |

Apache Polaris (the Iceberg REST catalog) joins them, along with the source
systems: ERP on PostgreSQL, the approval system on MySQL, a training-records SaaS
over REST, and the embedding service.

!!! warning "What you need"
    A machine with **at least 10 GB given to Docker**, plus `python3.11`, `psql`
    and the `aws` CLI. `ANTHROPIC_API_KEY` only if you want to run the agent. The
    tarballs total about 1.2 GB and the built images take roughly 25 GB.

---

## 1. Fetch the distributions

Each product has a `<name>-pack` repository holding one rolling tag,
`<name>-archive`, so the current build is always at the same URL.

**`demo/infra/dist/fetch.sh`**

```bash
#!/usr/bin/env bash
##
## 컴포넌트 배포본을 내려받습니다.
##
##   bash infra/dist/fetch.sh
##
## 네 제품은 GitHub 릴리스로 배포됩니다. 저장소마다 `<name>-pack` 이 있고 그
## 안에 `<name>-archive` 라는 고정 태그가 하나 있어서, 최신 빌드는 항상 같은
## URL 에 있습니다. 공식 Docker 이미지는 발행하지 않기 때문에, 이미지는 여기서
## 받은 tarball 로 각자 빌드합니다.
##
## 이미 받아 둔 파일은 다시 받지 않습니다. 합쳐서 1GB 가 넘고, 데모를 다시
## 세울 때마다 내려받을 이유는 없습니다. 새 빌드를 받으려면 해당 tar.gz 를
## 지우고 다시 실행하십시오.
##
set -euo pipefail
cd "$(dirname "$0")"

VERSION="${COMPONENT_VERSION:-1.0.0}"
BASE="${COMPONENT_BASE_URL:-https://github.com/cloudcheflabs}"

for name in shannonstore neorunbase ontul kiok; do
  f="${name}-${VERSION}.tar.gz"
  if [ -s "$f" ]; then
    printf '  이미 있음  %s (%s)\n' "$f" "$(du -h "$f" | cut -f1)"
    continue
  fi
  url="$BASE/${name}-pack/releases/download/${name}-archive/${f}"
  printf '  내려받는 중 %s\n' "$url"
  curl -fL -# --retry 3 --retry-delay 2 -o "$f.part" "$url" || {
    rm -f "$f.part"
    echo "실패: $url" >&2
    exit 1
  }
  mv "$f.part" "$f"
  printf '  받음        %s (%s)\n' "$f" "$(du -h "$f" | cut -f1)"
done

echo
echo "배포본 준비 완료 — infra/up.sh 가 이 파일들로 이미지를 빌드합니다."
```


```bash
bash infra/dist/fetch.sh
```

```text
  내려받는 중 https://github.com/cloudcheflabs/shannonstore-pack/releases/download/shannonstore-archive/shannonstore-1.0.0.tar.gz
  받음        shannonstore-1.0.0.tar.gz (112M)
  ...
  배포본 준비 완료 — infra/up.sh 가 이 파일들로 이미지를 빌드합니다.
```

---

## 2. The image Dockerfiles

Three of the products have identically shaped tarballs — `bin/`, `conf/`, `lib/`,
`admin-ui/` — so unpacking one into `/app` is the whole build, and one Dockerfile
serves all three.

**`demo/infra/dist/Dockerfile`**

```dockerfile
# 배포 tarball 하나로 컴포넌트 이미지를 만듭니다. shannonstore / neorunbase /
# kiok 이 전부 같은 모양이라 Dockerfile 도 하나입니다 — 각 tarball 은 bin/,
# conf/, lib/, admin-ui/ 를 그대로 담고 있어서 풀어서 /app 에 놓으면 끝입니다.
#
# 소스 저장소를 참조하지 않는 것이 요점입니다. 이 데모를 따라 하는 사람에게는
# 저장소가 없고, 릴리스 tarball 만 있습니다. 빌드 컨텍스트도 이 디렉터리 하나로
# 닫혀 있어서, fetch.sh 가 받아 둔 tarball 말고는 이미지에 들어갈 것이 없습니다.
FROM eclipse-temurin:17-jre-jammy

# curl 은 헬스체크가, jq 는 setup 스크립트가, netcat 은 기동 순서를 기다리는
# 쪽이 씁니다. python3 은 kiok 이 태스크를 실행할 때 씁니다.
RUN apt-get update && apt-get install -y --no-install-recommends \
        curl jq python3 netcat-openbsd && \
    rm -rf /var/lib/apt/lists/*

WORKDIR /app

ARG NAME
ARG VERSION=1.0.0
COPY ${NAME}-${VERSION}.tar.gz /tmp/dist.tar.gz
RUN tar -xzf /tmp/dist.tar.gz -C /tmp && \
    mv /tmp/${NAME}-${VERSION}/* /app/ && \
    rm -rf /tmp/dist.tar.gz /tmp/${NAME}-${VERSION} && \
    chmod +x /app/bin/*.sh && \
    mkdir -p /app/data /app/logs

# 세 제품 모두 같은 규칙입니다. FOREGROUND 가 켜져 있으면 start-*.sh 가 JVM 을
# exec 해서 컨테이너의 1번 프로세스가 되고, 꺼져 있으면 데몬으로 띄운 뒤 스크립트가
# 끝나 컨테이너가 그대로 종료됩니다.
ENV SHANNONSTORE_FOREGROUND=true \
    NEORUNBASE_FOREGROUND=true \
    KIOK_FOREGROUND=true \
    SHANNONSTORE_LOG_PATH=/app/logs
```


kiok needs one extra thing: a switch for which role the container runs.

**`demo/infra/dist/Dockerfile.kiok`**

```dockerfile
# kiok 은 역할 스위치가 하나 더 필요할 뿐, 나머지는 공용 Dockerfile 과 같습니다.
FROM eclipse-temurin:17-jre-jammy

RUN apt-get update && apt-get install -y --no-install-recommends \
        curl jq python3 netcat-openbsd && \
    rm -rf /var/lib/apt/lists/*

WORKDIR /app

ARG VERSION=1.0.0
COPY kiok-${VERSION}.tar.gz /tmp/dist.tar.gz
COPY kiok-entrypoint.sh /app/docker-entrypoint.sh
RUN tar -xzf /tmp/dist.tar.gz -C /tmp && \
    mv /tmp/kiok-${VERSION}/* /app/ && \
    rm -rf /tmp/dist.tar.gz /tmp/kiok-${VERSION} && \
    chmod +x /app/bin/*.sh /app/docker-entrypoint.sh && \
    mkdir -p /app/data /app/logs

ENV KIOK_FOREGROUND=true
EXPOSE 8080 19999 19998
ENTRYPOINT ["/app/docker-entrypoint.sh"]
```
**`demo/infra/dist/kiok-entrypoint.sh`**

```bash
#!/bin/bash
# kiok 이미지 하나로 master 와 worker 를 모두 띄웁니다. compose 가 KIOK_ROLE 만
# 정해 주면 되고, 그 밖의 설정은 전부 환경변수로 들어갑니다.
set -e
case "${KIOK_ROLE}" in
  master) exec /app/bin/start-master.sh ;;
  worker) exec /app/bin/start-worker.sh ;;
  *) echo "KIOK_ROLE must be 'master' or 'worker' (got: '${KIOK_ROLE}')"; exit 1 ;;
esac
```


The Ontul image is the product distribution plus **this pipeline's Python
dependencies**. Extraction runs as a Python UDF and the worker executes UDFs with
its own system Python, so pdfplumber and friends have to be inside the container
rather than in the client's virtualenv. Keeping them here instead of in the
product image is the point: Ontul has no business shipping a PDF parser, and a
real project builds its own job image for exactly this reason.

**`demo/infra/dist/Dockerfile.ontul`**

```dockerfile
# 데모의 Ontul 이미지: 제품 배포본 + 이 파이프라인이 쓰는 파이썬 의존성.
#
# 추출은 Python UDF 로 돌고, UDF 는 워커가 자기 시스템 python3 으로 실행합니다.
# 그래서 pdfplumber 같은 것은 클라이언트의 virtualenv 가 아니라 컨테이너 안에
# 있어야 합니다. 제품 Dockerfile 이 아니라 여기에 두는 게 요점입니다 — Ontul 이
# PDF 파서를 같이 배포할 이유는 없고, 실제 프로젝트도 잡 이미지는 따로 만듭니다.
FROM eclipse-temurin:17-jre-jammy

# 파이썬 3.11. 잡을 제출하는 클라이언트와 같은 버전이어야 합니다.
#
# cloudpickle 은 함수를 값으로 직렬화하는데 code 객체의 레이아웃이 버전마다
# 다릅니다. 3.11 클라이언트가 jammy 기본 3.10 을 만나면 "code expected at most
# 16 arguments, got 18" 로 끝나는데, 이 메시지에는 파이썬도 버전도 없습니다.
# PySpark 와 같은 규칙입니다 — 드라이버와 실행기는 같은 인터프리터를 씁니다.
RUN apt-get update && apt-get install -y --no-install-recommends \
        curl jq netcat-openbsd software-properties-common gnupg && \
    add-apt-repository -y ppa:deadsnakes/ppa && \
    apt-get update && apt-get install -y --no-install-recommends \
        python3.11 python3.11-venv python3.11-distutils && \
    python3.11 -m ensurepip --upgrade && \
    rm -rf /var/lib/apt/lists/*

# 워커는 $PYTHON3 이 설정돼 있으면 그것으로 UDF 실행기를 띄웁니다.
ENV PYTHON3=/usr/bin/python3.11

# 파이프라인의 추출 의존성. 버전을 고정한 이유는, 추출기가 조용히 다른 텍스트를
# 내놓기 시작하면 그 아래 모든 답이 바뀌는데 파이프라인은 아무것도 보고하지
# 않기 때문입니다.
#
# pg8000 은 순수 파이썬 Postgres 클라이언트로, Ontul 을 통과할 수 없는 단 한
# 단계에 씁니다: NeorunBase 는 JDBC 카탈로그가 아니라서 DELETE 가 거부되고
# ("Not a JDBC catalog: nb") 벡터 테이블은 NeorunBase 자기 프로토콜로 비워야
# 합니다.
RUN python3.11 -m pip install --no-cache-dir \
        pdfplumber==0.11.4 \
        python-docx==1.1.2 \
        openpyxl==3.1.5 \
        boto3==1.35.36 \
        cloudpickle==3.1.0 \
        pyarrow==18.1.0 \
        pg8000==1.31.2

WORKDIR /app

ARG VERSION=1.0.0
COPY ontul-${VERSION}.tar.gz /tmp/dist.tar.gz
RUN tar -xzf /tmp/dist.tar.gz -C /tmp && \
    mv /tmp/ontul-${VERSION}/* /app/ && \
    rm -rf /tmp/dist.tar.gz /tmp/ontul-${VERSION} && \
    chmod +x /app/bin/*.sh && \
    mkdir -p /data/spill /data/rocksdb /data/metadata /data/deps /data/drivers /data/logs

ENV ONTUL_FOREGROUND=true
ENV JAVA_OPTS="-Xmx2g -XX:+UseG1GC"

EXPOSE 8080 47470 19999 29999
```


!!! danger "The Python versions have to match"
    cloudpickle serialises a function by value, and a code object's layout is
    version-specific. A 3.11 client against the container's 3.10 fails with
    `code expected at most 16 arguments, got 18` — a message that mentions
    neither Python nor a version. It is the same rule PySpark has: driver and
    executor run the same interpreter.

---

## 3. The compose files

Each product gets its own compose project. The names are kept distinct so that
tearing down later removes **only what this demo started**.

### ShannonStore — it owns the network

**`demo/infra/compose/shannonstore.yml`**

```yaml
##
## ShannonStore — 이 데모의 S3 스토리지이자, 네트워크의 주인입니다.
##
## 토폴로지는 zk 1 · data 2 · api 1 입니다. nginx 는 없고 S3 엔드포인트가
## api-server-1:8080 으로 바로 열립니다. EC 는 1+1(데이터+패리티)이라 데이터
## 노드 두 대에 맞습니다.
##
## 힙을 전부 명시한 이유: JVM 은 컨테이너가 보고하는 메모리에서 힙을 정하는데,
## 여기 컨테이너들은 호스트 전체를 봅니다. JVM 열 개가 각자 "11GB 의 1/4 은
## 써도 되겠지" 라고 판단하면, 개별 설정은 전부 합리적으로 보이는 채로 머신이
## 스왑을 시작합니다.
##
## setup 컨테이너가 하는 일이 중요합니다. ShannonStore 는 기동할 때 S3 자격증명을
## 갖고 있지 않습니다 — admin 으로 로그인해서 IAM 키를 발급받고, 그 키로 버킷을
## 만들고, 키를 볼륨에 적어 둡니다. Polaris 는 그 키를 받아서 시작해야 하고
## 나중에 바꿀 수 없기 때문에, 기동 순서는 취향이 아니라 제약입니다.
##
services:
  zookeeper:
    image: zookeeper:3.9.1
    container_name: nrn-iceberg-zk
    environment:
      JVMFLAGS: "-Xmx192m"
    networks: [neorun-iceberg-network]
    volumes:
      - zk-data:/data
      - zk-datalog:/datalog
    healthcheck:
      test: ["CMD-SHELL", "echo ruok | nc localhost 2181 | grep imok || nc -z localhost 2181"]
      interval: 3s
      timeout: 5s
      retries: 10

  data-1:
    build: &ss-build
      context: ../dist
      dockerfile: Dockerfile
      args:
        NAME: shannonstore
        VERSION: "${COMPONENT_VERSION:-1.0.0}"
    image: regdemo/shannonstore:${COMPONENT_VERSION:-1.0.0}
    container_name: nrn-iceberg-data-1
    hostname: data-1
    command: ["/app/bin/start-data-node.sh", "-Dshannonstore.nio.port=9001", "-Dshannonstore.data.storage.dirs=data/node1-disk-a,data/node1-disk-b"]
    environment:
      SHANNONSTORE_ZK_CONNECT: zookeeper:2181
      SHANNONSTORE_ADVERTISED_HOST: data-1
      SHANNONSTORE_MASTER_KEY: ShannonStoreMasterKey1200303003X
      SHANNONSTORE_LOG_MODE: RING_BUFFER
      JAVA_OPTS: "-Xms192m -Xmx448m -XX:+UseG1GC -XX:MaxMetaspaceSize=192m"
    depends_on:
      zookeeper: { condition: service_healthy }
    volumes:
      - data1-data:/app/data
    networks: [neorun-iceberg-network]

  data-2:
    build: *ss-build
    image: regdemo/shannonstore:${COMPONENT_VERSION:-1.0.0}
    container_name: nrn-iceberg-data-2
    hostname: data-2
    command: ["/app/bin/start-data-node.sh", "-Dshannonstore.nio.port=9002", "-Dshannonstore.data.storage.dirs=data/node2-disk-a,data/node2-disk-b"]
    environment:
      SHANNONSTORE_ZK_CONNECT: zookeeper:2181
      SHANNONSTORE_ADVERTISED_HOST: data-2
      SHANNONSTORE_MASTER_KEY: ShannonStoreMasterKey1200303003X
      SHANNONSTORE_LOG_MODE: RING_BUFFER
      JAVA_OPTS: "-Xms192m -Xmx448m -XX:+UseG1GC -XX:MaxMetaspaceSize=192m"
    depends_on:
      zookeeper: { condition: service_healthy }
    volumes:
      - data2-data:/app/data
    networks: [neorun-iceberg-network]

  api-server-1:
    build: *ss-build
    image: regdemo/shannonstore:${COMPONENT_VERSION:-1.0.0}
    container_name: nrn-iceberg-api-1
    hostname: api-server-1
    command: ["/app/bin/start-api-server.sh", "-Dshannonstore.api.s3.port=8080", "-Dshannonstore.api.admin.port=8888", "-Dshannonstore.nio.port=9000", "-Dshannonstore.api.node.id=ss-api-1"]
    environment:
      SHANNONSTORE_ZK_CONNECT: zookeeper:2181
      SHANNONSTORE_ADVERTISED_HOST: api-server-1
      SHANNONSTORE_MASTER_KEY: ShannonStoreMasterKey1200303003X
      SHANNONSTORE_LOG_MODE: RING_BUFFER
      SHANNONSTORE_API_S3_EC_DATA_SHARDS: "1"
      SHANNONSTORE_API_S3_EC_PARITY_SHARDS: "1"
      JAVA_OPTS: "-Xms256m -Xmx640m -XX:+UseG1GC -XX:MaxMetaspaceSize=192m"
    ports:
      - "28000:8080"
    depends_on:
      zookeeper: { condition: service_healthy }
      data-1: { condition: service_started }
      data-2: { condition: service_started }
    volumes:
      - api1-data:/app/data
    networks: [neorun-iceberg-network]

  setup:
    image: alpine/curl:latest
    container_name: nrn-iceberg-setup
    depends_on:
      api-server-1: { condition: service_started }
    entrypoint: "/bin/sh"
    command:
      - "-c"
      - |
        apk add --no-cache jq > /dev/null 2>&1
        ADMIN=http://api-server-1:8888
        echo "=== ShannonStore Setup ==="
        echo "[1/3] API 대기..."
        for i in $$(seq 1 90); do
          curl -sf $$ADMIN/admin/health > /dev/null 2>&1 && { echo "  준비됨"; break; }
          [ $$i -eq 90 ] && { echo "  ERROR: 180초 안에 뜨지 않음"; exit 1; }
          sleep 2
        done
        echo "[2/3] 로그인 후 IAM 액세스 키 발급..."
        LOGIN_RES=$$(curl -sf -X POST $$ADMIN/admin/auth/login \
          -H "Content-Type: application/json" -d '{"userId":"admin","password":"admin"}')
        TOKEN=$$(echo $$LOGIN_RES | jq -r '.token')
        [ -n "$$TOKEN" ] && [ "$$TOKEN" != "null" ] || { echo "  ERROR: 로그인 실패"; exit 1; }
        # 초기 비밀번호는 한 번만 통합니다. 회전을 요구하면 회전하고 다시 로그인합니다.
        if [ "$$(echo $$LOGIN_RES | jq -r '.requirePasswordChange')" = "true" ]; then
          curl -sf -X POST $$ADMIN/admin/auth/change-password \
            -H "Content-Type: application/json" -H "Authorization: Bearer $$TOKEN" \
            -d '{"oldPassword":"admin","newPassword":"password123"}'
          LOGIN_RES=$$(curl -sf -X POST $$ADMIN/admin/auth/login \
            -H "Content-Type: application/json" -d '{"userId":"admin","password":"password123"}')
          TOKEN=$$(echo $$LOGIN_RES | jq -r '.token')
        fi
        ACCESS_KEY=""; SECRET_KEY=""
        for attempt in 1 2 3 4 5 6 7 8 9 10; do
          KEY_RES=$$(curl -sf -X POST $$ADMIN/admin/iam/keys \
            -H "Content-Type: application/json" -H "Authorization: Bearer $$TOKEN" \
            -d '{"userId":"admin"}' || true)
          if [ -n "$$KEY_RES" ]; then
            ACCESS_KEY=$$(echo $$KEY_RES | jq -r '.accessKey // empty')
            SECRET_KEY=$$(echo $$KEY_RES | jq -r '.secretKey // empty')
          fi
          [ -n "$$ACCESS_KEY" ] && [ "$$ACCESS_KEY" != "null" ] && { echo "  발급됨 (시도 $$attempt)"; break; }
          sleep 2
        done
        [ -n "$$ACCESS_KEY" ] && [ "$$ACCESS_KEY" != "null" ] || { echo "  ERROR: IAM 키 발급 실패"; exit 1; }
        # 키는 클러스터 전체에 동기화된 뒤에야 S3 요청에 통합니다.
        sleep 10
        echo "[3/3] iceberg-warehouse 버킷 생성..."
        curl -sf -X POST $$ADMIN/admin/browser/buckets \
          -H "Authorization: Bearer $$TOKEN" -H "Content-Type: application/json" \
          -d '{"name":"iceberg-warehouse","versioning":false}'
        echo "$$ACCESS_KEY" > /tmp/setup/access_key
        echo "$$SECRET_KEY" > /tmp/setup/secret_key
        echo "done" > /tmp/setup/ready
        echo "=== 완료 ==="
        tail -f /dev/null
    volumes:
      - setup-data:/tmp/setup
    healthcheck:
      test: ["CMD", "test", "-f", "/tmp/setup/ready"]
      interval: 2s
      timeout: 5s
      retries: 90
      start_period: 10s
    networks: [neorun-iceberg-network]

volumes:
  setup-data:
  api1-data:
  data1-data:
  data2-data:
  zk-data:
  zk-datalog:

networks:
  neorun-iceberg-network:
    driver: bridge
    name: neorun-iceberg-network
```


What the `setup` container does is what fixes the startup order. ShannonStore
holds no S3 credentials when it boots: it logs in as admin, mints an IAM key,
creates the bucket with it, and writes the key to a volume. Polaris has to be
**started with** that key and cannot be told about it afterwards. The ordering is
a constraint, not a preference.

### NeorunBase — the serving layer

**`demo/infra/compose/neorunbase.yml`**

```yaml
##
## NeorunBase — 에이전트가 실제로 조회하는 서빙 계층.
##
## 코디네이터 1 · 데이터노드 2 이고 ZooKeeper 는 ShannonStore 것을 같이 씁니다.
## 이 데모에서 컨테이너 하나는 200MB 이고, 앙상블을 하나 더 띄울 이유가 없습니다.
##
## 코디네이터가 Postgres 와이어 프로토콜(5432)과 admin HTTP(8080)를 둘 다 엽니다.
## psql 로 붙을 수 있다는 뜻이고, 이 데모에서 벡터 테이블을 비우는 한 단계가
## 그 경로를 씁니다 — NeorunBase 는 JDBC 카탈로그가 아니라서 Ontul 을 통한
## DELETE 가 거부됩니다.
##
## 힙은 코디네이터에 더 줍니다. 계획을 세우고, 카탈로그를 들고, Iceberg 동기화를
## 돌리는 쪽이 코디네이터입니다. 데이터노드는 샤드만 서빙하니 더 작아도 됩니다.
##
services:
  neorun-datanode-1:
    build: &nb-build
      context: ../dist
      dockerfile: Dockerfile
      args:
        NAME: neorunbase
        VERSION: "${COMPONENT_VERSION:-1.0.0}"
    image: regdemo/neorunbase:${COMPONENT_VERSION:-1.0.0}
    hostname: neorun-datanode-1
    container_name: nrn-iceberg-datanode-1
    command:
      - "/app/bin/start-datanode.sh"
      - "-Dneorunbase.zookeeper.server.list=zookeeper:2181"
      - "-Dneorunbase.datanode.internal.port=7002"
      - "-Dneorunbase.datanode.advertised.host=neorun-datanode-1"
      - "-Dneorunbase.base.data.dir=data/datanode-1"
      - "-Dneorunbase.datanode.data.dir=data/datanode-1/shards"
      - "-Dneorunbase.log.mode=RING_BUFFER"
    environment:
      # KMS 마스터 키. 코디네이터와 데이터노드가 같은 값을 가져야 합니다 —
      # 다르면 KMS 초기화 단계에서 SecurityException 으로 즉시 죽습니다.
      NEORUNBASE_MASTER_KEY: "${NEORUNBASE_MASTER_KEY:-NeorunBaseMasterKey120030300312345}"
      JAVA_OPTS: "-Xms192m -Xmx448m -XX:+UseG1GC -XX:MaxMetaspaceSize=192m"
    networks: [neorun-iceberg-network]

  neorun-datanode-2:
    build: *nb-build
    image: regdemo/neorunbase:${COMPONENT_VERSION:-1.0.0}
    hostname: neorun-datanode-2
    container_name: nrn-iceberg-datanode-2
    command:
      - "/app/bin/start-datanode.sh"
      - "-Dneorunbase.zookeeper.server.list=zookeeper:2181"
      - "-Dneorunbase.datanode.internal.port=7003"
      - "-Dneorunbase.datanode.advertised.host=neorun-datanode-2"
      - "-Dneorunbase.base.data.dir=data/datanode-2"
      - "-Dneorunbase.datanode.data.dir=data/datanode-2/shards"
      - "-Dneorunbase.log.mode=RING_BUFFER"
    environment:
      # KMS 마스터 키. 코디네이터와 데이터노드가 같은 값을 가져야 합니다 —
      # 다르면 KMS 초기화 단계에서 SecurityException 으로 즉시 죽습니다.
      NEORUNBASE_MASTER_KEY: "${NEORUNBASE_MASTER_KEY:-NeorunBaseMasterKey120030300312345}"
      JAVA_OPTS: "-Xms192m -Xmx448m -XX:+UseG1GC -XX:MaxMetaspaceSize=192m"
    networks: [neorun-iceberg-network]

  neorun-coordinator-1:
    build: *nb-build
    image: regdemo/neorunbase:${COMPONENT_VERSION:-1.0.0}
    hostname: neorun-coordinator-1
    container_name: nrn-iceberg-coordinator-1
    entrypoint: ["sh", "-c"]
    command:
      - >
        /app/bin/start-coordinator.sh
        -Dneorunbase.zookeeper.server.list=zookeeper:2181
        -Dneorunbase.coordinator.pg.port=5432
        -Dneorunbase.coordinator.advertised.host=neorun-coordinator-1
        -Dneorunbase.admin.http.port=8080
        -Dneorunbase.coordinator.internal.port=7100
        -Dneorunbase.base.data.dir=data/coordinator-1
        -Dneorunbase.log.mode=RING_BUFFER
        $$ICEBERG_OPTS
    environment:
      # Iceberg 카탈로그 설정은 up.sh 가 만들어 넣습니다. Polaris 자격증명이
      # 기동할 때마다 새로 발급되기 때문에 compose 에 적어 둘 수가 없습니다.
      # entrypoint 가 sh -c 인 것도 이것 때문입니다 — 셸이 있어야 이 변수가
      # 명령줄 인자로 펼쳐집니다.
      ICEBERG_OPTS: "${ICEBERG_OPTS_1:-}"
      NEORUNBASE_MASTER_KEY: "${NEORUNBASE_MASTER_KEY:-NeorunBaseMasterKey120030300312345}"
      JAVA_OPTS: "-Xms256m -Xmx768m -XX:+UseG1GC -XX:MaxMetaspaceSize=256m"
      AWS_REGION: "${AWS_REGION:-us-east-1}"
    ports:
      - "${COORD1_PG_HOST_PORT:-5434}:5432"
      - "${COORD1_ADMIN_HOST_PORT:-8084}:8080"
    depends_on:
      neorun-datanode-1: { condition: service_started }
      neorun-datanode-2: { condition: service_started }
    networks: [neorun-iceberg-network]

networks:
  neorun-iceberg-network:
    external: true
    name: neorun-iceberg-network
```


### Ontul

**`demo/infra/compose/ontul.yml`**

```yaml
##
## Ontul for the regulation demo: zookeeper + 1 master + 1 worker.
##
## Reduced from the product's tests/docker-compose-ontul.yml (2 masters, 2
## workers, nginx). One master means no leader election to observe and no
## forwarding path to exercise; one worker means the scan does not visibly
## spread. Both still run the identical code path — embed_passage() is
## evaluated per Arrow batch by whichever worker owns the split, and owning
## every split is a valid case of owning some. What the reduction costs is the
## demonstration, not the behaviour. Raise the worker count when showing
## distribution is the point.
##
## Joins neorun-iceberg-network, which the ShannonStore slim compose creates.
## That matters more than it looks: Polaris hands clients an S3 endpoint of
## http://api-server-1:8080, a name that only resolves inside that network. An
## Ontul running on the host would take the catalog's word for it and then fail
## to reach the storage the catalog just described.
##
services:
  ontul-zookeeper:
    image: zookeeper:3.9.1
    hostname: ontul-zookeeper
    container_name: regdemo-ontul-zk
    environment:
      JVMFLAGS: "-Xmx256m"
    networks: [neorun-iceberg-network]
    healthcheck:
      test: ["CMD-SHELL", "echo ruok | nc localhost 2181 | grep imok || nc -z localhost 2181"]
      interval: 3s
      timeout: 5s
      retries: 20

  ontul-master-1:
    build:
      context: ../dist
      dockerfile: Dockerfile.ontul
      args:
        VERSION: "${COMPONENT_VERSION:-1.0.0}"
    image: regdemo/ontul:${COMPONENT_VERSION:-1.0.0}
    hostname: ontul-master-1
    container_name: regdemo-ontul-master-1
    volumes:
      - ontul-m1-data:/data
    entrypoint: ["sh", "-c"]
    command:
      - >
        /app/bin/start-master.sh
        -Dontul.zk.serverList=ontul-zookeeper:2181
        -Dontul.master.host=ontul-master-1
        -Dontul.master.admin.port=8080
        -Dontul.master.flight.sql.port=47470
        -Dontul.master.internal.port=19999
        -Dontul.master.admin.ui.static.path=/app/admin-ui
        -Dontul.base.data.dir=/data
        -Dontul.python.path=/usr/bin/python3.11
        -Dontul.log.output.name=ontul.log
        -Dontul.kms.master.key.env=ONTUL_MASTER_KEY
    environment:
      ONTUL_FOREGROUND: "true"
      ONTUL_MASTER_KEY: "regdemo-ontul-master-key-32chars"
      JAVA_OPTS: "-Xms512m -Xmx1280m -XX:+UseG1GC -XX:MaxMetaspaceSize=256m"
    ports:
      - "8080:8080"     # Admin UI + REST
      - "47470:47470"   # Arrow Flight SQL
    depends_on:
      ontul-zookeeper:
        condition: service_healthy
    networks: [neorun-iceberg-network]
    healthcheck:
      test: ["CMD-SHELL", "curl -sf http://localhost:8080/admin/ready || exit 1"]
      interval: 5s
      timeout: 5s
      retries: 40
      start_period: 20s

  ontul-worker-1:
    build:
      context: ../dist
      dockerfile: Dockerfile.ontul
      args:
        VERSION: "${COMPONENT_VERSION:-1.0.0}"
    image: regdemo/ontul:${COMPONENT_VERSION:-1.0.0}
    hostname: ontul-worker-1
    container_name: regdemo-ontul-worker-1
    entrypoint: ["sh", "-c"]
    command:
      - >
        /app/bin/start-worker.sh
        -Dontul.zk.serverList=ontul-zookeeper:2181
        -Dontul.worker.host=ontul-worker-1
        -Dontul.worker.internal.port=29999
        -Dontul.base.data.dir=/data
        -Dontul.python.path=/usr/bin/python3.11
        -Dontul.log.output.name=ontul.log
        -Dontul.kms.master.key.env=ONTUL_MASTER_KEY
        -Dontul.job.logs.memory.tail=2000
    environment:
      ONTUL_FOREGROUND: "true"
      ONTUL_MASTER_KEY: "regdemo-ontul-master-key-32chars"
      # Arrow allocates off-heap. The heap cap is deliberately below what the
      # worker could use so the batch pipeline stays in direct memory, where
      # embed_passage() hands its FixedSizeList straight to the vector.
      # 워커가 마스터보다 큽니다. 이 스택에서 워커 하나가 동시에 지는 짐이
      # 그만큼입니다 — CDC 5 + 그래프 2 + 결재 1, 여덟 개의 스트리밍 잡이 상주한
      # 상태에서 임베딩 배치가 청크 800여 개를 Arrow 배치로 밀어 넣습니다.
      # 1280m 에서는 색인 도중 워커가 죽었고, 그때 나오는 말은 메모리가 아니라
      # "Worker … failed and retry exhausted: null" 이라 원인을 가리키지 않습니다.
      JAVA_OPTS: "-Xms768m -Xmx2560m -XX:+UseG1GC -XX:MaxMetaspaceSize=384m -XX:MaxDirectMemorySize=1024m"
    depends_on:
      # service_started, not service_healthy — this saves a minute, it does not
      # avoid a deadlock. The master holds /admin/ready until a worker registers
      # and then gives up after 60s and proceeds anyway
      # (MasterServer.waitForClusterReady). Waiting for healthy therefore means
      # the worker sits out that entire timeout for no reason: what it needs is
      # ZooKeeper, which is already up before the master starts.
      ontul-master-1:
        condition: service_started
    networks: [neorun-iceberg-network]

networks:
  neorun-iceberg-network:
    external: true
    name: neorun-iceberg-network

volumes:
  ontul-m1-data:
```


### kiok

**`demo/infra/compose/kiok.yml`**

```yaml
# kiok — 파이프라인의 스케줄러.
#
# 지금까지 단계 사이의 의존 순서는 infra/index.sh 라는 셸 스크립트에만 있었습니다.
# 그건 한 번 돌리기에는 충분하지만 파이프라인이라고 부르기는 어렵습니다 — 어느
# 단계가 어디서 실패했는지, 무엇을 다시 돌려야 하는지, 어제 것과 오늘 것이 어떻게
# 달랐는지가 아무 데도 남지 않습니다. DAG 로 옮기면 그게 전부 데이터가 됩니다.
#
# ZooKeeper 는 새로 띄우지 않고 ontul 것을 씁니다. chroot 로 갈라두면 같은 앙상블을
# 쓰면서 서로의 노드를 보지 않고, 이 데모에서 컨테이너 하나는 200MB 입니다.
#
# 힙을 256m 까지 내린 이유도 같습니다. 이 DAG 는 태스크가 대여섯 개이고 무거운 일은
# 전부 ontul 워커가 합니다 — kiok 은 순서를 지키고 결과를 기록할 뿐이라 데이터를
# 들고 있지 않습니다. 동시 태스크가 적으니 G1 보다 Serial 이 상주 비용이 낮습니다.
services:
  kiok-master:
    build:
      context: ../dist
      dockerfile: Dockerfile.kiok
      args: { VERSION: "${COMPONENT_VERSION:-1.0.0}" }
    image: regdemo/kiok:${COMPONENT_VERSION:-1.0.0}
    container_name: regdemo-kiok-master
    hostname: kiok-master
    ports: ["18081:8080"]
    environment:
      KIOK_ROLE: master
      KIOK_MASTER_KEY: regdemo-kiok-master-key-32-chars
      KIOK_MASTER_HOST: kiok-master
      KIOK_ZK_SERVERLIST: ontul-zookeeper:2181/kiok
      KIOK_MASTER_ADMIN_CONTEXT_PATH: /
      JAVA_OPTS: "-Xmx256m -XX:MaxMetaspaceSize=192m"
    volumes:
      - kiok-master-data:/app/data
    networks: [neorun-iceberg-network]
    deploy:
      resources:
        limits: {memory: 512M}
    healthcheck:
      test: ["CMD", "curl", "-sf", "http://kiok-master:8080/healthz"]
      interval: 5s
      timeout: 5s
      retries: 40

  kiok-worker:
    build:
      context: ../dist
      dockerfile: Dockerfile.kiok
      args: { VERSION: "${COMPONENT_VERSION:-1.0.0}" }
    image: regdemo/kiok:${COMPONENT_VERSION:-1.0.0}
    container_name: regdemo-kiok-worker
    hostname: kiok-worker
    depends_on:
      kiok-master: { condition: service_healthy }
    environment:
      KIOK_ROLE: worker
      KIOK_MASTER_KEY: regdemo-kiok-master-key-32-chars
      KIOK_WORKER_HOST: kiok-worker
      KIOK_ZK_SERVERLIST: ontul-zookeeper:2181/kiok
      JAVA_OPTS: "-Xmx256m -XX:MaxMetaspaceSize=192m"
    volumes:
      - kiok-worker-data:/app/data
    networks: [neorun-iceberg-network]
    deploy:
      resources:
        limits: {memory: 512M}

volumes:
  kiok-master-data:
  kiok-worker-data:

networks:
  neorun-iceberg-network:
    external: true
    name: neorun-iceberg-network
```


### The source systems

ERP (PostgreSQL), the approval system (MySQL), training records (REST) and the
embedding service live in the demo's own root compose file.

**`demo/docker-compose.yml`**

```yaml
##
## Regulation lakehouse demo — the sources.
##
## Only the sources live here. ShannonStore, Polaris, NeorunBase and Ontul are
## brought up from their own compose files by infra/up.sh, which also creates the
## catalog and hands every resolved endpoint to out/stack.env. Running this file
## alone gives you the four services below and nothing to point them at.
##
## The network is external on purpose: ShannonStore's compose owns it, and
## Polaris hands out an S3 endpoint (http://api-server-1:8080) that resolves
## nowhere else.
##
##   make seed && bash infra/up.sh
##
name: regdemo

services:
  # ── The embedding model. Standalone, and offline by construction ──────────
  # Weights are baked into the image and HF_HUB_OFFLINE is set, so regulation
  # text has no route out of this network — the guarantee is structural, not a
  # policy someone has to remember. Indexing and search call this one process,
  # which is what makes a stored vector and a query vector comparable at all.
  # It refuses to serve if it cannot name the revision it loaded: a vector that
  # cannot be pinned to a generation is worse than no vector, because it still
  # looks like an answer.
  embed-svc:
    build:
      context: ./infra/embed-svc
      args:
        EMBED_MODEL_ID: ${EMBED_MODEL_ID:-intfloat/multilingual-e5-base}
    container_name: regdemo-embed-svc
    ports: ["8100:8000"]
    environment:
      EMBED_DIM: "768"
      EMBED_DEVICE: cpu
      EMBED_MAX_BATCH: "64"
    networks: [neorun-iceberg-network]
    deploy:
      resources:
        limits: {memory: 1500M}
    healthcheck:
      test: ["CMD", "python", "-c", "import urllib.request,json,sys; sys.exit(0 if json.load(urllib.request.urlopen('http://localhost:8000/health'))['ready'] else 1)"]
      interval: 10s
      retries: 60

  # ── ERP. Federated live rather than copied: a leave balance is only worth
  #    quoting if it is the balance right now ────────────────────────────────
  postgres-erp:
    image: postgres:16-alpine
    container_name: regdemo-erp
    # 55432, not 5434 — NeorunBase's coordinator serves the PostgreSQL wire
    # protocol on 5434 and would win the bind race silently.
    ports: ["55432:5432"]
    environment:
      POSTGRES_DB: erp
      POSTGRES_USER: erp
      POSTGRES_PASSWORD: ${ERP_PASSWORD:-regdemo}
    # wal_level=logical is what makes this an ERP a CDC pipeline can read. The
    # default (replica) carries enough for a physical standby and not enough to
    # reconstruct row changes, so Debezium fails at slot creation rather than
    # part-way through — which is the better direction to fail in.
    # One replication slot per synced table, so the ceiling is the table count.
    command: ["postgres", "-c", "shared_buffers=96MB", "-c", "max_connections=50",
              "-c", "wal_level=logical", "-c", "max_replication_slots=16",
              "-c", "max_wal_senders=16"]
    volumes:
      - ./out/sql/erp_postgres.sql:/docker-entrypoint-initdb.d/10_erp.sql:ro
      - erp-data:/var/lib/postgresql/data
    networks: [neorun-iceberg-network]
    deploy:
      resources:
        limits: {memory: 320M}
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U erp -d erp"]
      interval: 10s
      retries: 30

  # ── 전자결재. Authoritative for effective dates — the approval, not the
  #    document's 부칙, is what makes a regulation binding — and the CDC source
  #    Flow reacts to when an approval completes ─────────────────────────────
  mysql-groupware:
    image: mysql:8.0
    container_name: regdemo-groupware
    ports: ["33306:3306"]
    # Row-image binlog for CDC. The buffer pool is cut well below the default
    # because this database holds 122 approval rows, not a workload; the memory
    # is worth more to the JVMs sharing this machine.
    command: >
      --server-id=1 --log-bin=binlog --binlog-format=ROW --binlog-row-image=FULL
      --innodb-buffer-pool-size=64M --performance-schema=OFF
    environment:
      MYSQL_DATABASE: groupware
      MYSQL_USER: gw
      MYSQL_PASSWORD: ${GW_PASSWORD:-regdemo}
      MYSQL_ROOT_PASSWORD: ${GW_ROOT_PASSWORD:-regdemo-root}
    volumes:
      - ./out/sql/groupware_mysql.sql:/docker-entrypoint-initdb.d/10_groupware.sql:ro
      - gw-data:/var/lib/mysql
    networks: [neorun-iceberg-network]
    deploy:
      resources:
        limits: {memory: 448M}
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost", "-p${GW_ROOT_PASSWORD:-regdemo-root}"]
      interval: 10s
      retries: 40

  # ── A source with an API and no database, reached through rest-operation.
  #    Not every system worth joining will hand you a JDBC URL ───────────────
  mock-saas:
    build: ./infra/mock-saas
    container_name: regdemo-lms
    ports: ["8200:8000"]
    volumes: ["./out/lms.json:/app/lms.json:ro"]
    networks: [neorun-iceberg-network]
    deploy:
      resources:
        limits: {memory: 256M}
    healthcheck:
      test: ["CMD", "python", "-c", "import urllib.request; urllib.request.urlopen('http://localhost:8000/health')"]
      interval: 10s
      retries: 30

networks:
  neorun-iceberg-network:
    external: true
    name: neorun-iceberg-network

volumes:
  erp-data:
  gw-data:
```


The embedding service **bakes the model weights into its image** and pins itself
offline.

**`demo/infra/embed-svc/Dockerfile`**

```dockerfile
FROM python:3.11-slim

# CPU-only torch: measured faster than MPS for this model+sequence shape, and it
# keeps the image off the CUDA wheels (several GB) that this demo never uses.
RUN pip install --no-cache-dir \
        torch --index-url https://download.pytorch.org/whl/cpu \
 && pip install --no-cache-dir \
        "sentence-transformers>=3" fastapi "uvicorn[standard]" pydantic

# Bake the weights into the image. Two reasons, and the second is the point:
#   1. container start does not wait on a 1.1GB download
#   2. with HF_HUB_OFFLINE=1 below, the running service physically cannot reach
#      the internet — regulation text has no path out, by construction rather
#      than by policy.
ARG EMBED_MODEL_ID=intfloat/multilingual-e5-base
RUN python -c "from sentence_transformers import SentenceTransformer as S; S('${EMBED_MODEL_ID}')"

ENV EMBED_MODEL_ID=${EMBED_MODEL_ID} \
    EMBED_DIM=768 \
    EMBED_DEVICE=cpu \
    HF_HUB_OFFLINE=1 \
    TRANSFORMERS_OFFLINE=1

WORKDIR /app
COPY app.py .
EXPOSE 8000
HEALTHCHECK --interval=10s --timeout=3s --retries=30 \
    CMD python -c "import urllib.request,json,sys; \
        sys.exit(0 if json.load(urllib.request.urlopen('http://localhost:8000/health'))['ready'] else 1)"
CMD ["uvicorn", "app:app", "--host", "0.0.0.0", "--port", "8000"]
```
**`demo/infra/embed-svc/app.py`**

```python
"""Embedding service — the single vector-space authority for the demo.

Indexing (Ontul worker UDF) and search (agent) both call this one process, so
the index and the query cannot be produced by different weights. That is the
whole reason it exists as a service rather than a library each side imports.

Runs offline by design: the model is baked into the image at build time and
HF_HUB_OFFLINE=1 is set, so regulation text has no path out of the network.
"""
from __future__ import annotations

import logging, os, time
from pathlib import Path
from typing import Literal

from fastapi import FastAPI, HTTPException
from pydantic import BaseModel, Field

logging.basicConfig(level=logging.INFO, format="%(asctime)s %(levelname)s %(message)s")
log = logging.getLogger("embed-svc")

MODEL_ID = os.environ.get("EMBED_MODEL_ID", "intfloat/multilingual-e5-base")
DIM = int(os.environ.get("EMBED_DIM", "768"))
NORMALIZE = os.environ.get("EMBED_NORMALIZE", "true").lower() == "true"
DEVICE = os.environ.get("EMBED_DEVICE", "cpu")
# e5 is trained asymmetrically. Applied here rather than by callers, because a
# forgotten prefix costs retrieval quality and raises nothing.
PASSAGE_PREFIX, QUERY_PREFIX = "passage: ", "query: "
MAX_BATCH = int(os.environ.get("EMBED_MAX_BATCH", "64"))

app = FastAPI(title="regdemo embed-svc")
_state: dict = {}


@app.on_event("startup")
def _load() -> None:
    from sentence_transformers import SentenceTransformer

    t0 = time.time()
    st = SentenceTransformer(MODEL_ID, device=DEVICE)
    dim = st.get_sentence_embedding_dimension()
    if dim != DIM:
        # The dimension is fixed in the NeorunBase VECTOR(n) column; serving a
        # different one would produce rows the target table cannot hold.
        raise RuntimeError(f"model dim={dim} but service declares {DIM}")

    revision = _resolve_revision()

    _state.update(st=st, revision=revision, load_seconds=time.time() - t0)
    log.info("loaded %s rev=%s dim=%d device=%s in %.1fs",
             MODEL_ID, revision or "?", dim, DEVICE, _state["load_seconds"])


def _resolve_revision() -> str:
    """The commit actually on disk.

    Not read from the loaded model: `auto_model.name_or_path` echoes back the
    name that was requested ("intfloat/multilingual-e5-base"), not the revision
    it resolved to — so a service trusting it reports no revision at all and,
    by design, refuses to serve. The Hub cache records the commit in two places;
    refs/ is the branch pointer and snapshots/ is the materialised tree, and
    either is authoritative for what these weights are.
    """
    from huggingface_hub.constants import HF_HUB_CACHE

    repo = Path(HF_HUB_CACHE) / ("models--" + MODEL_ID.replace("/", "--"))
    refs = repo / "refs" / "main"
    if refs.is_file():
        rev = refs.read_text(encoding="utf-8").strip()
        if rev:
            return rev
    snaps = repo / "snapshots"
    if snaps.is_dir():
        # A pinned build has exactly one; more than one means the image was
        # rebuilt against a moving tag, which is precisely the drift worth
        # refusing rather than picking a winner from.
        found = sorted(d.name for d in snaps.iterdir() if d.is_dir())
        if len(found) == 1:
            return found[0]
        log.error("%d snapshots present for %s: %s — cannot pin a revision",
                  len(found), MODEL_ID, found)
    return ""


def _fingerprint() -> str:
    rev = _state.get("revision") or ""
    if not rev:
        raise HTTPException(503, "model revision unresolved; refusing to serve "
                                 "vectors that cannot be pinned to a generation")
    return f"{MODEL_ID}@{rev}:{DIM}:{'l2' if NORMALIZE else 'raw'}:e5v1"


class EmbedRequest(BaseModel):
    texts: list[str] = Field(..., min_length=1)
    kind: Literal["passage", "query"] = "passage"


class EmbedResponse(BaseModel):
    vectors: list[list[float]]
    fingerprint: str
    elapsed_ms: int


@app.get("/health")
def health() -> dict:
    return {"ready": "st" in _state, "model": MODEL_ID, "dim": DIM, "device": DEVICE}


@app.get("/fingerprint")
def fingerprint() -> dict:
    """Registered against an embedding generation at index time and checked at
    query time. Everything about the vector space that can drift is in here."""
    return {"fingerprint": _fingerprint(), "model_id": MODEL_ID,
            "revision": _state.get("revision"), "dim": DIM,
            "normalize": NORMALIZE, "distance": "cosine", "prefix_scheme": "e5v1"}


@app.post("/embed", response_model=EmbedResponse)
def embed(req: EmbedRequest) -> EmbedResponse:
    if "st" not in _state:
        raise HTTPException(503, "model still loading")
    if len(req.texts) > MAX_BATCH:
        # Bounded so one caller cannot occupy the (single, CPU-bound) model for
        # an unbounded time while others queue behind it.
        raise HTTPException(413, f"batch of {len(req.texts)} exceeds {MAX_BATCH}")

    prefix = PASSAGE_PREFIX if req.kind == "passage" else QUERY_PREFIX
    t0 = time.time()
    vecs = _state["st"].encode([prefix + t for t in req.texts],
                               batch_size=min(32, len(req.texts)),
                               normalize_embeddings=NORMALIZE,
                               show_progress_bar=False, convert_to_numpy=True)
    return EmbedResponse(vectors=vecs.tolist(), fingerprint=_fingerprint(),
                         elapsed_ms=int((time.time() - t0) * 1000))
```


!!! note "Why offline is not a preference here"
    You cannot send a corpus of internal regulations to a third-party API to find
    out what it says. `HF_HUB_OFFLINE=1` / `TRANSFORMERS_OFFLINE=1` is a condition
    of the demo existing at all. The service also refuses to serve unless it can
    prove which revision it loaded — a vector whose model revision cannot be
    named cannot be pinned to a generation.

Training records stand in for a SaaS with no database of its own. It is what
Ontul's REST connector attaches to.

**`demo/infra/mock-saas/Dockerfile`**

```dockerfile
FROM python:3.11-slim
RUN pip install --no-cache-dir fastapi "uvicorn[standard]"
WORKDIR /app
COPY app.py .
EXPOSE 8000
CMD ["uvicorn", "app:app", "--host", "0.0.0.0", "--port", "8000"]
```
**`demo/infra/mock-saas/app.py`**

```python
"""교육 이수 SaaS — a source with an API and no database.

Reached through Ontul's rest-operation connector, so the demo shows that a
system which never exposes a DB still joins with the rest. The payload is the
same file the seed generator wrote; this only serves it.
"""
from __future__ import annotations

import json
import os
from pathlib import Path

from fastapi import FastAPI, HTTPException, Query

app = FastAPI(title="mock LMS")
DATA = Path(os.environ.get("LMS_DATA", "/app/lms.json"))
_cache: dict = {}


def _load() -> dict:
    if not _cache:
        if not DATA.exists():
            raise HTTPException(503, f"{DATA} not present — run the seed generator first")
        _cache.update(json.loads(DATA.read_text(encoding="utf-8")))
    return _cache


@app.get("/health")
def health() -> dict:
    return {"ready": DATA.exists()}


@app.get("/courses")
def courses(doc_no: str | None = Query(None)) -> list[dict]:
    rows = _load()["courses"]
    return [c for c in rows if doc_no is None or c["doc_no"] == doc_no]


@app.get("/completions")
def completions(course_id: str | None = Query(None),
                emp_no: str | None = Query(None)) -> list[dict]:
    rows = _load()["completions"]
    if course_id:
        rows = [r for r in rows if r["course_id"] == course_id]
    if emp_no:
        rows = [r for r in rows if r["emp_no"] == emp_no]
    return rows


@app.get("/outstanding")
def outstanding(course_id: str) -> dict:
    """Who has not completed a required course.

    Answering "정보보안규정이 개정됐는데 교육 미이수자는?" still needs the
    regulation graph and the org chart on top of this — which is the point of
    having it as a separate source rather than another table.
    """
    data = _load()
    if not any(c["course_id"] == course_id for c in data["courses"]):
        raise HTTPException(404, f"no such course: {course_id}")
    done = {r["emp_no"] for r in data["completions"] if r["course_id"] == course_id}
    return {"course_id": course_id, "completed": sorted(done)}
```


---

## 4. Bring it up

**`demo/infra/up.sh`**

```bash
#!/usr/bin/env bash
##
## Bring up the whole demo stack, in the only order that works.
##
## The ordering is not stylistic. ShannonStore has to exist before Polaris,
## because Polaris is configured with credentials ShannonStore mints at startup
## and cannot be told about them later. Polaris has to hold a catalog before
## NeorunBase and Ontul start, because both read the catalog at boot and a
## missing warehouse is a startup failure rather than a retry. And the network
## belongs to ShannonStore's compose, so everything else joins as external and
## must follow it.
##
## 이 스크립트는 소스 저장소를 하나도 참조하지 않습니다. 네 제품은 전부
## infra/dist/fetch.sh 가 받아 둔 릴리스 tarball 로 이미지를 빌드합니다 —
## 공식 Docker 이미지는 발행되지 않기 때문에, 배포본을 받아 각자 굽는 것이
## 유일한 설치 경로이고, 이 데모를 따라 하는 쪽에도 그것만 있으면 됩니다.
##
set -euo pipefail

DEMO="$(cd "$(dirname "$0")/.." && pwd)"
OVR="$DEMO/infra/compose"
DIST="$DEMO/infra/dist"
COMPONENT_VERSION="${COMPONENT_VERSION:-1.0.0}"
export COMPONENT_VERSION

NETWORK=neorun-iceberg-network
SETUP_CONTAINER=nrn-iceberg-setup
POLARIS_NAME=regdemo-polaris
# Pinned to what chango ships and operates, not :latest. A demo that runs on a
# different Polaris than the product does is testing a different product.
POLARIS_IMAGE="${POLARIS_IMAGE:-apache/polaris:1.4.1}"
POLARIS_PORT=28181
CATALOG="${CATALOG:-regdemo_catalog}"
WAREHOUSE=s3://iceberg-warehouse/
S3_INTERNAL=http://api-server-1:8080
NB_PG_PORT=5434
NB_ADMIN_PORT=8084
STACK_ENV="$DEMO/out/stack.env"

log()  { printf '\n\033[1m=== %s ===\033[0m\n' "$*"; }
step() { printf '  %s\n' "$*"; }
fail() { printf '\033[31mFAIL: %s\033[0m\n' "$*" >&2; exit 1; }

# 배포본이 없으면 여기서 멈추는 편이 낫습니다. 없는 채로 진행하면 docker build 가
# COPY 에서 "not found" 로 죽는데, 그 메시지는 Docker 문제처럼 읽힙니다.
for n in shannonstore neorunbase ontul kiok; do
  [ -s "$DIST/$n-$COMPONENT_VERSION.tar.gz" ] || \
    fail "$n-$COMPONENT_VERSION.tar.gz 없음 — 'bash infra/dist/fetch.sh' 를 먼저 실행하십시오"
done
[ -f "$DEMO/out/sql/erp_postgres.sql" ] || fail "corpus not generated — run 'make seed' first"

# ── 1. ShannonStore. Creates the network, mints S3 credentials, makes the bucket
log "1/7  ShannonStore (zk + data x2 + api x1)"
docker compose -p regdemo-ss --project-directory "$OVR" -f "$OVR/shannonstore.yml" up -d --build \
  > /tmp/regdemo-ss.log 2>&1 || { tail -30 /tmp/regdemo-ss.log; fail "shannonstore up (see /tmp/regdemo-ss.log)"; }
step "waiting for setup to mint credentials and create the bucket..."
for i in $(seq 1 150); do
  [ "$(docker inspect -f '{{.State.Health.Status}}' "$SETUP_CONTAINER" 2>/dev/null || echo x)" = healthy ] && break
  [ "$i" = 150 ] && { docker logs --tail 40 "$SETUP_CONTAINER" 2>&1; fail "shannonstore setup never went healthy"; }
  sleep 2
done
ACCESS_KEY=$(docker exec "$SETUP_CONTAINER" cat /tmp/setup/access_key 2>/dev/null | tr -d '\r\n')
SECRET_KEY=$(docker exec "$SETUP_CONTAINER" cat /tmp/setup/secret_key 2>/dev/null | tr -d '\r\n')
[ -n "$ACCESS_KEY" ] && [ -n "$SECRET_KEY" ] || fail "no S3 credentials came out of setup"
step "S3 ready — access key ${ACCESS_KEY:0:8}…, bucket iceberg-warehouse"

# ── 2. Polaris. Its own container, named for this demo so it never fights the
#       shannon-polaris / nrn-perf-polaris that the product suites start.
log "2/7  Polaris (Iceberg REST catalog)"
docker rm -f "$POLARIS_NAME" >/dev/null 2>&1 || true
docker run -d --name "$POLARIS_NAME" --network "$NETWORK" -p "${POLARIS_PORT}:8181" \
  -e POLARIS_BOOTSTRAP_CREDENTIALS=POLARIS,root,s3cr3t \
  -e polaris.realm-context.realms=POLARIS \
  -e quarkus.otel.sdk.disabled=true \
  -e POLARIS_FEATURES_SKIP_CREDENTIAL_SUBSCOPING_INDIRECTION=true \
  -e POLARIS_FEATURES_ALLOW_SETTING_S3_ENDPOINTS=true \
  -e JAVA_OPTS_APPEND="-Xmx512m" \
  -e AWS_ACCESS_KEY_ID="$ACCESS_KEY" -e AWS_SECRET_ACCESS_KEY="$SECRET_KEY" -e AWS_REGION=us-east-1 \
  "$POLARIS_IMAGE" >/dev/null || fail "polaris did not start"

ptoken() {
  curl -sf "http://localhost:${POLARIS_PORT}/api/catalog/v1/oauth/tokens" \
    --user root:s3cr3t -H "Polaris-Realm: POLARIS" \
    -d grant_type=client_credentials -d "scope=PRINCIPAL_ROLE:ALL" \
    | python3 -c "import sys,json; print(json.load(sys.stdin)['access_token'])"
}
PT=""
for i in $(seq 1 60); do PT=$(ptoken 2>/dev/null || true); [ -n "$PT" ] && break; sleep 3; done
[ -n "$PT" ] || { docker logs --tail 30 "$POLARIS_NAME"; fail "polaris never issued a token"; }
step "polaris authenticated"

# ── 3. The catalog. Created before anything reads it.
log "3/7  Catalog '$CATALOG'"
curl -sf -H "Authorization: Bearer $PT" -H "Content-Type: application/json" -H "Polaris-Realm: POLARIS" \
  "http://localhost:${POLARIS_PORT}/api/management/v1/catalogs" -d "{
    \"catalog\": { \"name\": \"$CATALOG\", \"type\": \"INTERNAL\", \"readOnly\": false,
      \"properties\": { \"default-base-location\": \"$WAREHOUSE\" },
      \"storageConfigInfo\": { \"storageType\": \"S3\", \"endpoint\": \"$S3_INTERNAL\",
        \"endpointInternal\": \"$S3_INTERNAL\", \"pathStyleAccess\": true,
        \"allowedLocations\": [\"$WAREHOUSE\"], \"stsUnavailable\": true } } }" >/dev/null 2>&1 \
  && step "created" || step "already present"
curl -sf -H "Authorization: Bearer $PT" -H "Content-Type: application/json" -H "Polaris-Realm: POLARIS" \
  -X PUT "http://localhost:${POLARIS_PORT}/api/management/v1/catalogs/${CATALOG}/catalog-roles/catalog_admin/grants" \
  -d '{"type":"catalog","privilege":"CATALOG_MANAGE_CONTENT"}' >/dev/null 2>&1 || true
step "catalog_admin granted CATALOG_MANAGE_CONTENT"

# A principal that can actually use the catalog.
#
# On Polaris 1.4.x the bootstrap clientId is not registered as a PRINCIPAL, so
# catalog operations performed as 'root' are rejected with "TopLevelEntity of
# type PRINCIPAL does not exist" — the credentials authenticate and then cannot
# do anything. chango provisions a per-catalog admin principal for exactly this
# reason (PolarisClusterService.ensureCatalogAdminPrincipal); the demo does the
# same rather than inventing a second answer.
PRINCIPAL="${CATALOG}_admin"
PRINCIPAL_ROLE="${CATALOG}_admin_pr"
pol() {  # pol <METHOD> <path> [body]
  if [ -n "${3:-}" ]; then
    curl -s -o /tmp/regdemo-pol.out -w '%{http_code}' -X "$1" \
      -H "Authorization: Bearer $PT" -H 'Content-Type: application/json' -H 'Polaris-Realm: POLARIS' \
      "http://localhost:${POLARIS_PORT}$2" -d "$3"
  else
    curl -s -o /tmp/regdemo-pol.out -w '%{http_code}' -X "$1" \
      -H "Authorization: Bearer $PT" -H 'Polaris-Realm: POLARIS' \
      "http://localhost:${POLARIS_PORT}$2"
  fi
}

CODE=$(pol POST /api/management/v1/principals "{\"principal\":{\"name\":\"$PRINCIPAL\"}}")
if [ "${CODE:0:1}" = "2" ]; then
  CAT_CLIENT_ID=$(python3 -c "import json;print(json.load(open('/tmp/regdemo-pol.out'))['credentials']['clientId'])")
  CAT_CLIENT_SECRET=$(python3 -c "import json;print(json.load(open('/tmp/regdemo-pol.out'))['credentials']['clientSecret'])")
  step "principal $PRINCIPAL created"
elif [ "$CODE" = "409" ]; then
  # Polaris only ever vends a principal's secret once, at creation. Rotate to get
  # a usable pair rather than guessing at one we were never given.
  CODE=$(pol POST "/api/management/v1/principals/$PRINCIPAL/rotate")
  [ "${CODE:0:1}" = "2" ] || fail "principal $PRINCIPAL exists and its secret could not be rotated (HTTP $CODE)"
  CAT_CLIENT_ID=$(python3 -c "import json;print(json.load(open('/tmp/regdemo-pol.out'))['credentials']['clientId'])")
  CAT_CLIENT_SECRET=$(python3 -c "import json;print(json.load(open('/tmp/regdemo-pol.out'))['credentials']['clientSecret'])")
  step "principal $PRINCIPAL existed — secret rotated"
else
  fail "could not create principal $PRINCIPAL (HTTP $CODE): $(head -c 200 /tmp/regdemo-pol.out)"
fi

pol POST /api/management/v1/principal-roles "{\"principalRole\":{\"name\":\"$PRINCIPAL_ROLE\"}}" >/dev/null
pol PUT "/api/management/v1/principals/$PRINCIPAL/principal-roles" \
    "{\"principalRole\":{\"name\":\"$PRINCIPAL_ROLE\"}}" >/dev/null
pol PUT "/api/management/v1/principal-roles/$PRINCIPAL_ROLE/catalog-roles/$CATALOG" \
    '{"catalogRole":{"name":"catalog_admin"}}' >/dev/null
step "$PRINCIPAL → $PRINCIPAL_ROLE → catalog_admin on $CATALOG"

# Prove it before anything depends on it: authenticate as the new principal and
# list namespaces. Creating the chain and assuming it works is how a catalog that
# rejects every operation gets discovered three stages later.
CT=$(curl -sf "http://localhost:${POLARIS_PORT}/api/catalog/v1/oauth/tokens" \
      --user "$CAT_CLIENT_ID:$CAT_CLIENT_SECRET" -H 'Polaris-Realm: POLARIS' \
      -d grant_type=client_credentials -d 'scope=PRINCIPAL_ROLE:ALL' \
      | python3 -c "import sys,json;print(json.load(sys.stdin).get('access_token',''))") || true
[ -n "$CT" ] || fail "the catalog principal could not authenticate"
NSCODE=$(curl -s -o /dev/null -w '%{http_code}' \
  -H "Authorization: Bearer $CT" -H 'Polaris-Realm: POLARIS' \
  "http://localhost:${POLARIS_PORT}/api/catalog/v1/${CATALOG}/namespaces")
[ "${NSCODE:0:1}" = "2" ] || fail "the catalog principal cannot use the catalog (namespaces → HTTP $NSCODE)"
step "verified: the principal can list namespaces"

# ── 4. NeorunBase. Serves the vectors and the Korean FTS.
log "4/7  NeorunBase (coordinator + datanode x2)"
IC="-Dneorunbase.iceberg.catalog.type=rest \
-Dneorunbase.iceberg.catalog.rest.uri=http://${POLARIS_NAME}:8181/api/catalog \
-Dneorunbase.iceberg.catalog.rest.warehouse=${CATALOG} \
-Dneorunbase.iceberg.catalog.rest.security=OAUTH2 \
-Dneorunbase.iceberg.catalog.rest.client-id=${CAT_CLIENT_ID} \
-Dneorunbase.iceberg.catalog.rest.client-secret=${CAT_CLIENT_SECRET} \
-Dneorunbase.iceberg.catalog.rest.extra.properties=prefix=${CATALOG},header.Polaris-Realm=POLARIS,scope=PRINCIPAL_ROLE:ALL \
-Dneorunbase.iceberg.s3.endpoint=${S3_INTERNAL} \
-Dneorunbase.iceberg.s3.access.key=${ACCESS_KEY} \
-Dneorunbase.iceberg.s3.secret.key=${SECRET_KEY} \
-Dneorunbase.iceberg.s3.region=us-east-1 \
-Dneorunbase.iceberg.s3.path.style.access=true \
-Dneorunbase.iceberg.sync.interval.ms=8000 \
-Dneorunbase.iceberg.default.namespace=reg"
export ICEBERG_OPTS_1="$IC" AWS_REGION=us-east-1
export COORD1_PG_HOST_PORT=$NB_PG_PORT COORD1_ADMIN_HOST_PORT=$NB_ADMIN_PORT
docker compose -p regdemo-nb --project-directory "$OVR" -f "$OVR/neorunbase.yml" up -d --build \
  > /tmp/regdemo-nb.log 2>&1 || { tail -30 /tmp/regdemo-nb.log; fail "neorunbase up (see /tmp/regdemo-nb.log)"; }
for i in $(seq 1 90); do
  curl -sf "http://localhost:${NB_ADMIN_PORT}/admin/health" >/dev/null 2>&1 && break
  [ "$i" = 90 ] && { docker logs --tail 40 nrn-iceberg-coordinator-1 2>&1 | tail -40; fail "neorunbase coordinator never answered"; }
  sleep 2
done
step "coordinator up — pg :$NB_PG_PORT, admin :$NB_ADMIN_PORT"

# NeorunBase refuses connections on the PostgreSQL wire until the default admin
# password has been rotated, and rotation only happens over the admin HTTP API.
# A cluster that is "up" but unreachable by psql is not up for any purpose this
# demo has, so rotating belongs here rather than in a later script.
NB_PASSWORD="${NEORUNBASE_PASSWORD:-Regdemo12345}"
nb_token() {
  curl -s -X POST "http://localhost:${NB_ADMIN_PORT}/admin/auth/login" \
    -H 'Content-Type: application/json' -d "{\"username\":\"admin\",\"password\":\"$1\"}" \
  | python3 -c "import sys,json;d=json.load(sys.stdin);print(d.get('token') or d.get('accessToken') or '')" 2>/dev/null
}
if [ -n "$(nb_token "$NB_PASSWORD")" ]; then
  step "password already rotated"
else
  NBT=$(nb_token admin)
  [ -n "$NBT" ] || fail "neorunbase admin login failed with the default password"
  curl -sf -X POST "http://localhost:${NB_ADMIN_PORT}/admin/auth/change-password" \
    -H 'Content-Type: application/json' -H "Authorization: Bearer $NBT" \
    -d "{\"oldPassword\":\"admin\",\"newPassword\":\"$NB_PASSWORD\"}" >/dev/null \
    || fail "neorunbase password rotation failed"
  step "default password rotated"
fi
PGPASSWORD="$NB_PASSWORD" psql -h localhost -p "$NB_PG_PORT" -U admin -d neorunbase -t -A -c "SELECT 1" >/dev/null 2>&1 \
  || fail "neorunbase rotated its password but still refuses the wire protocol"
step "postgres wire accepts connections"

# ── 5. The sources: the embedding model, ERP, 전자결재, the SaaS with no database.
log "5/7  Sources (embed-svc, ERP, groupware, LMS)"
S3_ACCESS_KEY="$ACCESS_KEY" S3_SECRET_KEY="$SECRET_KEY" \
  docker compose -p regdemo --project-directory "$DEMO" -f "$DEMO/docker-compose.yml" up -d --build \
  > /tmp/regdemo-src.log 2>&1 || { tail -30 /tmp/regdemo-src.log; fail "sources up (see /tmp/regdemo-src.log)"; }
for i in $(seq 1 120); do
  curl -sf http://localhost:8100/health 2>/dev/null | grep -q '"ready": *true' && break
  [ "$i" = 120 ] && { docker logs --tail 30 regdemo-embed-svc 2>&1; fail "embed-svc never became ready"; }
  sleep 2
done
FP=$(curl -sf http://localhost:8100/fingerprint | python3 -c "import sys,json;print(json.load(sys.stdin)['fingerprint'])")
step "embedding model: $FP"

# ── 6. Ontul. Last, because it reads the catalog and the sources at boot.
log "6/7  Ontul (master 1 + worker 1)"
docker compose -p regdemo-ontul --project-directory "$OVR" -f "$OVR/ontul.yml" up -d --build \
  > /tmp/regdemo-ontul.log 2>&1 || { tail -30 /tmp/regdemo-ontul.log; fail "ontul up (see /tmp/regdemo-ontul.log)"; }
for i in $(seq 1 90); do
  curl -sf http://localhost:8080/admin/ready >/dev/null 2>&1 && break
  [ "$i" = 90 ] && { docker logs --tail 40 regdemo-ontul-master-1 2>&1 | tail -40; fail "ontul master never became ready"; }
  sleep 2
done
step "master ready on :8080"

# ── 7. kiok. 파이프라인의 스케줄러입니다.
#
# ZooKeeper 는 새로 띄우지 않고 Ontul 것을 chroot(/kiok) 로 갈라 씁니다. 그래서
# Ontul 다음입니다. 이 DAG 는 태스크가 예닐곱 개이고 무거운 일은 전부 Ontul
# 워커가 하기 때문에 kiok 은 순서를 지키고 결과를 기록할 뿐이고, 힙 256m 이면
# 충분합니다.
log "7/7  kiok (master + worker)"
# chroot 는 미리 있어야 합니다. Curator 는 접속 문자열에 붙은 chroot 자체는
# 만들어 주지 않기 때문에, 없으면 마스터가 "NoNode for /kiok" 로 기동하다 죽습니다.
# ZooKeeper 문제처럼 읽히지만 실제로는 아직 아무도 만들지 않았다는 뜻입니다.
docker exec regdemo-ontul-zk \
  bash -c 'echo "create /kiok \"\"" | /apache-zookeeper-*-bin/bin/zkCli.sh -server localhost:2181' \
  > /dev/null 2>&1 || true
step "ZooKeeper chroot /kiok 준비"
docker compose -p regdemo-kiok --project-directory "$OVR" -f "$OVR/kiok.yml" up -d --build \
  > /tmp/regdemo-kiok.log 2>&1 || { tail -30 /tmp/regdemo-kiok.log; fail "kiok up (see /tmp/regdemo-kiok.log)"; }
for i in $(seq 1 90); do
  curl -sf http://localhost:18081/healthz >/dev/null 2>&1 && break
  [ "$i" = 90 ] && { docker logs --tail 40 regdemo-kiok-master 2>&1 | tail -40; fail "kiok master never became ready"; }
  sleep 2
done
step "kiok ready on :18081"

# ── Hand the resolved stack to everything downstream, so no script re-derives it.
mkdir -p "$(dirname "$STACK_ENV")"
cat > "$STACK_ENV" <<ENVEOF
# Written by infra/up.sh. Credentials are minted per bring-up — do not commit.
export S3_ENDPOINT_INTERNAL=$S3_INTERNAL
export S3_ENDPOINT_HOST=http://localhost:28000
export S3_ACCESS_KEY=$ACCESS_KEY
export S3_SECRET_KEY=$SECRET_KEY
export S3_REGION=us-east-1
export S3_WAREHOUSE=$WAREHOUSE
export POLARIS_URI_INTERNAL=http://${POLARIS_NAME}:8181/api/catalog
export POLARIS_URI_HOST=http://localhost:${POLARIS_PORT}/api/catalog
export POLARIS_CATALOG=$CATALOG
# The catalog principal, not the bootstrap root — root authenticates but cannot
# perform catalog operations on 1.4.x.
export POLARIS_CLIENT_ID=$CAT_CLIENT_ID
export POLARIS_CLIENT_SECRET=$CAT_CLIENT_SECRET
export POLARIS_ROOT_ID=root
export POLARIS_ROOT_SECRET=s3cr3t
export NEORUNBASE_PG=postgresql://admin@localhost:${NB_PG_PORT}/neorunbase
export NEORUNBASE_ADMIN=http://localhost:${NB_ADMIN_PORT}
export NEORUNBASE_PASSWORD=$NB_PASSWORD
export NEORUNBASE_INTERNAL_HOST=neorun-coordinator-1
export ONTUL_URL=http://localhost:8080
export EMBED_URL_INTERNAL=http://embed-svc:8000
export EMBED_URL_HOST=http://localhost:8100
export EMBED_FINGERPRINT=$FP
# Split out so a job can record the generation's identity field by field. The
# fingerprint is the authority; these are its parts, taken from the same string
# the live endpoint reported rather than re-declared.
export EMBED_MODEL_ID=${FP%%@*}
export EMBED_MODEL_REVISION=$(printf '%s' "${FP#*@}" | cut -d: -f1)
export EMBED_GENERATION=${EMBED_GENERATION:-gen1}
export EMBED_DIM=$(printf '%s' "$FP" | cut -d: -f2)
# The source passwords, so nothing downstream has to guess them or repeat the
# compose defaults. They are the compose defaults unless overridden there.
export ERP_PASSWORD=${ERP_PASSWORD:-regdemo}
export GW_PASSWORD=${GW_PASSWORD:-regdemo}
export ERP_PG=postgresql://erp@localhost:55432/erp
export GROUPWARE_MYSQL=mysql://gw@localhost:33306/groupware
export LMS_URL=http://localhost:8200
ENVEOF

log "Stack up"
cat <<SUMEOF
  Ontul Admin UI    http://localhost:8080
  Polaris           http://localhost:${POLARIS_PORT}
  ShannonStore S3   http://localhost:28000
  NeorunBase        psql -h localhost -p ${NB_PG_PORT} -U admin -d neorunbase
  Embedding         http://localhost:8100   $FP
  ERP / 전자결재     :55432 / :33306
  LMS               http://localhost:8200
  kiok (스케줄러)    http://localhost:18081

  Resolved endpoints written to out/stack.env
  Next:  bash infra/register.sh   (catalogs, connections, schema, IAM)
SUMEOF
```


```bash
bash infra/up.sh
```

```text
=== 1/7  ShannonStore (zk + data x2 + api x1) ===
  S3 ready — access key 7C546709…, bucket iceberg-warehouse
=== 2/7  Polaris (Iceberg REST catalog) ===
  polaris authenticated
=== 3/7  Catalog 'regdemo_catalog' ===
  regdemo_catalog_admin → regdemo_catalog_admin_pr → catalog_admin on regdemo_catalog
  verified: the principal can list namespaces
=== 4/7  NeorunBase (coordinator + datanode x2) ===
  coordinator up — pg :5434, admin :8084
  default password rotated
  postgres wire accepts connections
=== 5/7  Sources (embed-svc, ERP, groupware, LMS) ===
  embedding model: intfloat/multilingual-e5-base@d128750…:768:l2:e5v1
=== 6/7  Ontul (master 1 + worker 1) ===
  master ready on :8080
=== 7/7  kiok (master + worker) ===
  ZooKeeper chroot /kiok 준비
  kiok ready on :18081

=== Stack up ===
  Ontul Admin UI    http://localhost:8080/admin
  Polaris           http://localhost:28181
  ShannonStore S3   http://localhost:28000
  NeorunBase        psql -h localhost -p 5434 -U admin -d neorunbase
  Embedding         http://localhost:8100
  ERP / approvals   :55432 / :33306
  Training records  http://localhost:8200
  kiok (scheduler)  http://localhost:18081
```

### Why the odd-looking lines in that script are there

Tearing the whole thing down and rebuilding it from tarballs is what surfaced
these. Each one was something an operator did by hand once and would not have
remembered.

- **`NEORUNBASE_MASTER_KEY`** used to arrive as an `ENV` in the product
  Dockerfile. Without it in compose, the coordinator dies immediately in KMS
  initialisation with a `SecurityException`.
- **The coordinator's `$$ICEBERG_OPTS`.** Omit it and NeorunBase comes up
  **without Iceberg**, quietly. Nothing fails; the vector table is simply empty
  much later.
- **The ZooKeeper chroot `/kiok`.** Curator does not create the chroot named in a
  connect string. Without it the kiok master dies with `NoNode for /kiok`, which
  reads as a ZooKeeper problem and is not one.
- **Bringing kiok up at all.** It was in no script. Someone started it by hand
  once and it kept running.

---

## 5. Tear down

**`demo/infra/down.sh`**

```bash
#!/usr/bin/env bash
##
## Tear down everything infra/up.sh started, and nothing else.
##
## The three product composes use distinct project names precisely so this can
## be specific. Volumes go with them: the Iceberg catalog lives in Polaris and
## the tables live in ShannonStore, so keeping one without the other leaves a
## catalog pointing at files that are gone, or files no catalog knows about.
##
set -uo pipefail

DEMO="$(cd "$(dirname "$0")/.." && pwd)"
OVR="$DEMO/infra/compose"

echo "=== tearing down the demo stack ==="
docker compose -p regdemo-ontul --project-directory "$OVR" -f "$OVR/ontul.yml" down -v 2>/dev/null
docker compose -p regdemo --project-directory "$DEMO" -f "$DEMO/docker-compose.yml" down -v 2>/dev/null
docker compose -p regdemo-kiok --project-directory "$OVR" -f "$OVR/kiok.yml" down -v 2>/dev/null
docker compose -p regdemo-nb --project-directory "$OVR" -f "$OVR/neorunbase.yml" down -v 2>/dev/null
docker rm -f regdemo-polaris 2>/dev/null
docker compose -p regdemo-ss --project-directory "$OVR" -f "$OVR/shannonstore.yml" down -v 2>/dev/null
rm -f "$DEMO/out/stack.env"
echo "  done — out/stack.env removed; the corpus in out/ is kept"
```


The distinct project names are what make this specific: only what this demo
started comes down. Volumes go with the containers — the Iceberg catalog lives in
Polaris and the tables live in ShannonStore, so keeping one without the other
leaves either a catalog pointing at files that are gone, or files no catalog
knows about.

---

Next: [the corpus](corpus.md) — what this demo actually reads.
