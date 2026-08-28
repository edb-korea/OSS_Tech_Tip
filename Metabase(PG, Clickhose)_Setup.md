Metabase 설치 가이드 (PostgreSQL + ClickHouse)

> Metabase 54버전부터는 ClickHouse 드라이버가 코어에 기본 내장되어 별도 플러그인 설치가 필요 없습니다. 설치 후 버전을 확인해서 목록에 ClickHouse가 안 보이면 그때 별도 드라이버를 추가하면 됩니다.

---

## 1. 사전 준비 (RHEL 9)

Metabase는 Java(JVM) 위에서 동작합니다. **Java 21 이상**이 필요합니다.

```bash
# 시스템 업데이트
sudo dnf update -y

# Java 21 설치 (Temurin/OpenJDK)
sudo dnf install -y java-21-openjdk java-21-openjdk-devel

# 확인
java -version
```

---

## 2. Metabase 애플리케이션 DB용 PostgreSQL 설치

Metabase는 자체 메타데이터(대시보드, 질문, 계정 등)를 저장할 DB가 필요합니다. 기본 내장 H2는 운영 환경에 권장되지 않으므로, PostgreSQL을 애플리케이션 DB로 사용합니다. (동일한 PostgreSQL 서버를 나중에 데이터 소스로도 연결 가능)

```bash
# PostgreSQL 설치
sudo dnf install -y postgresql-server postgresql
sudo postgresql-setup --initdb
sudo systemctl enable --now postgresql

# Metabase 전용 DB/사용자 생성
sudo -u postgres psql <<EOF
CREATE USER metabase WITH PASSWORD '비밀번호를_강력하게_설정하세요';
CREATE DATABASE metabaseappdb OWNER metabase;
GRANT ALL PRIVILEGES ON DATABASE metabaseappdb TO metabase;
EOF
```

방화벽에서 5432 포트를 열어야 한다면:

```bash
sudo firewall-cmd --permanent --add-port=5432/tcp
sudo firewall-cmd --reload
```

---

## 3. Metabase 설치 (JAR 방식)

```bash
# 전용 사용자와 디렉토리 생성
sudo useradd -r -m -s /sbin/nologin metabase
sudo mkdir -p /opt/metabase/plugins
cd /opt/metabase

# 최신 Metabase JAR 다운로드
sudo curl -L -o metabase.jar https://downloads.metabase.com/latest/metabase.jar

sudo chown -R metabase:metabase /opt/metabase
```

> Docker/Podman을 쓸 수 있다면 `docker run -d -p 3000:3000 --name metabase metabase/metabase` 로 훨씬 간단하게 설치할 수 있습니다.

---

## 4. ClickHouse 드라이버 준비 (내장 안 되어 있을 경우 대비)

```bash
sudo curl -L -o /opt/metabase/plugins/clickhouse.metabase-driver.jar \
  https://github.com/ClickHouse/metabase-clickhouse-driver/releases/latest/download/clickhouse.metabase-driver.jar
sudo chown metabase:metabase /opt/metabase/plugins/clickhouse.metabase-driver.jar
```

Metabase 54+ 라면 이미 내장되어 있어 이 파일이 무시되거나 충돌할 수 있습니다. 먼저 이 단계를 건너뛰고 설치 후 **Admin > Databases**에서 "ClickHouse"가 목록에 보이는지 확인하고, 안 보이면 그때 이 드라이버를 추가하는 것을 권장합니다.

---

## 5. systemd 서비스 등록

### 환경변수 파일 생성

```bash
sudo tee /etc/sysconfig/metabase > /dev/null <<'EOF'
MB_DB_TYPE=postgres
MB_DB_DBNAME=metabaseappdb
MB_DB_PORT=5432
MB_DB_USER=metabase
MB_DB_PASS=비밀번호를_강력하게_설정하세요
MB_DB_HOST=localhost
MB_JETTY_PORT=3000
MB_JETTY_HOST=0.0.0.0
EOF

sudo chmod 600 /etc/sysconfig/metabase
sudo chown metabase:metabase /etc/sysconfig/metabase
```

### 서비스 파일 생성

```bash
sudo tee /etc/systemd/system/metabase.service > /dev/null <<'EOF'
[Unit]
Description=Metabase Business Intelligence Server
After=network.target postgresql.service

[Service]
Type=simple
User=metabase
Group=metabase
EnvironmentFile=/etc/sysconfig/metabase
WorkingDirectory=/opt/metabase
ExecStart=/usr/bin/java --add-opens java.base/java.nio=ALL-UNNAMED -jar /opt/metabase/metabase.jar
Restart=on-failure
RestartSec=10
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=multi-user.target
EOF
```

### 실행

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now metabase

# 로그 확인 (초기 마이그레이션에 몇 분 걸림)
sudo journalctl -u metabase -f
```

로그에 `Metabase Initialization Complete`가 뜨면 준비 완료입니다.

---

## 6. 방화벽 및 초기 설정

```bash
sudo firewall-cmd --permanent --add-port=3000/tcp
sudo firewall-cmd --reload
```

브라우저에서 `http://서버IP:3000` 접속 → 초기 설정 마법사 진행 (관리자 계정 생성 등)

---

## 7. 데이터 소스 연결

### PostgreSQL 연결

**Admin > Databases > Add database**

| 항목 | 값 |
|---|---|
| Database type | PostgreSQL |
| Host | (PostgreSQL 서버 주소) |
| Port | 5432 |
| Database name | (연결할 DB명) |
| Username | (계정) |
| Password | (비밀번호) |

### ClickHouse 연결

**Admin > Databases > Add database**

| 항목 | 값 |
|---|---|
| Database type | ClickHouse |
| Host | (ClickHouse 서버 주소) |
| Port | 8123 (HTTP) / 8443 (HTTPS) |
| Database name | (연결할 DB명) |
| Username | (계정) |
| Password | (비밀번호) |

운영 환경에서는 조회 전용(SELECT-only) 계정 사용을 권장합니다:

```sql
CREATE USER metabase_user IDENTIFIED BY 'password'
  SETTINGS readonly = 1;
GRANT SELECT ON your_db.* TO metabase_user;
```

---

## 참고 사항 / 추가로 고려할 점

- Docker/Podman 방식으로 대신 설치
- Nginx 리버스 프록시 + HTTPS(SSL) 구성
- SELinux 관련 설정
