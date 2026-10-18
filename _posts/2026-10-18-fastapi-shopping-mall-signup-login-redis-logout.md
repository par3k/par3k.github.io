---
layout: post
title: "FastAPI 쇼핑몰 백엔드 만들기 — 회원가입/로그인과 Redis로 JWT 즉시 로그아웃"
date: 2026-10-18
categories: [테크]
tags: [FastAPI, Python, JWT, Redis, SQLAlchemy, PostgreSQL, bcrypt, 로깅, pytest]
---

이번 포스팅에서는 **FastAPI로 쇼핑몰 백엔드의 서버 뼈대와 회원 기능**(회원가입, 로그인, 로그아웃, 내 정보 조회)을 만든 과정을 정리합니다. 이전 글([JWT에서 Redis 서버사이드 세션으로 전환한 이유](/posts/jwt-to-redis-session-security/))에서 JWT의 약점으로 짚었던 **로그아웃 문제**를, 이번에는 세션으로 갈아타지 않고 JWT를 유지한 채 Redis로 풀어 봤습니다.

- 저장소: [par3k/shopping-ai-agent](https://github.com/par3k/shopping-ai-agent)
- 이 글의 코드: 태그 `step-1-login`

---

## 서버 구조

서버는 FastAPI 하나이고, 마이크로서비스로 쪼개지 않습니다. 영역은 폴더로 구분합니다. 지금 코드에 있는 파일만 그대로 옮기면 다음과 같습니다.

```
app/
├── main.py            # 앱 생성, 라우터 등록, 시작 시 테이블 생성
├── config.py          # 환경변수 → Settings
├── database.py        # SQLAlchemy 엔진/세션
├── dependencies.py    # get_db, get_redis, 토큰 검증, 현재 회원
├── redis_client.py
├── logging_config.py  # 로그 설정
├── middleware.py      # 접근 로그
├── models/member.py
├── schemas/member.py
├── services/auth.py   # 비밀번호 해시, JWT 발급/검증
└── routers/member.py
```

마이그레이션 도구는 쓰지 않고, 앱 시작 시 `Base.metadata.create_all`로 테이블을 만듭니다. 학습용 프로젝트에서 Alembic까지 끌어들이면 글의 초점이 흐려지기 때문입니다.

---

## 설정: 필수 환경변수가 없으면 시작하지 않는다

설정은 환경변수 세 개가 필수입니다. `DATABASE_URL`, `SECRET_KEY`, `REDIS_URL`입니다.

```python
REQUIRED_KEYS = ("DATABASE_URL", "SECRET_KEY", "REDIS_URL")

def load_settings(env: Mapping[str, str] | None = None) -> Settings:
    env = os.environ if env is None else env
    missing = [key for key in REQUIRED_KEYS if not env.get(key)]
    if missing:
        raise RuntimeError(f"필수 환경변수가 없습니다: {', '.join(missing)}")
    ...
```

값이 없는 채로 서버가 떠서 첫 요청에서 이상한 에러가 나는 것보다, **시작 시점에 어떤 키가 빠졌는지 알려 주며 실패**하는 편이 훨씬 디버깅하기 쉽습니다. `.env`는 `.gitignore`에 넣어 커밋하지 않고, 문서에는 키 이름만 적습니다.

---

## 회원 모델과 비밀번호 해시

`members` 테이블은 단순합니다. 이메일(unique, index), 해시된 비밀번호, 이름, 나이, 생성 시각입니다.

비밀번호는 `bcrypt`로 해시합니다. 평문은 어디에도 저장하지 않습니다.

```python
def hash_password(password: str) -> str:
    return bcrypt.hashpw(password.encode("utf-8"), bcrypt.gensalt()).decode("utf-8")

def verify_password(plain: str, hashed: str) -> bool:
    try:
        return bcrypt.checkpw(plain.encode("utf-8"), hashed.encode("utf-8"))
    except ValueError:
        return False
```

이메일은 소문자로 정규화해서 저장하고 조회합니다. `A@test.com`과 `a@test.com`이 서로 다른 회원이 되는 일을 막기 위해서입니다. 가입 시 중복 검사를 먼저 하고, 동시 요청으로 검사를 통과하더라도 unique 제약에서 `IntegrityError`가 나면 같은 400으로 응답합니다.

---

## 핵심: Redis로 JWT 즉시 로그아웃

이전 글에서 JWT의 가장 큰 약점으로 **로그아웃해도 토큰이 만료될 때까지 유효하다**는 점을 짚었습니다. 이번에는 세션으로 갈아타는 대신, JWT는 그대로 두고 **로그아웃된 토큰의 ID만 Redis에 기록**하는 방식을 택했습니다.

토큰을 발급할 때 고유 ID(`jti`)를 넣습니다.

```python
def create_access_token(data: dict, expires_delta: timedelta | None = None) -> str:
    expire = datetime.now(UTC) + (
        expires_delta or timedelta(minutes=settings.access_token_expire_minutes)
    )
    payload = {**data, "exp": expire, "jti": uuid.uuid4().hex}
    return jwt.encode(payload, settings.secret_key, algorithm=settings.algorithm)
```

로그아웃하면 `logout:{jti}` 키를 **토큰의 남은 유효 시간만큼만** 저장합니다.

```python
@router.post("/logout", status_code=status.HTTP_204_NO_CONTENT)
def logout(payload: dict = Depends(get_token_payload), ...):
    ttl = max(int(payload["exp"] - time.time()), 1)
    redis_client.set(logout_key(payload["jti"]), "1", ex=ttl)
```

TTL을 토큰 만료와 맞춘 이유는 두 가지입니다.

- 어차피 만료된 토큰은 JWT 검증 단계에서 걸러지므로, 그 이후에는 Redis에 남겨 둘 이유가 없습니다.
- 블랙리스트가 무한히 커지지 않습니다. 키가 알아서 사라집니다.

요청마다 토큰을 검증하는 의존성은 서명 검증 뒤에 이 키가 있는지 확인합니다.

```python
revoked = redis_client.exists(logout_key(jti))
if revoked:
    raise HTTPException(status.HTTP_401_UNAUTHORIZED, "Token revoked")
```

### Redis 장애 시에는 어떻게 할까

여기서 정책을 하나 정해야 했습니다. Redis가 죽었을 때 인증을 통과시킬 것인가, 거부할 것인가.

**통과시키면 로그아웃한 토큰이 되살아납니다.** 보안 쪽을 택해서, 인증 경로에서 Redis 오류가 나면 **503으로 거부**합니다. 가용성을 조금 포기하는 대신 "로그아웃한 토큰은 반드시 무효"라는 보장을 지키는 선택입니다.

---

## 로깅 추가

서버를 직접 띄워 보니 무슨 일이 일어나는지 볼 수 있어야 했습니다. 워크스페이스의 `log/app.log`에 롤링(10MB × 5개)으로 기록하도록 했습니다.

- **접근 로그(`api.access`)**: 요청 ID, method, path, member_id, client, 상태 코드, 소요 시간. 응답 헤더에도 `X-Request-ID`를 붙여서 클라이언트가 본 요청과 로그를 연결할 수 있습니다.
- **업무 로그(`api.member`)**: 가입, 로그인, 로그아웃의 성공과 실패.
- **SQL 로그(`sqlalchemy.engine`)**: 파일에만 기록하고, 한 건이 1500자를 넘으면 자릅니다.

```
INFO [api.access] rid=638374a0a061 POST /members/login member=- client=127.0.0.1 status=200 172.4ms
INFO [api.access] rid=50dabf1c6238 GET /members/me member=2 client=127.0.0.1 status=200 4.8ms
INFO [api.member] 로그아웃: member_id=2
```

로깅에서 가장 신경 쓴 부분은 **남기지 않을 것**을 정하는 일이었습니다. 지난 글의 보안 취약점 중 하나가 비밀번호 평문 로그 출력이었기 때문입니다.

- 요청 body는 기록하지 않습니다. 비밀번호가 들어 있을 수 있습니다.
- 토큰 값은 기록하지 않습니다. `member_id`는 토큰을 디코딩해 `sub`만 꺼내고, 검증에 실패하면 `-`로 남깁니다.
- SQL 바인딩 파라미터는 `hide_parameters`로 숨깁니다. 필요할 때만 `SQL_LOG_PARAMS=true`로 켭니다.

테스트로도 고정해 두었습니다. 로그인 후 `/members/me`를 호출하고, 로그 파일에 토큰과 비밀번호가 없는지 검사합니다.

---

## 트러블슈팅: `No module named 'psycopg'`

`requirements.txt`에 `psycopg2-binary`를 넣고 `.env`의 `DATABASE_URL`을 `postgresql://...`로 적었는데, 서버를 띄우자마자 이 오류가 났습니다.

```
File ".../sqlalchemy/dialects/postgresql/psycopg.py", line 497, in import_dbapi
    import psycopg
ModuleNotFoundError: No module named 'psycopg'
```

설치한 것은 `psycopg2`인데 SQLAlchemy가 `psycopg`(v3)를 찾고 있습니다. 원인은 **SQLAlchemy 2.1에서 `postgresql://`만 쓰면 기본 드라이버가 psycopg v3**라는 점입니다. 드라이버를 URL에 명시하면 해결됩니다.

```
DATABASE_URL=postgresql+psycopg2://...
```

해결 방법은 둘입니다. URL에 `+psycopg2`를 붙이거나, 아예 `psycopg[binary]`(v3)를 설치하는 것입니다. 이번에는 이미 설치된 psycopg2에 맞춰 URL을 고쳤습니다. 기본 드라이버는 라이브러리 버전에 따라 바뀔 수 있으니, **URL에 드라이버를 명시**해 두는 편이 안전합니다.

---

## 테스트: DB도 Redis도 없이 돌린다

테스트는 31개이고, 외부 서비스 없이 실행됩니다.

- DB는 SQLite 인메모리(`StaticPool`)로 바꿔 끼웁니다. FastAPI의 `dependency_overrides`로 `get_db`를 교체합니다.
- Redis는 `fakeredis`로 교체합니다. `get_redis`를 같은 방식으로 override합니다.
- 로그 디렉토리는 임시 폴더로 보냅니다. 테스트 때문에 `log/`가 오염되지 않습니다.

의존성 주입 덕분에 라우터 코드는 한 줄도 바꾸지 않고 저장소만 교체할 수 있었습니다. Redis 장애 시 503 동작도 fakeredis에 오류를 주입해서 검증합니다.

---

## 실서버 확인

테스트가 통과해도 실제 PostgreSQL과 Redis에 붙여 본 것은 별개입니다. `curl`로 흐름 전체를 순서대로 호출했습니다.

| 단계 | 결과 |
|------|------|
| `GET /health` | 200 |
| 회원가입 | 201 |
| 같은 이메일로 재가입 | 400 |
| 틀린 비밀번호로 로그인 | 401 |
| 로그인 | 토큰 발급 |
| `GET /members/me` (토큰 포함) | 200 |
| `GET /members/me` (토큰 없음) | 401 |
| 로그아웃 | 204 |
| **같은 토큰으로 `GET /members/me`** | **401 `Token revoked`** |

마지막 줄이 이번 글의 핵심입니다. 로그아웃한 토큰이 아직 만료되지 않았는데도 바로 거부됩니다. 이후 로그 파일을 열어 보니 요청마다 `api.access`와 `api.member` 로그가 남았고, 토큰과 비밀번호는 한 번도 기록되지 않았습니다. 확인에 쓴 테스트 계정은 끝난 뒤 DB에서 삭제했습니다.

---

## 마치며

이번 작업은 기능 자체는 평범한 회원가입/로그인이지만, 결정할 것이 생각보다 많았습니다.

- JWT의 약점(로그아웃)을 **세션 전환 없이 `jti` + TTL 블랙리스트**로 풀었습니다. 키가 토큰 만료와 함께 사라지므로 저장소가 커지지 않습니다.
- Redis 장애 시 인증 요청은 안전을 위해 거부(503)하도록 정했습니다.
- 로깅은 무엇을 남길지보다 **무엇을 남기지 않을지**가 중요했습니다.
- 드라이버 접두사 하나(`postgresql+psycopg2://`)로 한참 헤맬 수 있다는 것도 다시 확인했습니다.

`get_current_member` 의존성으로 인증을 한곳에 모아 두었기 때문에, 인증이 필요한 API는 이 의존성만 붙이면 됩니다.
