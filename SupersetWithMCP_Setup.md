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

---

## 2. 가상환경 생성 및 Superset 설치

```bash
python3 -m venv /opt/superset/venv
source /opt/superset/venv/bin/activate

pip install --upgrade pip setuptools

# Superset 본체 + PostgreSQL 드라이버 + 운영에 필요한 패키지
pip install apache_superset psycopg2 gunicorn gevent flask_cors pillow rich cachetools
```

> `psycopg2`가 PostgreSQL용 Python DB 드라이버입니다. 메타데이터 DB든, 연결 대상 데이터 소스든 PostgreSQL을 쓴다면 반드시 필요합니다.

### 2.1 환경 변수 및 초기화

```bash
export SUPERSET_SECRET_KEY=$(openssl rand -base64 42)
export FLASK_APP=superset

# 메타데이터 DB 스키마 초기화
superset db upgrade

# 관리자 계정 생성
superset fab create-admin \
  --username admin \
  --firstname Admin \
  --lastname User \
  --email admin@example.com \
  --password '강력한비밀번호'

# 기본 역할/권한 생성
superset init
```

`SUPERSET_SECRET_KEY`는 재시작 시에도 동일해야 하므로, `superset_config.py`나 systemd 서비스의 환경변수 파일에 고정해 두는 것이 좋습니다.

### 2.2 실행

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

## 5. 체크리스트

- [ ] OS 의존성(`libpq-dev` 포함) 설치 완료
- [ ] 가상환경 내 `apache_superset`, `psycopg2`, `fastmcp` 설치 완료
- [ ] `superset db upgrade` / `fab create-admin` / `superset init` 실행 완료
- [ ] Superset 웹 UI(8088) 정상 기동 및 관리자 로그인 확인
- [ ] PostgreSQL 데이터베이스 연결 테스트("Connection looks good!") 통과
- [ ] MCP 서버(5008) 기동 및 `/mcp` 엔드포인트 응답 확인
- [ ] 운영 배포라면 `MCP_AUTH_ENABLED = True` + JWT 설정, TLS 리버스 프록시 적용
- [ ] systemd 유닛 등록으로 재부팅/장애 시 자동 재기동 확인

---

## 참고 자료

- [PyPI 설치 가이드 — Apache Superset](https://superset.apache.org/admin-docs/installation/pypi/)
- [Docker Compose 설치 — Apache Superset](https://superset.apache.org/admin-docs/installation/docker-compose/)
- [Connecting to Databases — Apache Superset](https://superset.apache.org/docs/configuration/databases/)
- [MCP Server Deployment & Authentication — Apache Superset](https://superset.apache.org/admin-docs/configuration/mcp-server/)
- [Using AI with Superset — Apache Superset](https://superset.apache.org/user-docs/using-superset/using-ai-with-superset)
