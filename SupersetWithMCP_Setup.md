# Apache Superset Linux 설치 · PostgreSQL 연동 · MCP 서버 구성 가이드

> 대상: Ubuntu/Debian 계열 리눅스 서버 (RHEL 계열은 패키지 매니저만 `yum`/`dnf`로 대체)
> 기준 버전: Apache Superset 5.0+ (MCP 서버 기본 내장), 2026년 8월 기준 공식 문서

---

## 1. 사전 준비: OS 의존성 설치

Superset은 접속 정보 암호화를 위해 `cryptography` 라이브러리를 사용하며, 이 라이브러리가 OS 레벨 패키지에 의존합니다.

```bash
sudo apt-get update
sudo apt-get install -y \
  build-essential libssl-dev libffi-dev python3-dev python3-pip python3-venv \
  libsasl2-dev libldap2-dev libpq-dev default-libmysqlclient-dev pkg-config
```

- `libpq-dev`는 PostgreSQL 연결에 필요한 패키지입니다(이번 가이드에서 필수).
- RHEL/Fedora 계열은 다음 명령을 사용합니다.

```bash
sudo dnf install -y gcc gcc-c++ libffi-devel python3-devel python3-pip python3-wheel \
  openssl-devel cyrus-sasl-devel openldap-devel
```

Python은 Superset이 공식 지원하는 버전(3.11 권장)을 사용하세요.

### 1.1 Superset 전용 리눅스 사용자 계정 생성

Superset을 root로 직접 실행하는 것은 권장되지 않습니다. systemd 유닛(2.4절/4.6절)에서도 `User=superset`으로 지정하고 있으므로, 아래처럼 로그인 셸이 없는 시스템 계정을 만들어 둡니다.

```bash
# 홈 디렉터리 /opt/superset, 로그인 셸 없음(서비스 전용 계정)
sudo useradd --system --create-home --home-dir /opt/superset --shell /usr/bin/bash superset

# 계정 생성 확인
id superset
```

venv 생성이나 `pip install`처럼 최초 설치 작업을 이 계정 권한으로 수행하려면 `sudo -u superset -H bash -c '...'`를 쓰거나, 아래처럼 셸을 잠깐 전환합니다.

```bash
sudo -iu superset   # bash 로그인 셸이 없으므로 -s 옵션으로 진입 필요할 수 있음
```

> 만약 sudo 권한 없이 특정 관리자 계정(예: `ec2-user`)에서 그대로 설치하고 싶다면 위 계정 생성 단계는 건너뛰고, 이후 예시의 `superset:superset` 소유권 및 systemd `User=superset` 값을 본인 계정명으로 바꿔서 따라가면 됩니다.

---

## 2. 가상환경 생성 및 Superset 설치

```bash
python3 -m venv /opt/superset/venv
source /opt/superset/venv/bin/activate

pip install --upgrade pip
# setuptools 82+ 는 pkg_resources를 제거해 Superset 기동이 깨지므로 82 미만으로 고정 (5.1절 참고)
pip install 'setuptools<82'

# Superset 본체 + PostgreSQL 드라이버 + 운영에 필요한 패키지
pip install apache_superset psycopg2-binary gunicorn gevent flask_cors Pillow rich cachetools

pip install "Flask-Caching<2.3" 
```

> `psycopg2`가 PostgreSQL용 Python DB 드라이버입니다. 메타데이터 DB든, 연결 대상 데이터 소스든 PostgreSQL을 쓴다면 반드시 필요합니다. `Pillow`가 빠지면 차트 스크린샷/PDF 내보내기가 동작하지 않습니다(5.2절 참고).

> **순서 주의**: `superset db upgrade`(스키마 초기화)와 `fab create-admin`(관리자 계정 생성)은 **`superset_config.py`에 `SQLALCHEMY_DATABASE_URI`를 PostgreSQL로 지정한 뒤에** 실행해야 합니다. 먼저 실행하면 `superset_config.py`가 없거나 `SUPERSET_CONFIG_PATH`가 지정되지 않은 상태이므로 Superset 기본값인 SQLite(`~/.superset/superset.db`)에 스키마와 관리자 계정이 만들어져 버립니다. 그 상태에서 나중에 설정을 PostgreSQL로 바꾸면 관리자 계정을 포함한 모든 메타데이터가 새 DB에는 없어서 다시 `db upgrade`/`fab create-admin`/`init`을 처음부터 해야 합니다. 아래 2.1 → 2.2 → 2.3 순서를 그대로 따라가세요.

### 2.1 PostgreSQL 메타데이터 DB 준비

Superset의 대시보드/차트/사용자 정의(메타데이터)를 저장할 전용 PostgreSQL DB와 계정을 먼저 만듭니다. 3장에서 연동할 **분석 대상 데이터소스와는 별개의 DB**로 분리하는 것을 권장합니다(같은 PostgreSQL 서버는 공유해도 무방).

```bash
psql -d edb -h <db_ip_addr> -U enterprisedb -p 5444
```

```sql
CREATE USER superset_meta WITH PASSWORD '강력한비밀번호';  -- P@ssw0rd!
CREATE DATABASE superset_meta OWNER superset_meta;
\q
```

접속이 되는지 미리 확인해 둡니다.

```bash
psql "postgresql://superset_meta:강력한비밀번호@localhost:5432/superset_meta" -c '\conninfo'
# or
psql -d superset_meta -p 5444 -U superset_meta -h 10.0.8.192 -c '\conninfo'
psql "postgresql://superset_meta:P%40ssw0rd!@localhost:5444/superset_meta" -c '\conninfo'
```

> `localhost`가 아닌 별도 PostgreSQL 서버를 쓴다면 `pg_hba.conf`에서 이 계정의 접속을 허용해야 하고, 방화벽에서 5432 (혹은 5444) 포트를 열어줘야 합니다.

### 2.2 superset_config.py 만들기

Superset은 `superset/config.py`에 정의된 수백 개의 기본값을 그대로 쓰되, 사용자가 원하는 항목만 **자신만의 설정 모듈**(관례상 파일명 `superset_config.py`)로 덮어쓰는 구조입니다. 이 파일을 만든 뒤, 다음 둘 중 하나로 Superset에게 위치를 알려줘야 합니다.

- 파일을 `PYTHONPATH`에 포함시키기, 또는
- `SUPERSET_CONFIG_PATH` 환경변수로 절대경로 지정 (권장)

```bash
mkdir -p /opt/superset
nano /opt/superset/superset_config.py
```

최소 구성 예시 — **메타데이터 DB로 방금 만든 PostgreSQL 사용**:

```python
# /opt/superset/superset_config.py
import os

# --- 필수: 세션 서명/암호화에 쓰이는 키 (openssl rand -base64 42 로 생성) ---
SECRET_KEY = os.environ.get("SUPERSET_SECRET_KEY", "여기에_강력한_랜덤키")

# --- 메타데이터 DB: PostgreSQL (운영 권장) ---
SQLALCHEMY_DATABASE_URI = (
    "postgresql://superset_meta:강력한비밀번호@localhost:5432/superset_meta"
)

# 커넥션 풀 설정 (동시 접속량에 맞게 조정)
SQLALCHEMY_POOL_SIZE = 10
SQLALCHEMY_MAX_OVERFLOW = 20
SQLALCHEMY_POOL_TIMEOUT = 30
SQLALCHEMY_POOL_RECYCLE = 3600

# --- 기본 행 제한 ---
ROW_LIMIT = 5000

# --- 리버스 프록시(Nginx 등) 뒤에 있다면 X-Forwarded-* 헤더 신뢰 ---
ENABLE_PROXY_FIX = True

# --- CSRF 보호 (기본 True 유지 권장) ---
WTF_CSRF_ENABLED = True
WTF_CSRF_TIME_LIMIT = 60 * 60 * 24 * 365

# --- 피처 플래그 예시 ---
FEATURE_FLAGS = {
    "ALERT_REPORTS": True,
}

# --- 필수 캐시 설정: 지정하지 않으면 기본 SupersetMetastoreCache로 폴백되며,
# flask-caching 최신 버전과의 호환성 문제(5.3절)를 피하기 위해 명시적으로 지정 ---
FILTER_STATE_CACHE_CONFIG = {
    "CACHE_TYPE": "SimpleCache",
    "CACHE_DEFAULT_TIMEOUT": 86400,
}
EXPLORE_FORM_DATA_CACHE_CONFIG = {
    "CACHE_TYPE": "SimpleCache",
    "CACHE_DEFAULT_TIMEOUT": 86400,
}

# --- MCP 서버 설정 (4장 참고) ---
MCP_SERVICE_HOST = "0.0.0.0"
MCP_SERVICE_PORT = 5008
MCP_DEV_USERNAME = "admin"  # 운영에서는 JWT 인증으로 교체
```

> **참고**: 메타데이터 DB(위 `superset_meta`)와 3장에서 연동하는 **분석 대상 PostgreSQL 데이터소스는 서로 다른 DB**입니다 — 같은 PostgreSQL 서버를 공유해도 되지만, 대시보드/차트 정의(메타데이터)와 실제 조회 데이터(데이터소스)는 반드시 별도 DB로 분리하세요.
>
> `FILTER_STATE_CACHE_CONFIG`/`EXPLORE_FORM_DATA_CACHE_CONFIG`를 `SimpleCache`로 지정한 것은 별도 Redis 없이 바로 기동하기 위함입니다. gunicorn을 멀티 워커(`-w 4` 등)로 운영한다면 워커별로 캐시가 분리되어 필터 상태가 가끔 안 맞을 수 있으니, 운영 환경에서는 Redis(`CACHE_TYPE: "RedisCache"`)로 교체하는 것을 권장합니다.

파일을 저장한 뒤 환경변수로 위치를 지정합니다.

```bash
export SUPERSET_CONFIG_PATH=/opt/superset/superset_config.py
```

### 2.3 메타데이터 DB 스키마 초기화 및 관리자 계정 생성

**여기서부터 실행하는 `superset db upgrade`, `fab create-admin`, `superset init`이 실제로 2.1에서 만든 PostgreSQL(`superset_meta`)에 스키마와 관리자 계정을 씁니다.** `SUPERSET_CONFIG_PATH`가 지정된 셸(또는 앞서 `export`한 것과 같은 셸 세션)에서 실행하세요.

```bash
# superset_config.py를 아직 못 찾는 새 셸이라면 다시 지정
export SUPERSET_CONFIG_PATH=/opt/superset/superset_config.py
export FLASK_APP=superset

# superset_config.py의 SECRET_KEY = os.environ.get("SUPERSET_SECRET_KEY", ...) 가
# 이 값을 읽으므로, 파일에 키를 직접 하드코딩하지 않았다면 반드시 먼저 export
export SUPERSET_SECRET_KEY=$(openssl rand -base64 42)    # jYqWJpZbV9S5rBH54K61EFpJlpmXBbLVxQtpfA4hC96sWqVI7pCNVH6D

# PostgreSQL(superset_meta)에 메타데이터 스키마 생성
superset db upgrade

# 관리자 계정 생성 -- 이 계정도 PostgreSQL의 ab_user 테이블에 저장됨
superset fab create-admin \
  --username admin \
  --firstname Admin \
  --lastname User \
  --email admin@example.com \
  --password '강력한비밀번호'   # P@ssw0rd!

# 기본 역할/권한 생성
superset init
```

정상적으로 PostgreSQL을 보고 있는지 확인하려면:

```bash
psql "postgresql://superset_meta:강력한비밀번호@localhost:5432/superset_meta" -c '\dt' | head
# or
psql 'postgresql://superset_meta:P%40ssw0rd!@localhost:5444/superset_meta' -c '\dt' | head
# ab_user, dashboards, slices 등 Superset 테이블이 보이면 정상
```

만약 이미 `superset_config.py`를 만들기 전에 `superset db upgrade`/`fab create-admin`을 먼저 실행해버렸다면, SQLite(기본적으로 `~/.superset/superset.db` 또는 실행 계정 홈 디렉터리 하위)에 스키마가 생성된 상태입니다. `superset_config.py`로 PostgreSQL을 지정한 뒤 위 3개 명령을 다시 실행해 PostgreSQL 쪽에도 스키마/계정을 새로 만들어 주세요(SQLite 쪽 데이터는 옮겨지지 않고 그냥 버려집니다 — 아직 운영 데이터가 없는 신규 설치라면 문제 없습니다).

> `SUPERSET_SECRET_KEY`는 재시작 시에도 동일해야 합니다. systemd 유닛에 `Environment="SUPERSET_SECRET_KEY=..."`로 고정해 두거나, `superset_config.py`의 `SECRET_KEY` 값 자체를 고정 문자열로 박아 두세요(버전관리에는 올리지 않도록 주의).

### 2.4 실행

개발/테스트용 실행:

```bash
superset run -h 0.0.0.0 -p 8088 --with-threads
```

운영 환경에서는 gunicorn으로 실행하는 것을 권장합니다:

```bash
gunicorn -w 4 -k gevent \
  --timeout 120 \
  -b 0.0.0.0:8088 \
  "superset.app:create_app()"
```

systemd 유닛으로 등록해두면 서버 재부팅 시에도 자동 기동됩니다.

```ini
# /etc/systemd/system/superset.service
[Unit]
Description=Apache Superset
After=network.target

[Service]
User=superset
WorkingDirectory=/opt/superset
Environment="PATH=/opt/superset/venv/bin"
Environment="SUPERSET_CONFIG_PATH=/opt/superset/superset_config.py"
ExecStart=/opt/superset/venv/bin/gunicorn -w 4 -k gevent -b 0.0.0.0:8088 "superset.app:create_app()"
Restart=always

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now superset
```

브라우저에서 `http://<서버IP>:8088` 접속 후 위에서 만든 관리자 계정으로 로그인합니다.

---

## 3. PostgreSQL 데이터베이스 연동

### 3.1 드라이버

`psycopg2`는 1단계에서 이미 설치했습니다. 별도 작업이 필요 없습니다(도커 이미지를 쓰는 경우도 기본 포함).

### 3.2 연결 문자열(SQLAlchemy URI)

```
postgresql://<사용자명>:<비밀번호>@<DB호스트>:5432/<DB이름>
```

SSL을 강제하려면:

```
postgresql://<사용자명>:<비밀번호>@<DB호스트>:5432/<DB이름>?sslmode=require
```

### 3.3 UI에서 연결 추가

1. 관리자 계정으로 로그인 후 우측 상단 **Settings → Data: Database Connections** 이동
2. **+ DATABASE** 클릭
3. 데이터베이스 종류에서 **PostgreSQL** 선택 (동적 폼 지원 — Redshift, MySQL, BigQuery와 함께 신규 UI 지원 대상)
4. Host, Port(기본 5432), Database name, Username, Password 입력
5. **Test Connection** 클릭 → "Connection looks good!" 확인
6. **Connect** 클릭하여 저장

동적 폼이 아닌 레거시 방식(직접 SQLAlchemy URI 입력)을 쓰고 싶다면 "SQLALCHEMY URI" 필드에 위 3.2 문자열을 그대로 입력하면 됩니다.

### 3.4 추가 옵션 (필요 시)

- **스키마 지정**: 연결 후 Sources → Tables → Edit 에서 schema 파라미터 설정
- **SSL 인증서**: Database 편집 화면의 **Extra**(JSON) 필드에 다음과 같이 추가

```json
{
  "engine_params": {
    "connect_args": {
      "sslmode": "require",
      "sslrootcert": "/path/to/root.pem"
    }
  }
}
```

- **커넥션 풀 크기 등**은 `engine_params`를 통해 SQLAlchemy `create_engine`에 그대로 전달됩니다.

---

## 4. MCP 서버 구성 (Superset 내장 MCP)

Superset 5.0+에는 **Model Context Protocol(MCP) 서버가 기본 내장**되어 있습니다. Claude, ChatGPT 등 MCP 호환 클라이언트가 대시보드 조회, 데이터셋 쿼리, SQL 실행, 차트 생성 등을 수행할 수 있습니다.

### 4.1 사전 조건

```bash
pip install fastmcp
```

- Superset 5.0 이상, Python 3.11 이상 필요

### 4.2 MCP 서버 기동

Superset 웹 서버와는 별도 프로세스로 실행합니다.

```bash
export SUPERSET_SECRET_KEY=jYqWJpZbV9S5rBH54K61EFpJlpmXBbLVxQtpfA4hC96sWqVI7pCNVH6D
superset mcp run --host 0.0.0.0 --port 5008
```

| 옵션 | 기본값 | 설명 |
|---|---|---|
| `--host` | `127.0.0.1` | 바인딩 호스트 |
| `--port` | `5008` | 바인딩 포트 |
| `--debug` | off | 디버그 로깅 |

엔드포인트: `http://<host>:<port>/mcp`

### 4.3 개발 모드 (인증 없음) — 테스트용

`superset_config.py`에 추가:

```python
MCP_AUTH_ENABLED = False
MCP_DEV_USERNAME = "admin"   # 실제 존재하는 Superset 사용자명
```

이 모드는 **내부망/테스트 전용**입니다. 외부에 노출되는 서버에는 절대 사용하지 마세요.

### 4.4 운영 환경 — JWT 인증

IdP(OAuth2/OIDC)가 JWKS 엔드포인트를 제공하는 경우:

```python
MCP_AUTH_ENABLED = True
MCP_JWT_ALGORITHM = "RS256"
MCP_JWKS_URI = "https://your-idp.example.com/.well-known/jwks.json"
MCP_JWT_ISSUER = "https://your-idp.example.com/"
MCP_JWT_AUDIENCE = "your-superset-instance"
```

고정 시크릿 기반(HS256)인 경우:

```python
import os

MCP_AUTH_ENABLED = True
MCP_JWT_ALGORITHM = "HS256"
MCP_JWT_SECRET = os.environ.get("MCP_JWT_SECRET")
MCP_JWT_ISSUER = "your-issuer"
MCP_JWT_AUDIENCE = "your-audience"
```

JWT의 `sub`(또는 `email`/`username`) 클레임 값이 Superset `ab_user` 테이블의 `username`과 일치해야 합니다.

특정 스코프만 허용하려면:

```python
MCP_REQUIRED_SCOPES = ["mcp:read", "mcp:write"]
```

### 4.5 리버스 프록시(Nginx) + TLS 구성

```nginx
server {
    listen 443 ssl;
    server_name superset.example.com;
    ssl_certificate     /path/to/cert.pem;
    ssl_certificate_key /path/to/key.pem;

    # Superset 웹 UI
    location / {
        proxy_pass http://127.0.0.1:8088;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }

    # MCP 엔드포인트
    location /mcp {
        proxy_pass http://127.0.0.1:5008/mcp;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header Authorization $http_authorization;
    }
}
```

리버스 프록시 뒤에 있다면 `superset_config.py`에 공개 URL도 지정해야 차트 미리보기/SQL Lab 링크가 올바르게 생성됩니다:

```python
MCP_SERVICE_URL = "https://superset.example.com"
```

### 4.6 systemd로 MCP 서버 상시 실행

```ini
# /etc/systemd/system/superset-mcp.service
[Unit]
Description=Apache Superset MCP Server
After=network.target superset.service

[Service]
User=superset
WorkingDirectory=/opt/superset
Environment="PATH=/opt/superset/venv/bin"
Environment="SUPERSET_CONFIG_PATH=/opt/superset/superset_config.py"
ExecStart=/opt/superset/venv/bin/superset mcp run --host 0.0.0.0 --port 5008
Restart=always

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now superset-mcp
```

### 4.7 Open WebUI / Claude 등 MCP 클라이언트 연결

**Claude Desktop** (`~/.config/Claude/claude_desktop_config.json` — Linux 클라이언트 기준):

```json
{
  "mcpServers": {
    "superset": {
      "url": "http://localhost:5008/mcp"
    }
  }
}
```

JWT 인증이 켜져 있다면:

```json
{
  "mcpServers": {
    "superset": {
      "command": "npx",
      "args": [
        "-y",
        "mcp-remote@latest",
        "http://your-superset-host:5008/mcp",
        "--header",
        "Authorization: Bearer YOUR_TOKEN"
      ]
    }
  }
}
```

**Open WebUI**의 경우 (이전에 구성하신 Whpg-Mcp/Metabase-Mcp와 동일한 방식):

- Settings > Admin > Integrations > External Tool Servers > **+ Add Connection**
- Type: **MCP (Streamable HTTP)**
- URL: `http://<superset-host>:5008/mcp`
- 인증: 개발 모드면 "없음", JWT 사용 시 Bearer 토큰 입력

**직접 HTTP 테스트**:

```bash
curl -X POST http://localhost:5008/mcp \
  -H 'Content-Type: application/json' \
  -H 'Authorization: Bearer YOUR_JWT_TOKEN' \
  -d '{"jsonrpc": "2.0", "method": "tools/list", "id": 1}'
```

### 4.8 접근 제어(RBAC)

MCP 서버는 Superset의 RBAC를 그대로 따릅니다. 웹 UI에서 권한이 없는 데이터는 MCP를 통해서도 접근할 수 없습니다.

| 툴 그룹 | 필요 권한 |
|---|---|
| `list_charts`, `get_chart_data`, `generate_chart` | Chart `can_read`/`can_write` |
| `list_dashboards`, `generate_dashboard` | Dashboard `can_read`/`can_write` |
| `list_datasets`, `create_virtual_dataset` | Dataset `can_read`/`can_write` |
| `execute_sql` | SQLLab `can_execute_sql_query` |

신뢰된 내부망 테스트 목적이 아니라면 다음 설정은 켜두는 것이 안전합니다(기본값 `True`):

```python
MCP_RBAC_ENABLED = True
```

모든 MCP 툴 호출은 **Settings → Action Log**에서 감사 로그로 확인할 수 있습니다.

---

## 5. 설치/기동 트러블슈팅

실제 배포 중 자주 만나는 3가지 문제와 해결책입니다.

### 5.1 `ModuleNotFoundError: No module named 'pkg_resources'`

**증상**: `superset db upgrade` 또는 gunicorn 기동 시 `superset.config` 임포트 단계에서 `pkg_resources` 모듈을 찾을 수 없다는 에러.

**원인**: 2026년 2월 배포된 **Setuptools 82.0**부터 `pkg_resources` 모듈이 완전히 제거되었습니다. Superset(정확히는 Flask-AppBuilder 등 하위 의존성)이 아직 `pkg_resources`를 사용하므로, venv에 82 이상 버전의 setuptools가 깔리면 이 에러가 발생합니다.

**해결**:

```bash
source /opt/superset/venv/bin/activate
pip install 'setuptools<82'
sudo systemctl restart superset
```

`pip show setuptools`로 현재 버전을 확인하고, 82 이상이면 위 명령으로 낮추세요.

### 5.2 `INFO:superset.utils.screenshots:No PIL installation found`

**증상**: 에러는 아니고 정보성 로그지만, 방치하면 **차트 스크린샷, PDF 내보내기, Alerts & Reports 썸네일** 기능이 동작하지 않습니다.

**해결**:

```bash
source /opt/superset/venv/bin/activate
pip install Pillow
sudo systemctl restart superset
```

> 1단계 설치 명령에 `pillow`를 포함해두면 처음부터 이 문제를 피할 수 있습니다.

### 5.3 `TypeError: SupersetMetastoreCache.__init__() got an unexpected keyword argument 'ignore_delete_many_errors'`

**증상**: `superset db upgrade` 또는 앱 기동 시 캐시 초기화(`configure_cache`) 단계에서 실패.

**원인**: 최근 Flask-Caching 릴리스가 `CACHE_IGNORE_ERRORS` 설정을 모든 캐시 백엔드에 공통 적용하도록 바뀌면서, 캐시 팩토리가 모든 커스텀 캐시 클래스에 `ignore_delete_many_errors`라는 새 키워드 인자를 자동으로 넘깁니다. 그런데 `FILTER_STATE_CACHE_CONFIG`/`EXPLORE_FORM_DATA_CACHE_CONFIG`를 지정하지 않으면 기본으로 쓰이는 Superset 내장 폴백 캐시 `SupersetMetastoreCache`가 이 인자를 받지 못해 깨집니다. 즉 **pip가 설치한 flask-caching 버전이 설치된 Superset 버전보다 최신이라 생기는 호환성 버그**입니다.

**해결 A — flask-caching 다운그레이드(원인 제거)**:

```bash
source /opt/superset/venv/bin/activate
pip install "Flask-Caching<2.3"
# 그래도 안 되면: pip install "Flask-Caching==2.1.0"
superset db upgrade
```

**해결 B — superset_config.py에서 폴백 캐시를 명시적으로 우회** (2.2절 예시에 이미 반영됨):

```python
FILTER_STATE_CACHE_CONFIG = {"CACHE_TYPE": "SimpleCache", "CACHE_DEFAULT_TIMEOUT": 86400}
EXPLORE_FORM_DATA_CACHE_CONFIG = {"CACHE_TYPE": "SimpleCache", "CACHE_DEFAULT_TIMEOUT": 86400}
```

해결 A가 근본 해결이고, 해결 B는 버전 문제와 무관하게 즉시 우회하는 방법입니다. 운영 환경에서는 결국 Redis(`CACHE_TYPE: "RedisCache"`)로 옮기는 것을 권장합니다.

---

## 6. 체크리스트

- [ ] OS 의존성(`libpq-dev` 포함) 설치 완료
- [ ] 가상환경 내 `apache_superset`, `psycopg2`, `fastmcp`, `Pillow`, `setuptools<82` 설치 완료
- [ ] `superset_config.py`에 `SQLALCHEMY_DATABASE_URI`(PostgreSQL), `FILTER_STATE_CACHE_CONFIG`, `EXPLORE_FORM_DATA_CACHE_CONFIG` 지정 완료
- [ ] `superset db upgrade` / `fab create-admin` / `superset init` 실행 완료
- [ ] Superset 웹 UI(8088) 정상 기동 및 관리자 로그인 확인
- [ ] PostgreSQL 데이터소스 연결 테스트("Connection looks good!") 통과
- [ ] MCP 서버(5008) 기동 및 `/mcp` 엔드포인트 응답 확인
- [ ] 운영 배포라면 `MCP_AUTH_ENABLED = True` + JWT 설정, TLS 리버스 프록시 적용
- [ ] systemd 유닛 등록으로 재부팅/장애 시 자동 재기동 확인

---

## 참고 자료

- [PyPI 설치 가이드 — Apache Superset](https://superset.apache.org/admin-docs/installation/pypi/)
- [Docker Compose 설치 — Apache Superset](https://superset.apache.org/admin-docs/installation/docker-compose/)
- [Connecting to Databases — Apache Superset](https://superset.apache.org/docs/configuration/databases/)
- [Configuring Superset (superset_config.py) — Apache Superset](https://superset.apache.org/admin-docs/configuration/configuring-superset/)
- [Caching — Apache Superset](https://superset.apache.org/admin-docs/configuration/cache/)
- [MCP Server Deployment & Authentication — Apache Superset](https://superset.apache.org/admin-docs/configuration/mcp-server/)
- [Using AI with Superset — Apache Superset](https://superset.apache.org/user-docs/using-superset/using-ai-with-superset)
