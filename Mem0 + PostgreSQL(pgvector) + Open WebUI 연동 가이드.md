# Mem0 + PostgreSQL(pgvector) + Open WebUI 연동 가이드

Python 기반으로 Mem0를 설치하고, 벡터 저장소로 PostgreSQL(pgvector)을 붙이고, LLM을 연동한 뒤, Open WebUI에서 메모리 레이어로 사용하는 전체 과정을 정리했습니다.

## 0. 전체 아키텍처

```
[Open WebUI] --(Filter Pipeline)--> [Mem0 (Memory/AsyncMemory)]
                                          |         |
                                     [LLM API]  [Embedder API]
                                          |
                                   [PostgreSQL + pgvector]
```

- Open WebUI가 사용자 메시지를 받으면, 연결된 Pipeline(필터)이 응답 생성 전에 Mem0에서 관련 기억을 검색해 시스템 프롬프트에 주입합니다.
- 응답이 끝나면 해당 턴의 대화를 Mem0에 `add()`하여 사실을 추출하고 pgvector에 저장합니다.
- Mem0 내부적으로 LLM(사실 추출용)과 Embedder(벡터화용)를 별도로 호출합니다. 메인 대화 응답을 생성하는 LLM과 Mem0가 내부적으로 쓰는 LLM은 같은 모델일 필요가 없습니다(오히려 추출용은 더 저렴한 모델을 쓰는 것이 일반적).

---

## 1. PostgreSQL + pgvector 준비

가장 간단한 방법은 Docker로 pgvector가 포함된 이미지를 띄우는 것입니다.

```yaml
# docker-compose.yml
services:
  postgres:
    image: pgvector/pgvector:pg16
    container_name: mem0-postgres
    restart: unless-stopped
    environment:
      POSTGRES_USER: mem0
      POSTGRES_PASSWORD: mem0pass
      POSTGRES_DB: mem0db
    ports:
      - "5432:5432"
    volumes:
      - pgdata:/var/lib/postgresql/data

volumes:
  pgdata:
```

```bash
docker compose up -d
```

컨테이너가 뜨면 pgvector 확장을 활성화합니다(최초 1회).

```bash
docker exec -it mem0-postgres psql -U mem0 -d mem0db -c "CREATE EXTENSION IF NOT EXISTS vector;"
```

이미 운영 중인 PostgreSQL이 있다면 `CREATE EXTENSION IF NOT EXISTS vector;`만 해당 DB에서 실행하면 됩니다(단, pgvector 확장이 서버에 설치되어 있어야 함 — 관리형 서비스라면 지원 여부를 먼저 확인).

---

## 2. Mem0 Python 설치

```bash
python -m venv .venv
source .venv/bin/activate   # Windows는 .venv\Scripts\activate

pip install mem0ai
# 하이브리드 검색(BM25 키워드 + 엔티티 부스팅)까지 쓰려면:
pip install "mem0ai[nlp]"
python -m spacy download en_core_web_sm

# pgvector 드라이버
pip install "psycopg[binary]"
```

---

## 3. Mem0 설정: pgvector + LLM + Embedder

`Memory.from_config()`에 `vector_store`, `llm`, `embedder` 세 블록을 지정합니다. 세 컴포넌트를 서로 다른 provider로 자유롭게 조합할 수 있습니다.

### 3-1. OpenAI LLM + pgvector 조합 예시

```python
import os
from mem0 import Memory

os.environ["OPENAI_API_KEY"] = "sk-xxxx"

config = {
    "vector_store": {
        "provider": "pgvector",
        "config": {
            "user": "mem0",
            "password": "mem0pass",
            "host": "127.0.0.1",
            "port": "5432",
            "dbname": "mem0db",
            "collection_name": "memories",
            "embedding_model_dims": 1536,   # 임베딩 모델 차원과 반드시 일치
            "hnsw": True,                   # 검색 속도 개선(기본 True)
        },
    },
    "llm": {
        "provider": "openai",
        "config": {
            "model": "gpt-5-mini",   # 추출 전용이므로 저렴한 모델 권장
            "temperature": 0.1,
            "max_tokens": 1500,
        },
    },
    "embedder": {
        "provider": "openai",
        "config": {
            "model": "text-embedding-3-small",
        },
    },
}

m = Memory.from_config(config)

# 저장
m.add(
    [
        {"role": "user", "content": "나는 PostgreSQL과 pgvector로 RAG 시스템을 구축 중이야."},
        {"role": "assistant", "content": "네, pgvector 기반 구축을 도와드릴게요."},
    ],
    user_id="paul",
)

# 검색
result = m.search("내가 어떤 DB를 쓰고 있다고 했지?", user_id="paul", top_k=3)
print(result)
```

### 3-2. 로컬 LLM(Ollama) + pgvector 조합 (API 비용 없이 운영)

```python
config = {
    "vector_store": {
        "provider": "pgvector",
        "config": {
            "user": "mem0",
            "password": "mem0pass",
            "host": "127.0.0.1",
            "port": "5432",
            "dbname": "mem0db",
            "embedding_model_dims": 768,   # 사용하는 로컬 임베딩 모델 차원에 맞춰 변경
        },
    },
    "llm": {
        "provider": "ollama",
        "config": {
            "model": "llama3.1:8b",
            "ollama_base_url": "http://localhost:11434",
        },
    },
    "embedder": {
        "provider": "ollama",
        "config": {
            "model": "nomic-embed-text",
            "ollama_base_url": "http://localhost:11434",
        },
    },
}
```

> 주의: `embedding_model_dims`는 실제 임베딩 모델이 출력하는 벡터 차원과 정확히 맞아야 합니다(예: `text-embedding-3-small`=1536, `nomic-embed-text`=768). 다르면 pgvector 컬럼 생성/검색 단계에서 에러가 납니다.

### 3-3. 연결 문자열(connection string) 방식

개별 host/port 대신 커넥션 스트링을 쓰거나 커넥션 풀을 재사용하고 싶다면:

```python
"vector_store": {
    "provider": "pgvector",
    "config": {
        "connection_string": "postgresql://mem0:mem0pass@127.0.0.1:5432/mem0db",
        "sslmode": "prefer",
        "collection_name": "memories",
        "embedding_model_dims": 1536,
    },
}
```

우선순위는 `connection_pool` > `connection_string` > 개별 필드(user/password/host/port) 순입니다.

---

## 4. Open WebUI 연동

Open WebUI 자체에는 Mem0 내장 지원이 없으므로, **Pipelines(플러그인 프레임워크)**를 통해 Filter 형태로 붙입니다. 흐름은: 사용자 메시지 수신(`inlet`) → Mem0 검색 결과를 시스템 프롬프트에 주입 → LLM 응답 생성 → 응답 완료 후(`outlet`) 해당 턴을 Mem0에 저장.

### 4-1. Pipelines 서버 구동

```yaml
# docker-compose.pipelines.yml
services:
  pipelines:
    image: ghcr.io/open-webui/pipelines:main
    container_name: owui-pipelines
    ports:
      - "9099:9099"
    volumes:
      - pipelines_data:/app/pipelines
    restart: unless-stopped

volumes:
  pipelines_data:
```

```bash
docker compose -f docker-compose.pipelines.yml up -d
```

Open WebUI 관리자 설정(Settings → Admin Settings → Connections)에서 OpenAI 호환 엔드포인트로 `http://<pipelines-host>:9099`와 Pipelines API 키(기본값 `0p3n-w3bu!`)를 등록합니다.

### 4-2. pgvector 기반 커스텀 Filter Pipeline 작성

공개된 `mem0-owui` 예제는 Qdrant를 하드코딩하고 있으므로, pgvector를 쓰려면 아래처럼 직접 Filter를 작성합니다. `AsyncMemory`를 사용해 응답 지연을 최소화합니다.

```python
# mem0_pgvector_filter.py  (Pipelines "Upload" 메뉴로 업로드)
from typing import List, Optional
from pydantic import BaseModel
from mem0 import AsyncMemory


class Pipeline:
    class Valves(BaseModel):
        pipelines: List[str] = ["*"]
        priority: int = 0

        pg_host: str = "host.docker.internal"
        pg_port: str = "5432"
        pg_user: str = "mem0"
        pg_password: str = "mem0pass"
        pg_dbname: str = "mem0db"

        llm_provider: str = "openai"
        llm_model: str = "gpt-5-mini"
        llm_api_key: str = "sk-xxxx"

        embedder_provider: str = "openai"
        embedder_model: str = "text-embedding-3-small"
        embedding_dims: int = 1536

        top_k: int = 5

    def __init__(self):
        self.type = "filter"
        self.name = "Mem0 (pgvector) Memory Filter"
        self.valves = self.Valves()
        self.memory: Optional[AsyncMemory] = None

    async def on_startup(self):
        config = {
            "vector_store": {
                "provider": "pgvector",
                "config": {
                    "host": self.valves.pg_host,
                    "port": self.valves.pg_port,
                    "user": self.valves.pg_user,
                    "password": self.valves.pg_password,
                    "dbname": self.valves.pg_dbname,
                    "embedding_model_dims": self.valves.embedding_dims,
                },
            },
            "llm": {
                "provider": self.valves.llm_provider,
                "config": {
                    "model": self.valves.llm_model,
                    "api_key": self.valves.llm_api_key,
                },
            },
            "embedder": {
                "provider": self.valves.embedder_provider,
                "config": {
                    "model": self.valves.embedder_model,
                    "api_key": self.valves.llm_api_key,
                },
            },
        }
        self.memory = await AsyncMemory.from_config(config)

    async def on_shutdown(self):
        pass

    async def inlet(self, body: dict, user: Optional[dict] = None) -> dict:
        """LLM 호출 전: 관련 기억을 검색해 system 메시지에 주입"""
        user_id = (user or {}).get("id", "default_user")
        messages = body.get("messages", [])
        if not messages:
            return body

        last_user_msg = messages[-1]["content"]
        results = await self.memory.search(
            query=last_user_msg, user_id=user_id, top_k=self.valves.top_k
        )
        memories = results.get("results", [])
        if memories:
            memory_text = "\n".join(f"- {m['memory']}" for m in memories)
            system_note = {
                "role": "system",
                "content": f"다음은 이 사용자에 대해 기억하고 있는 정보입니다:\n{memory_text}",
            }
            body["messages"] = [system_note] + messages

        return body

    async def outlet(self, body: dict, user: Optional[dict] = None) -> dict:
        """LLM 응답 후: 이번 턴 대화를 Mem0에 저장(비동기, 응답 지연에 영향 없음)"""
        user_id = (user or {}).get("id", "default_user")
        messages = body.get("messages", [])
        if len(messages) >= 2:
            turn = messages[-2:]  # 마지막 user+assistant 메시지 쌍
            await self.memory.add(turn, user_id=user_id)
        return body
```

업로드 후 순서:

1. Open WebUI → Settings → Admin Settings → Pipelines → **Upload** 로 `mem0_pgvector_filter.py` 업로드
2. Valves 항목에서 `pg_host`, `pg_password`, `llm_api_key` 등 실제 값으로 채움
3. 적용할 모델/파이프라인을 선택해 Filter를 활성화
4. 채팅 테스트: 사용자 정보를 말한 뒤 새 세션(또는 새 대화)에서 물어봐서 기억이 반영되는지 확인

> 참고: Docker로 띄운 Pipelines 컨테이너에서 호스트의 PostgreSQL에 접근하려면 `host.docker.internal`(Mac/Windows) 또는 컨테이너 네트워크를 같이 묶는 방식(Linux는 `--network`로 postgres 컨테이너와 동일 네트워크에 두거나 postgres 서비스명 사용)을 써야 합니다.

---

## 5. 동작 확인 체크리스트

- [ ] `psql`로 접속해 `\dx`로 vector 확장이 설치됐는지 확인
- [ ] `m.add(...)` 실행 후 `SELECT count(*) FROM memories;` (또는 지정한 `collection_name` 테이블)로 실제 row가 쌓이는지 확인
- [ ] `embedding_model_dims` 값이 임베딩 모델의 실제 출력 차원과 일치하는지 확인 (불일치 시 insert 단계 에러)
- [ ] Open WebUI에서 Pipeline 활성화 후 로그(`docker logs owui-pipelines`)에서 inlet/outlet 호출이 찍히는지 확인
- [ ] 다른 `user_id`로 검색했을 때 서로의 기억이 섞이지 않는지 확인(멀티유저 격리)

## 6. 흔한 문제

| 증상 | 원인 | 해결 |
|---|---|---|
| pgvector insert 시 dimension mismatch 에러 | `embedding_model_dims` 설정값과 실제 임베딩 모델 차원 불일치 | 설정값을 모델 차원에 맞게 수정 후 컬렉션 재생성 |
| Pipeline이 목록에 안 뜸 | Pipelines 서버 연결(Connections) 설정 누락 | Admin Settings → Connections에 Pipelines 엔드포인트/키 등록 확인 |
| 기억이 다음 대화에 반영 안 됨 | outlet에서 저장은 됐지만 `user_id`가 매 세션마다 달라짐 | Open WebUI 로그인 사용자 ID를 고정 키로 매핑해서 사용 |
| 응답 지연 증가 | inlet에서 검색 host가 원거리이거나 LLM 추출 호출이 동기로 블로킹 | `AsyncMemory` 사용, 저장(add)은 outlet에서 fire-and-forget으로 처리 |

---

## 참고 자료

- [Mem0 – pgvector 공식 문서](https://docs.mem0.ai/components/vectordbs/dbs/pgvector)
- [Mem0 – LLM 설정 문서](https://docs.mem0.ai/components/llms/config)
- [Mem0 GitHub](https://github.com/mem0ai/mem0)
- [mem0-owui (Open WebUI 연동 예제, Qdrant 기반)](https://github.com/cloudsbird/mem0-owui)
- [Open WebUI Pipelines](https://github.com/open-webui/pipelines)
- [Open WebUI Filter Function 문서](https://docs.openwebui.com/features/extensibility/plugin/functions/filter/)
