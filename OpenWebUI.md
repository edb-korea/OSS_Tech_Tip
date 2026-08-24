# uv 기반 Open WebUI 구성 가이드

**대상 구성**: uv로 실행, PostgreSQL 채팅 DB, Ollama 연동, Claude(Anthropic) 연동, 로컬 MCP 연동

---

## 0. 아키텍처 개요

```
┌─────────────┐      DATABASE_URL       ┌──────────────┐
│  Open WebUI │ ───────────────────────▶│ PostgreSQL   │
│  (uvx 실행)  │                          └──────────────┘
│             │      OLLAMA_BASE_URL     ┌──────────────┐
│             │ ───────────────────────▶│ Ollama       │
│             │                          └──────────────┘
│             │      Anthropic API       ┌──────────────┐
│             │ ───────────────────────▶│ Claude API   │
│             │                          └──────────────┘
│             │      OpenAPI (HTTP)      ┌──────────────┐
│             │ ───────────────────────▶│ mcpo (MCP    │
└─────────────┘                          │  → OpenAPI)  │
                                          └──────┬───────┘
                                                 │ stdio
                                          ┌──────▼───────┐
                                          │ 로컬 MCP 서버 │
                                          └──────────────┘
```

로컬 MCP 서버는 대부분 stdio로 통신하기 때문에, Open WebUI(웹 서버)가 직접 붙을 수 없습니다. 그래서 `mcpo`라는 프록시로 MCP를 OpenAPI(REST)로 감싸서 연결합니다.

---

## 1. 사전 준비

### 1-1. uv 설치

```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
```

### 1-2. Python 버전

Open WebUI는 **Python 3.11 / 3.12**만 지원합니다 (3.13 미지원). `uvx --python 3.11`로 고정하는 것을 권장합니다.

---

## 2. PostgreSQL 준비

이미 운영 중인 PostgreSQL이 있다면 이 단계는 건너뛰고 접속 정보만 준비하면 됩니다. 없다면 Docker로 간단히 띄웁니다.

```bash
docker run -d \
  --name openwebui-postgres \
  -e POSTGRES_USER=openwebui \
  -e POSTGRES_PASSWORD=CHANGE_ME \
  -e POSTGRES_DB=openwebui \
  -p 5432:5432 \
  -v openwebui-pgdata:/var/lib/postgresql/data \
  --restart always \
  postgres:16
```

접속 문자열 형식:

```
postgresql://openwebui:CHANGE_ME@localhost:5432/openwebui
```

> 비밀번호에 특수문자가 있으면 URL 인코딩해야 합니다.

---

## 3. Open WebUI 실행 (uv + Postgres)

### 3-1. 중요: postgres extra 필요

`uvx open-webui@latest`로 그냥 실행하면 **psycopg2 드라이버가 설치되어 있지 않아** 아래와 같은 에러가 납니다.

```
Failed to initialize the database connection: Postgres driver not installed!
```

따라서 `--from` 옵션으로 `postgres` extra를 포함해서 실행해야 합니다.

### 3-2. 실행 명령

```bash
DATA_DIR=~/.open-webui \
DATABASE_URL="postgresql://openwebui:CHANGE_ME@localhost:5432/openwebui" \
WEBUI_SECRET_KEY="$(openssl rand -hex 32)" \
uvx --python 3.11 --from "open-webui[postgres]" open-webui serve
```

| 환경변수 | 설명 |
|---|---|
| `DATA_DIR` | 업로드 파일 등 데이터 저장 위치 (DB 자체는 Postgres에 저장) |
| `DATABASE_URL` | PostgreSQL 접속 문자열 |
| `WEBUI_SECRET_KEY` | 세션/토큰 암호화 키. **고정값으로 관리**하지 않으면 재시작마다 로그아웃되고, MCP OAuth 토큰도 깨집니다 |

실제 운영에서는 `WEBUI_SECRET_KEY`를 한 번 생성해서 `.env` 파일 등에 고정 저장하세요.

### 3-3. 백그라운드 상시 실행 (systemd)

`uvx`는 포그라운드 프로세스이므로, EC2 등에서 상시 서비스로 돌리려면 systemd 유닛으로 등록하는 것을 권장합니다.

`/etc/systemd/system/open-webui.service`:

```ini
[Unit]
Description=Open WebUI (uv, PostgreSQL backend)
After=network.target postgresql.service

[Service]
Type=simple
User=ec2-user
WorkingDirectory=/home/ec2-user/OpenWebUI
Environment="DATA_DIR=/home/ec2-user/OpenWebUI/.open-webui"
Environment="VECTOR_DB=pgvector"
Environment="DATABASE_URL=postgresql://openwebui:change_me_strong_password@localhost:5444/openwebui"
Environment="WEBUI_SECRET_KEY=dd27cd9747e3526233ca607f70ae470a2b3e5fddc9e4e7d87e9d33552b2b7343"
Environment="OLLAMA_BASE_URL=http://localhost:11434"
Environment="PATH=/home/ec2-user/.local/bin:/usr/bin:/bin"
Environment="HOST=0.0.0.0"
Environment="PORT=8080"

ExecStart=/home/ec2-user/.local/bin/uvx --python 3.11 --from "open-webui[postgres]" open-webui serve
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

브라우저에서 `http://<서버IP>:8080` 접속 → 첫 계정이 관리자 계정이 됩니다.

---

## 4. Ollama 연동

Open WebUI는 기본적으로 로컬 Ollama(`http://localhost:11434`)에 자동 접속을 시도합니다. 별도 서버라면 admin 설정에서 URL을 지정합니다.

1. **⚙️ Admin Settings → Connections → Ollama**
2. URL 입력 (같은 머신: 기본값 그대로, 다른 서버: `http://<ollama-host>:11434`)
3. 저장 후 모델 다운로드/선택

> DeepSeek-R1, Qwen3 같은 reasoning 모델을 쓸 경우 `ollama serve --reasoning-parser deepseek_r1` 옵션으로 실행하면 `<think>` 블록이 접힌 형태로 표시됩니다.

---

## 5. Claude(Anthropic) 연동

Open WebUI는 Anthropic의 OpenAI 호환 엔드포인트를 네이티브로 지원합니다. 별도 미들웨어 없이 바로 연결됩니다.

1. [console.anthropic.com](https://console.anthropic.com/)에서 API 키 발급
2. **⚙️ Admin Settings → Connections → OpenAI → ➕ Add Connection**
3. 아래 값 입력:

| 항목 | 값 |
|---|---|
| URL | `https://api.anthropic.com/v1` |
| API Key | 발급받은 Anthropic API 키 |

4. 저장하면 Claude 모델이 자동으로 목록에 나타납니다. 필요하면 **Model IDs (Filter)**에 아래 ID만 허용할 수 있습니다.

| 모델 | ID |
|---|---|
| Claude Opus 5 | `claude-opus-5` |
| Claude Sonnet 5 | `claude-sonnet-5` |
| Claude Haiku 4.5 | `claude-haiku-4-5` |

> 스트리밍, 멀티턴, 툴 콜링까지 모두 동작합니다. 다만 이 방식은 Anthropic이 "평가/테스트용"으로 안내하는 호환 레이어이며, PDF 처리·citations·extended thinking·prompt caching 같은 Claude 네이티브 전용 기능까지 쓰려면 Pipe Function이나 LiteLLM 프록시로 `/v1/messages` 네이티브 API를 직접 붙여야 합니다.

---

## 6. 로컬 MCP 연동

로컬 MCP 서버(주로 stdio 방식)를 붙이는 방법은 두 가지입니다.

### 방법 A — mcpo 프록시 (로컬 stdio MCP 서버에 권장)

`mcpo`는 stdio/SSE 기반 MCP 서버를 REST(OpenAPI)로 감싸주는 Open WebUI 공식 프록시입니다.

**단일 도구 예시** (시간 조회 MCP 서버):

```bash
uvx mcpo --port 8000 --api-key "top-secret" -- uvx mcp-server-time --local-timezone=Asia/Seoul
```

**여러 MCP 서버를 한 번에 (config 파일 방식, 권장)**

`mcpo-config.json`:

```json
{
  "mcpServers": {
    "time": {
      "command": "uvx",
      "args": ["mcp-server-time", "--local-timezone=Asia/Seoul"]
    },
    "filesystem": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "/home/ec2-user/data"]
    }
  }
}
```

```bash
uvx mcpo --config mcpo-config.json --api-key "top-secret" --hot-reload
```

이제 각 도구는 `http://localhost:8000/time`, `http://localhost:8000/filesystem` 처럼 개별 경로로 노출되고, `/docs`에서 Swagger UI로 바로 테스트할 수 있습니다.

**Open WebUI에 연결**

1. **⚙️ Admin Settings → Settings → Tools** (또는 개인 설정 → Integrations, 권한에 따라 다름)
2. **+ Add Tool Server** 클릭
3. URL에 `http://localhost:8000/time` (도구별로 각각 등록) 입력
4. API Key에 `top-secret` 입력 후 저장

> 상시 서비스로 돌리려면 3-3과 같은 방식으로 `mcpo`도 systemd 유닛으로 등록해두는 것을 권장합니다.

### 방법 B — 네이티브 MCP (Streamable HTTP 서버를 이미 갖고 있는 경우)

Open WebUI v0.6.31+부터 **Streamable HTTP** 방식 MCP 서버는 프록시 없이 직접 연결할 수 있습니다. 단, **stdio/SSE는 지원하지 않으므로** 로컬 stdio MCP 서버라면 방법 A(mcpo)를 써야 합니다.

1. **⚙️ Admin Settings → Integrations → + (Add Server)**
2. **Type**: `MCP (Streamable HTTP)` 선택 (⚠️ `OpenAPI`로 잘못 선택하면 UI가 멈춤)
3. Server URL, 인증 방식(None / Bearer / OAuth 2.1) 입력 후 저장

로컬 서버이고 인증이 필요 없다면 **Auth: None**으로 두세요. `Bearer`를 키 없이 선택하면 빈 토큰이 전송되어 연결이 거부됩니다.

MCP 서버 등록은 **관리자만 가능**하며, 일반 사용자에게 권한을 주고 싶다면 Access Control로 특정 사용자/그룹에 스코프를 지정합니다.

---

## 7. 체크리스트

- [ ] PostgreSQL 접속 정보 확인, `DATABASE_URL` 설정
- [ ] `uvx --from "open-webui[postgres]"`로 실행 (postgres extra 필수)
- [ ] `WEBUI_SECRET_KEY` 고정값 설정 (재시작/OAuth 토큰 보존)
- [ ] systemd로 상시 서비스화
- [ ] Ollama URL 연결 확인
- [ ] Anthropic API 키로 Claude 연결 확인
- [ ] mcpo로 로컬 MCP 서버 프록시 후 Tool Server 등록

---

## 참고 자료

- [Quick Start / Open WebUI](https://docs.openwebui.com/getting-started/quick-start/)
- [Anthropic (Claude) / Open WebUI](https://docs.openwebui.com/getting-started/quick-start/connect-a-provider/starting-with-anthropic)
- [Ollama / Open WebUI](https://docs.openwebui.com/getting-started/quick-start/connect-a-provider/starting-with-ollama)
- [MCP Support (mcpo) / Open WebUI](https://docs.openwebui.com/features/extensibility/plugin/tools/openapi-servers/mcp/)
- [Model Context Protocol (native) / Open WebUI](https://docs.openwebui.com/features/extensibility/mcp)
- [open-webui/mcpo (GitHub)](https://github.com/open-webui/mcpo)
- [Environment Variable Configuration / Open WebUI](https://docs.openwebui.com/reference/env-configuration/)
- [Issue #17048: uv + postgres psycopg2 missing](https://github.com/open-webui/open-webui/issues/17048)
