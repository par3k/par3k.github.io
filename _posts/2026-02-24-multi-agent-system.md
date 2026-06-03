---
layout: post
title: "Claude API로 프로젝트 관리 멀티 에이전트 시스템 만들기"
date: 2026-02-24
categories: [테크]
tags: [Claude, Anthropic, Multi-Agent, FastAPI, Python, Spring Boot]
---

> Spring Boot 프로젝트를 예시로, Claude API를 활용해 개발자·인프라·보안 등 5개 전문 에이전트를 단계적으로 구축하는 방법을 정리합니다.

---

## 목차

1. [멀티 에이전트란?](#1-멀티-에이전트란)
2. [1단계: 기본 에이전트 구조](#2-1단계-기본-에이전트-구조)
3. [2단계: 파일 읽기 툴 추가](#3-2단계-파일-읽기-툴-추가)
4. [3단계: 에이전트 간 협업](#4-3단계-에이전트-간-협업)
5. [4단계: FastAPI REST API 래핑](#5-4단계-fastapi-rest-api-래핑)
6. [전체 아키텍처 정리](#6-전체-아키텍처-정리)

---

## 1. 멀티 에이전트란?

하나의 Claude 인스턴스가 모든 것을 처리하는 대신, **역할이 다른 여러 에이전트**가 협력하는 구조입니다.

```
                    ┌─────────────────────┐
  사용자 요청  ────→  │   오케스트레이터      │
                    │  (작업 분류 & 위임)   │
                    └─────────────────────┘
                              │
          ┌───────────────────┼───────────────────┐
          ↓                   ↓                   ↓
   ┌─────────────┐    ┌─────────────┐    ┌─────────────┐
   │ 개발자 에이전트│    │인프라 에이전트│    │보안 에이전트 │
   └─────────────┘    └─────────────┘    └─────────────┘
```

**핵심 개념:** 각 에이전트 = Claude API 호출 + 전문화된 system prompt + 역할별 tools

### 5개 에이전트 역할

| 에이전트 | 담당 |
|---------|------|
| `developer` | Java/Spring Boot 코드 개발, 버그 수정 |
| `infra` | Terraform, AWS EC2/RDS/KMS 인프라 |
| `quality` | 코드 품질 리뷰, SOLID 원칙, 테스트 |
| `security` | KMS 암호화, JWT, OWASP 취약점 |
| `config` | Git, Jenkins CI/CD, 배포 파이프라인 |

---

## 2. 1단계: 기본 에이전트 구조

### 설치

```bash
pip install anthropic
export ANTHROPIC_API_KEY="sk-ant-..."
```

### 에이전트 정의 및 오케스트레이터

```python
import anthropic

client = anthropic.Anthropic()

AGENTS = {
    "developer": {
        "name": "소스코드 개발자",
        "emoji": "👨‍💻",
        "system": "당신은 Java/Spring Boot 전문 개발자입니다. ..."
    },
    "infra": {
        "name": "인프라 관리자",
        "emoji": "🏗️",
        "system": "당신은 AWS 인프라 전문가입니다. Terraform으로 관리합니다. ..."
    },
    # quality, security, config ...
}

def call_agent(agent_key: str, user_message: str) -> str:
    agent = AGENTS[agent_key]
    response = client.messages.create(
        model="claude-opus-4-6",
        max_tokens=2048,
        system=agent["system"],
        messages=[{"role": "user", "content": user_message}]
    )
    return response.content[0].text
```

### 오케스트레이터 (자동 라우팅)

```python
def orchestrate(user_request: str) -> str:
    # Claude에게 어떤 에이전트가 적합한지 판단 위임
    routing_response = client.messages.create(
        model="claude-opus-4-6",
        max_tokens=50,
        system="사용자 요청을 보고 담당 에이전트를 선택하세요: developer / infra / quality / security / config. 키 하나만 답하세요.",
        messages=[{"role": "user", "content": user_request}]
    )
    agent_key = routing_response.content[0].text.strip()
    return call_agent(agent_key, user_request)

# 사용 예시
result = orchestrate("KMS 키 로테이션 정책을 검토해줘")
# → [보안담당 관리자]에게 자동 위임
```

---

## 3. 2단계: 파일 읽기 툴 추가

단순 텍스트 답변을 넘어 **실제 프로젝트 파일을 읽어서 분석**하도록 툴을 추가합니다.

### 아젠틱 루프 (Agentic Loop)

```
사용자 요청
    ↓
Claude API 호출
    ↓
응답에 tool_use 블록이 있나?
    YES → 툴 실행 → 결과를 messages에 추가 → 다시 Claude 호출 (반복)
    NO  → end_turn → 최종 답변 반환
```

### 3개 파일 툴 구현

```python
import os, glob

def read_file(path: str) -> str:
    with open(os.path.join(PROJECT_ROOT, path), "r", encoding="utf-8") as f:
        return f.read()

def list_files(directory: str = ".", pattern: str = "*") -> str:
    matched = glob.glob(os.path.join(PROJECT_ROOT, directory, "**", pattern), recursive=True)
    return "\n".join(sorted(os.path.relpath(p, PROJECT_ROOT) for p in matched if os.path.isfile(p)))

def search_code(keyword: str, file_pattern: str = "*.java") -> str:
    results = []
    for fpath in glob.glob(os.path.join(PROJECT_ROOT, "**", file_pattern), recursive=True):
        with open(fpath) as f:
            for lineno, line in enumerate(f, 1):
                if keyword.lower() in line.lower():
                    results.append(f"{os.path.relpath(fpath)}:{lineno}  {line.rstrip()}")
    return "\n".join(results[:50])
```

### 아젠틱 루프 구현

```python
def run_agent(agent_key: str, user_message: str, on_event=None) -> str:
    agent = AGENTS[agent_key]
    messages = [{"role": "user", "content": user_message}]

    while True:
        response = client.messages.create(
            model="claude-opus-4-6",
            max_tokens=4096,
            system=agent["system"],
            tools=TOOLS,
            messages=messages,
        )
        messages.append({"role": "assistant", "content": response.content})

        # 툴 호출 없으면 최종 답변
        if response.stop_reason == "end_turn":
            return next(b.text for b in response.content if hasattr(b, "text"))

        # 툴 호출 처리 → 결과를 다음 메시지로
        tool_results = []
        for block in response.content:
            if block.type == "tool_use":
                result = execute_tool(block.name, block.input)
                tool_results.append({
                    "type": "tool_result",
                    "tool_use_id": block.id,
                    "content": result,
                })
        messages.append({"role": "user", "content": tool_results})
```

실행하면 에이전트가 실제로 파일을 열어 분석합니다.

```
🔒 [보안담당 관리자] 작업 시작...
  🔧 툴 호출: read_file({"path": "src/main/resources/application.yml"})
  🔧 툴 호출: search_code({"keyword": "password", "file_pattern": "*.yml"})
  🔧 툴 호출: search_code({"keyword": "secret", "file_pattern": "*.java"})
→ 실제 파일 내용 기반 보안 리포트 생성
```

---

## 4. 3단계: 에이전트 간 협업

3가지 협업 패턴을 구현합니다.

### Pattern 1: 순차 파이프라인

이전 에이전트의 출력이 다음 에이전트의 입력 컨텍스트로 전달됩니다.

```
initial_task
    │
    ▼
[개발자] 코드 구조 분석
    │ 결과 전달
    ▼
[품질관리자] 개발자 분석 참고 → 품질 이슈 추가
    │ 결과 전달
    ▼
[보안담당자] 앞 두 분석 참고 → 보안 취약점 최종 점검
```

```python
def run_pipeline(steps: list[tuple[str, str]], initial_task: str, on_event=None) -> dict:
    step_results = []
    previous_output = ""

    for i, (agent_key, instruction) in enumerate(steps, 1):
        message = f"""전체 목표: {initial_task}
이전 단계 결과: {previous_output}
당신의 임무: {instruction}"""

        result = run_agent(agent_key, message, on_event=on_event)
        step_results.append({"step": i, "agent": AGENTS[agent_key]["name"], "result": result})
        previous_output = result

    return {"step_results": step_results, "final": step_results[-1]["result"]}

# 사용 예시
result = run_pipeline(
    steps=[
        ("developer", "KmsEncryptionService 구조 분석"),
        ("quality",   "앞 분석 참고해 품질 리뷰"),
        ("security",  "보안 취약점 최종 점검"),
    ],
    initial_task="KMS 암호화 서비스 코드 리뷰",
)
```

### Pattern 2: 병렬 분석

여러 에이전트가 동시에 독립적으로 분석하고, 마스터가 결과를 종합합니다.

```
              ┌→ [인프라관리자]  terraform 분석  ─┐
task ─→ 분배  ├→ [보안담당자]   취약점 점검    ─┤→ 마스터 종합 → 리포트
              └→ [형상관리자]   CI/CD 분석     ─┘
```

```python
from concurrent.futures import ThreadPoolExecutor, as_completed

def run_parallel(agent_tasks: list[tuple[str, str]], on_event=None) -> dict:
    individual_results = {}

    with ThreadPoolExecutor(max_workers=len(agent_tasks)) as executor:
        futures = {
            executor.submit(run_agent, agent_key, task, on_event): agent_key
            for agent_key, task in agent_tasks
        }
        for future in as_completed(futures):
            agent_key, result = future.result()
            individual_results[agent_key] = result

    # 마스터 Claude가 종합 리포트 작성
    combined = "\n\n".join(f"[{AGENTS[k]['name']}]\n{v}" for k, v in individual_results.items())
    summary = client.messages.create(
        model="claude-opus-4-6",
        max_tokens=2048,
        system="여러 전문가의 분석을 통합해 핵심 인사이트와 우선순위 액션 아이템을 도출합니다.",
        messages=[{"role": "user", "content": combined}]
    )
    return {"individual": individual_results, "summary": summary.content[0].text}
```

### Pattern 3: 슈퍼바이저

**가장 강력한 패턴.** 슈퍼바이저 자체가 Claude이며, `ask_agent`가 툴입니다. 어떤 에이전트를 얼마나 호출할지 스스로 결정합니다.

```
               ask_agent("developer", ...)
[슈퍼바이저] ──ask_agent("security", ...) ──→ 동적 결정 → 최종 종합 리포트
               ask_agent("infra", ...)
```

```python
SUPERVISOR_TOOLS = [{
    "name": "ask_agent",
    "description": "전문 에이전트에게 분석을 위임합니다. (developer/infra/quality/security/config)",
    "input_schema": {
        "type": "object",
        "properties": {
            "agent": {"type": "string", "enum": list(AGENTS.keys())},
            "task":  {"type": "string"}
        },
        "required": ["agent", "task"]
    }
}]

def run_supervisor(task: str, on_event=None) -> str:
    messages = [{"role": "user", "content": task}]

    while True:
        response = client.messages.create(
            model="claude-opus-4-6",
            max_tokens=4096,
            system="당신은 기술 리더입니다. ask_agent로 전문가에게 위임하고, 완료되면 종합 리포트를 작성하세요.",
            tools=SUPERVISOR_TOOLS,
            messages=messages,
        )
        messages.append({"role": "assistant", "content": response.content})

        if response.stop_reason == "end_turn":
            return next(b.text for b in response.content if hasattr(b, "text"))

        tool_results = []
        for block in response.content:
            if block.type == "tool_use" and block.name == "ask_agent":
                result = run_agent(block.input["agent"], block.input["task"], on_event=on_event)
                tool_results.append({"type": "tool_result", "tool_use_id": block.id, "content": result})
        messages.append({"role": "user", "content": tool_results})
```

### 패턴 선택 가이드

| 상황 | 추천 패턴 |
|------|-----------|
| A 검토 후 B가 추가 리뷰 | 1. 순차 파이프라인 |
| 여러 팀이 각자 분석, 빠른 결과 필요 | 2. 병렬 분석 |
| 복잡한 과제, 어떤 에이전트가 필요한지 모름 | 3. 슈퍼바이저 |

---

## 5. 4단계: FastAPI REST API 래핑

에이전트 시스템을 HTTP API로 노출합니다. **SSE(Server-Sent Events)** 를 활용해 에이전트가 파일을 읽고 생각하는 과정을 실시간으로 스트리밍합니다.

### 설치 및 실행

```bash
pip install fastapi uvicorn
uvicorn api:app --reload --port 8000
```

### 엔드포인트 목록

| 메서드 | 경로 | 설명 |
|--------|------|------|
| `GET` | `/agents` | 에이전트 목록 |
| `POST` | `/agent/{key}` | 에이전트 직접 호출 (동기) |
| `POST` | `/agent/{key}/stream` | 에이전트 호출 (SSE 스트리밍) |
| `POST` | `/orchestrate` | 자동 라우팅 |
| `POST` | `/pipeline` | 순차 파이프라인 |
| `POST` | `/parallel` | 병렬 분석 |
| `POST` | `/supervisor/stream` | 슈퍼바이저 (SSE 스트리밍) |

### SSE 스트리밍 핵심 구현

에이전트는 동기(blocking) 함수이지만, `asyncio.Queue`와 `threading`을 조합해 실시간 스트리밍을 구현합니다.

```python
def make_sse_generator(target_fn, *args, **kwargs):
    queue = asyncio.Queue()
    loop = asyncio.get_event_loop()

    def on_event(event: dict):
        # 워커 스레드 → asyncio 큐 (thread-safe)
        asyncio.run_coroutine_threadsafe(queue.put(event), loop)

    def worker():
        target_fn(*args, on_event=on_event, **kwargs)
        asyncio.run_coroutine_threadsafe(queue.put({"type": "stream_end"}), loop)

    threading.Thread(target=worker, daemon=True).start()

    async def generate():
        while True:
            event = await queue.get()
            yield f"data: {json.dumps(event, ensure_ascii=False)}\n\n"
            if event["type"] == "stream_end":
                break

    return generate()

@app.post("/agent/{agent_key}/stream")
async def stream_agent(agent_key: str, body: TaskRequest):
    return StreamingResponse(
        make_sse_generator(run_agent, agent_key, body.task),
        media_type="text/event-stream",
    )
```

### SSE 이벤트 흐름

```bash
curl -N -X POST http://localhost:8000/agent/security/stream \
  -H "Content-Type: application/json" \
  -d '{"task": "KmsEncryptionService 보안 취약점 분석해줘"}'
```

```
data: {"type": "agent_start",  "agent": "security", "name": "보안담당 관리자"}
data: {"type": "tool_call",    "tool": "read_file",  "input": {"path": "src/.../KmsEncryptionService.java"}}
data: {"type": "tool_result",  "tool": "read_file",  "result": "..."}
data: {"type": "done",         "agent": "security",  "result": "분석 완료 리포트..."}
data: {"type": "stream_end"}
```

---

## 6. 전체 아키텍처 정리

```
클라이언트 (curl / 브라우저 / 다른 서비스)
          │
          ▼
    FastAPI api.py  (port 8000)
    ├── POST /agent/{key}           → run_agent()
    ├── POST /agent/{key}/stream    → SSE 스트리밍
    ├── POST /orchestrate           → orchestrate()
    ├── POST /pipeline              → run_pipeline()
    ├── POST /parallel              → run_parallel()
    └── POST /supervisor/stream     → SSE 스트리밍
          │
    on_event 콜백 (워커 스레드 → asyncio.Queue → SSE)
          │
    agents.py          파일 툴 (read_file / list_files / search_code)
    collaboration.py   파이프라인 / 병렬 / 슈퍼바이저
          │
    Claude API (claude-opus-4-6)
```

### 최종 파일 구조

```
agents/
├── agents.py          # 5개 에이전트 + 파일 툴 + 아젠틱 루프
├── collaboration.py   # 3가지 협업 패턴
├── api.py             # FastAPI REST API + SSE 스트리밍
└── requirements.txt   # anthropic, fastapi, uvicorn
```

---

## 마치며

이 시스템의 핵심은 단순합니다.

- **에이전트 = system prompt + tools** — 역할은 프롬프트로, 능력은 툴로 정의
- **아젠틱 루프** — `tool_use` → 실행 → 결과 전달 → 반복
- **협업 패턴** — 파이프라인(순차), 병렬, 슈퍼바이저 3가지로 대부분의 시나리오 커버
- **SSE 스트리밍** — `asyncio.Queue` + `threading`으로 동기 에이전트를 실시간 스트리밍으로 노출
