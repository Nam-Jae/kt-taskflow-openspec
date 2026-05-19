# TaskFlow MVP — OpenSpec Spec v1
> propose 기반 문서: `proposal.md`
> 이 문서는 Claude Code가 구현 시 따를 유일한 계약서. 여기 없는 것은 구현하지 않는다.

---

## 1. 프로젝트 구조 (File Tree)

```
taskflow/
├── backend/
│   ├── main.py              # FastAPI app, StaticFiles 마운트, CORS
│   ├── database.py          # SQLAlchemy engine + session (SQLite/Neon 전환)
│   ├── models.py            # ORM 4테이블 정의
│   ├── schemas.py           # Pydantic 요청/응답 스키마
│   ├── auth.py              # JWT 발급·검증, bcrypt 헬퍼
│   ├── dependencies.py      # get_current_user (JWT 미들웨어)
│   └── routers/
│       ├── auth.py          # POST /auth/signup, login, logout · GET /auth/me
│       ├── teams.py         # POST /teams, /teams/join · GET /teams/{id}, members · DELETE leave
│       ├── tasks.py         # GET/POST /teams/{id}/tasks · GET/PUT/PATCH/DELETE /tasks/{id}
│       └── messages.py      # GET/POST /teams/{id}/messages · DELETE /messages/{id}
├── frontend/
│   ├── index.html           # 진입점 → /login redirect
│   ├── login.html           # B-2 로그인
│   ├── signup.html          # B-1 회원가입
│   ├── teams.html           # C-1 팀선택 (미가입 진입)
│   ├── kanban.html          # D 칸반
│   ├── chat.html            # E 채팅
│   ├── css/
│   │   └── tailwind.min.css # CDN 빌드 (로컬 파일)
│   └── js/
│       ├── api.js           # fetch 래퍼, JWT 헤더 자동 첨부, 401 catch
│       ├── auth.js          # localStorage JWT 관리
│       ├── kanban.js        # 드래그앤드롭, 칸반 렌더링
│       ├── chat.js          # 폴링 setInterval, since= 증분
│       └── router.js        # 페이지 진입 시 team_id 분기 guard
├── requirements.txt
├── .env.example             # DATABASE_URL, SECRET_KEY
├── .gitignore               # taskflow.db, .env
└── vercel.json              # Vercel 배포 설정
```

---

## 2. 환경 설정

### 의존성 (`requirements.txt`)
```
fastapi
uvicorn[standard]
sqlalchemy
psycopg2-binary        # Neon(PostgreSQL) 연결용
python-dotenv
passlib[bcrypt]        # bcrypt 해시
python-jose[cryptography]  # JWT
pydantic[email]        # 이메일 검증
```

### 환경 변수 (`.env`)
```
DATABASE_URL=sqlite:///./taskflow.db   # 로컬
# DATABASE_URL=postgresql://...neon.tech  # 배포 시 Vercel 환경변수로 주입
SECRET_KEY=<랜덤 32바이트 hex>
ALGORITHM=HS256
ACCESS_TOKEN_EXPIRE_HOURS=24
```

### CORS 허용 도메인 (`main.py`)
```python
origins = [
    "http://localhost:8000",
    "http://localhost:5500",
    "https://taskflow.vercel.app",   # 배포 URL 확정 후 교체
]
```

---

## 3. DB 모델 (`models.py`)

```python
# users
id            Integer  PK  autoincrement
email         String   unique=True  not null
password_hash String   not null
team_id       Integer  FK(teams.id)  nullable  # 결정#1: 1인1팀
created_at    DateTime default=utcnow

# teams
id            Integer  PK  autoincrement
name          String   not null  # 1–30자 (서버 검증)
invite_code   String   unique=True  not null  # 자동 생성: ^[A-Z]{4}-[0-9]{4}$
owner_id      Integer  FK(users.id)  not null
created_at    DateTime default=utcnow

# tasks
id            Integer  PK  autoincrement
team_id       Integer  FK(teams.id)  not null
title         String   not null  # 1–100자 (서버 검증)
status        String   default='TODO'  # CHECK: TODO|DOING|DONE
creator_id    Integer  FK(users.id)  not null
assignee_id   Integer  FK(users.id)  nullable  # 결정#4: '내태스크' 기준
created_at    DateTime default=utcnow  # 정렬용

# messages
id            Integer  PK  autoincrement
team_id       Integer  FK(teams.id)  not null
user_id       Integer  FK(users.id)  not null
content       String   not null  # 1–1000자 (서버 검증)
created_at    DateTime default=utcnow
```

**인덱스 (SQLAlchemy `Index`)**
```python
Index('ix_tasks_team_created',    tasks.c.team_id,    tasks.c.created_at)
Index('ix_messages_team_created', messages.c.team_id, messages.c.created_at)
Index('ix_users_team',            users.c.team_id)
# teams.invite_code 는 unique=True 로 자동 인덱스
```

---

## 4. 인증 (`auth.py`, `dependencies.py`)

```python
# bcrypt
def hash_password(plain: str) -> str: ...
def verify_password(plain: str, hashed: str) -> bool: ...

# JWT
def create_access_token(user_id: int) -> str:
    # exp = utcnow + 24h
    # payload = { "sub": str(user_id), "exp": ... }

def get_current_user(token: str = Depends(oauth2_scheme), db = Depends(get_db)) -> User:
    # 401 TOKEN_EXPIRED: exp 초과
    # 401 TOKEN_EXPIRED: 토큰 없음/파싱 실패
    # users.team_id 포함해서 반환
```

**초대코드 생성**
```python
import random, string
def generate_invite_code() -> str:
    letters = ''.join(random.choices(string.ascii_uppercase, k=4))
    digits  = ''.join(random.choices(string.digits, k=4))
    return f"{letters}-{digits}"
```

---

## 5. API 엔드포인트 상세

### 5-1. Auth (`/auth`)

| 엔드포인트 | 요청 Body | 응답 | 에러 |
|-----------|----------|------|------|
| `POST /auth/signup` | `{ email, password }` | `201 { token, user: {id,email,team_id} }` | `409 EMAIL_TAKEN` · `400 VALIDATION_ERROR` |
| `POST /auth/login`  | `{ email, password }` | `200 { token, user: {id,email,team_id} }` | `401 INVALID_CREDENTIALS` |
| `GET /auth/me`      | — (JWT 필요)          | `200 { id, email, team_id, created_at }` | `401 TOKEN_EXPIRED` |
| `POST /auth/logout` | — (JWT 선택)          | `200 {}` | — |

**서버 검증**
- 이메일: `pydantic EmailStr`
- 비밀번호: 길이 8자 이상 (서버 측 체크)
- 로그인 실패: 이메일 존재 여부 관계없이 `INVALID_CREDENTIALS` 단일 메시지

---

### 5-2. Team (`/teams`)

| 엔드포인트 | 요청 | 응답 | 에러 |
|-----------|------|------|------|
| `POST /teams` | `{ name }` (JWT 필요) | `201 { id, name, invite_code, owner_id, created_at }` | `400 VALIDATION_ERROR` (name 길이) |
| `POST /teams/join` | `{ invite_code }` (JWT 필요) | `200 { team: {id,name,member_count}, redirect }` | `400 VALIDATION_ERROR` · `404 NOT_FOUND` · `409` 이미 팀 소속 |
| `GET /teams/{id}` | JWT 필요 + 멤버 검증 | `200 { id, name, invite_code, member_count }` | `403 FORBIDDEN` |
| `GET /teams/{id}/members` | JWT 필요 + 멤버 검증 | `200 [{ id, email, role, joined_at }]` | `403 FORBIDDEN` |
| `DELETE /teams/{id}/leave` | JWT 필요 + 멤버 검증 | `200 {}` | `403 FORBIDDEN` |

**멤버 검증 의존성**
```python
def require_team_member(team_id: int, current_user = Depends(get_current_user)):
    if current_user.team_id != team_id:
        raise HTTPException(403, detail={"code": "FORBIDDEN", "message": "이 팀의 멤버가 아닙니다"})
```

**팀 합류 시퀀스**
1. `invite_code` 형식 검증: `^[A-Z]{4}-[0-9]{4}$`
2. DB에서 `teams WHERE invite_code = ?` 조회 → 없으면 `404 NOT_FOUND`
3. `current_user.team_id IS NOT NULL` → `409`
4. `users.team_id = teams.id` UPDATE
5. 응답: `{ team: {...}, redirect: "/teams/{id}" }`

---

### 5-3. Task (`/teams/{id}/tasks`, `/tasks/{id}`)

| 엔드포인트 | 요청 | 응답 | 에러 |
|-----------|------|------|------|
| `POST /teams/{id}/tasks` | `{ title, assignee_id? }` | `201` task object | `403` · `400` |
| `GET /teams/{id}/tasks` | `?filter=me\|unassigned` (선택) | `200 [task]` (created_at DESC) | `403` |
| `GET /tasks/{id}` | JWT + 멤버 검증 | `200` task object | `403` · `404` |
| `PUT /tasks/{id}` | `{ title?, assignee_id? }` | `200` task object | `403` · `404` |
| `PATCH /tasks/{id}/status` | `{ status }` | `200 { id, status }` | `403` · `404` · `400` (status 값 오류) |
| `DELETE /tasks/{id}` | JWT + 권한 검증 | `200 {}` | `403 FORBIDDEN` · `404` |

**DELETE 권한 검증**
```python
def require_task_delete_permission(task, current_user):
    is_creator = task.creator_id == current_user.id
    is_owner   = (team.owner_id == current_user.id)
    if not (is_creator or is_owner):
        raise HTTPException(403, detail={"code": "FORBIDDEN", ...})
```

**필터 쿼리**
```python
# filter=me
WHERE team_id=? AND assignee_id = current_user.id ORDER BY created_at DESC

# filter=unassigned
WHERE team_id=? AND assignee_id IS NULL ORDER BY created_at DESC

# 기본 (전체)
WHERE team_id=? ORDER BY created_at DESC
```

**task object 스키마**
```json
{
  "id": 100,
  "team_id": 7,
  "title": "DB 마이그레이션 작성",
  "status": "TODO",
  "creator_id": 42,
  "creator_email": "user@ex.com",
  "assignee_id": 42,
  "assignee_email": "user@ex.com",
  "created_at": "2026-05-19T10:30:00Z"
}
```

---

### 5-4. Chat (`/teams/{id}/messages`, `/messages/{id}`)

| 엔드포인트 | 요청 | 응답 | 에러 |
|-----------|------|------|------|
| `POST /teams/{id}/messages` | `{ content }` | `201` message object | `403` · `400 TOO_LONG` |
| `GET /teams/{id}/messages` | `?since=ISO8601` (선택) | `200 [message]` (최근 50개, since 있으면 이후 전체) | `403` |
| `DELETE /messages/{id}` | JWT + 본인 검증 | `200 {}` | `403 NOT_OWNER` · `404` |

**폴링 쿼리**
```python
# since 없음 (최초): 최근 50개
SELECT ... WHERE team_id=? ORDER BY created_at DESC LIMIT 50

# since 있음 (증분): 해당 시각 이후 전체
SELECT ... WHERE team_id=? AND created_at > since ORDER BY created_at ASC
```

**message object 스키마**
```json
{
  "id": 23,
  "team_id": 7,
  "user_id": 42,
  "user_email": "user@ex.com",
  "content": "JWT 미들웨어만 끝내고 옮길게요",
  "created_at": "2026-05-13T14:27:00Z"
}
```

**DELETE 권한**
```python
if message.user_id != current_user.id:
    raise HTTPException(403, detail={"code": "NOT_OWNER", "message": "본인의 메시지만 삭제할 수 있습니다"})
```

---

## 6. 에러 응답 표준

모든 4xx/5xx는 아래 형태를 유지한다. `message`는 한국어로 UI에 그대로 표시.

```python
# FastAPI HTTPException detail에 dict 사용
raise HTTPException(status_code=409, detail={
    "code": "EMAIL_TAKEN",
    "message": "이미 가입된 이메일입니다"
})

# 응답 JSON
{
  "error": {
    "code": "EMAIL_TAKEN",
    "message": "이미 가입된 이메일입니다"
  }
}
```

> FastAPI 기본 `detail` 포맷을 `{ "error": {...} }` 로 래핑하는 **exception handler** 를 `main.py`에 등록한다.

**전체 코드 목록**

| code | HTTP | 발생 조건 |
|------|------|---------|
| `EMAIL_TAKEN` | 409 | 회원가입 중복 이메일 |
| `INVALID_CREDENTIALS` | 401 | 로그인 실패 |
| `TOKEN_EXPIRED` | 401 | JWT 만료·누락 |
| `FORBIDDEN` | 403 | 비멤버 접근 / 권한 없는 삭제 |
| `NOT_OWNER` | 403 | 타인 메시지 삭제 시도 |
| `NOT_FOUND` | 404 | 초대코드·리소스 없음 |
| `VALIDATION_ERROR` | 400 | 필드 형식·길이 오류 |
| `TOO_LONG` | 400 | 메시지 1000자 초과 |

---

## 7. 프론트엔드 구현 명세

### 7-1. `api.js` — fetch 래퍼

```javascript
const BASE = '';  // 로컬: FastAPI StaticFiles 일체형이므로 동일 origin

async function apiFetch(path, options = {}) {
  const token = localStorage.getItem('token');
  const res = await fetch(BASE + path, {
    ...options,
    headers: {
      'Content-Type': 'application/json',
      ...(token ? { Authorization: `Bearer ${token}` } : {}),
      ...options.headers,
    },
  });
  if (res.status === 401) {
    localStorage.removeItem('token');
    location.href = '/login.html';
    return;
  }
  if (!res.ok) {
    const body = await res.json();
    throw body.error;  // { code, message }
  }
  return res.json();
}
```

### 7-2. `router.js` — 페이지 진입 guard

```javascript
// 모든 HTML 페이지 <script>에서 호출
async function guardPage(requireTeam = false) {
  const token = localStorage.getItem('token');
  if (!token) { location.href = '/login.html'; return; }

  const user = await apiFetch('/auth/me');
  if (requireTeam && !user.team_id) {
    location.href = '/teams.html';  // 팀 미가입 → 강제 redirect
  }
  if (!requireTeam && user.team_id) {
    location.href = `/kanban.html?team=${user.team_id}`;  // 팀 있으면 칸반으로
  }
  return user;
}
```

### 7-3. `kanban.js` — 드래그앤드롭

```javascript
// HTML5 native drag API
card.setAttribute('draggable', true);
card.addEventListener('dragstart', e => e.dataTransfer.setData('taskId', task.id));

column.addEventListener('dragover', e => e.preventDefault());
column.addEventListener('drop', async e => {
  const taskId = e.dataTransfer.getData('taskId');
  const newStatus = column.dataset.status;  // 'TODO'|'DOING'|'DONE'
  await apiFetch(`/tasks/${taskId}/status`, {
    method: 'PATCH',
    body: JSON.stringify({ status: newStatus }),
  });
  renderKanban();  // 즉시 리렌더
});
```

### 7-4. `chat.js` — 5초 폴링

```javascript
let lastSince = null;
let pollInterval = null;

async function pollMessages(teamId) {
  const url = lastSince
    ? `/teams/${teamId}/messages?since=${encodeURIComponent(lastSince)}`
    : `/teams/${teamId}/messages`;

  try {
    const msgs = await apiFetch(url);
    if (msgs.length > 0) {
      renderMessages(msgs);
      lastSince = msgs[msgs.length - 1].created_at;
    }
    showConnected();
  } catch {
    showDisconnected();   // ⚠ 연결 끊김 배너
    // exponential backoff 는 clearInterval + setTimeout으로 구현
  }
}

function startPolling(teamId) {
  pollMessages(teamId);
  pollInterval = setInterval(() => pollMessages(teamId), 5000);
}

// 입력 포커스 시 2초로 단축
input.addEventListener('focus',  () => { clearInterval(pollInterval); pollInterval = setInterval(() => pollMessages(teamId), 2000); });
input.addEventListener('blur',   () => { clearInterval(pollInterval); pollInterval = setInterval(() => pollMessages(teamId), 5000); });
```

### 7-5. 메시지 1000자 카운터

```javascript
input.addEventListener('input', () => {
  const len = input.value.length;
  counter.textContent = `${len} / 1000`;
  counter.classList.toggle('text-red-500', len > 1000);
  sendBtn.disabled = len === 0 || len > 1000;
});
```

### 7-6. 반응형 브레이크포인트 (Tailwind)

| 구간 | 동작 |
|------|------|
| `< 768px` (모바일) | 칸반 1컬럼 + 스와이프, 채팅 풀스크린, 햄버거 메뉴 |
| `768–1024px` (태블릿) | 헤더 탭 통합 |
| `> 1024px` (데스크탑) | 3컬럼 칸반 + 사이드 패널 |

Tailwind 클래스 기준: `md:` = 768px, `lg:` = 1024px

---

## 8. Vercel 배포 설정

### `vercel.json`
```json
{
  "builds": [
    { "src": "backend/main.py", "use": "@vercel/python" },
    { "src": "frontend/**",     "use": "@vercel/static" }
  ],
  "routes": [
    { "src": "/api/(.*)", "dest": "backend/main.py" },
    { "src": "/(.*)",     "dest": "frontend/$1" }
  ]
}
```

### 환경 변수 (Vercel Dashboard)
```
DATABASE_URL  =  postgresql://...neon.tech/taskflow?sslmode=require
SECRET_KEY    =  <랜덤 32바이트>
```

### 로컬 실행
```bash
# 백엔드 (포트 8000, StaticFiles로 frontend/ 서빙)
uvicorn backend.main:app --reload

# frontend/는 별도 서버 불필요 (FastAPI StaticFiles 일체형)
# 브라우저: http://localhost:8000
```

---

## 9. 구현 순서 (Implementation Order)

```
Phase 1 — 기반 (막힘 없이 순서대로)
  1. 프로젝트 디렉토리 생성 + requirements.txt + .env
  2. database.py — SQLAlchemy engine + session
  3. models.py — 4테이블 + 인덱스 정의
  4. main.py — FastAPI 앱, CORS, exception handler, StaticFiles 마운트
  5. auth.py — bcrypt 헬퍼, JWT 발급·검증
  6. dependencies.py — get_current_user, require_team_member

Phase 2 — 백엔드 API (라우터 순서)
  7.  routers/auth.py — signup, login, logout, me
  8.  routers/teams.py — create, join, get, members, leave
  9.  routers/tasks.py — CRUD 6개 + filter + PATCH status
  10. routers/messages.py — polling GET, POST, DELETE

Phase 3 — 프론트엔드
  11. js/api.js — fetch 래퍼, 401 redirect
  12. js/router.js — 페이지 guard
  13. login.html + signup.html (B 화면)
  14. teams.html (C 화면 — 팀 선택)
  15. kanban.html + js/kanban.js (D 화면)
  16. chat.html + js/chat.js (E 화면)
  17. 모바일 반응형 (F 화면 — Tailwind 클래스 추가)

Phase 4 — 배포
  18. vercel.json 작성
  19. Vercel MCP로 배포 + DATABASE_URL 환경변수 주입 (Neon)
  20. 동작 확인 (9개 화면 수동 검증)
```

---

## 10. 수동 검증 체크리스트

> pytest/jest 없음. 아래 항목을 브라우저에서 직접 확인.

```
인증
[ ] 회원가입 → 201 + JWT 반환
[ ] 중복 이메일 → 409 에러 메시지 표시
[ ] 비밀번호 7자 → 클라이언트 + 서버 400
[ ] 로그인 성공 → team_id 분기 정상
[ ] 로그인 실패 → 401 단일 메시지 (이메일 노출 X)
[ ] JWT 만료 후 API 호출 → 401 → /login redirect

팀
[ ] 팀 생성 → 초대코드 표시 (XXXX-YYYY 형식)
[ ] 초대코드 합류 → users.team_id 업데이트 → 칸반 redirect
[ ] 잘못된 초대코드 → 404 에러 메시지
[ ] 이미 팀 소속 → 409 에러 메시지
[ ] 비멤버 /teams/{id} 직접 접근 → 403 화면

칸반
[ ] TODO/DOING/DONE 카드 표시
[ ] 드래그 → PATCH 호출 → 컬럼 이동
[ ] 인라인 입력 → Enter 저장, Esc 취소
[ ] 필터(@me, 미할당) 동작
[ ] creator만 삭제 가능 (타인 카드 삭제 버튼 없음)
[ ] owner는 타인 카드도 삭제 가능

채팅
[ ] 메시지 전송 → 5초 폴링으로 표시
[ ] 1000자 초과 → 카운터 적색 + 전송 비활성화
[ ] 본인 메시지 hover → 삭제 아이콘
[ ] 타인 메시지 → 삭제 아이콘 없음
[ ] 폴링 실패 → ⚠ 연결끊김 배너

배포
[ ] Vercel 배포 URL 접근 → 로그인 화면
[ ] Neon DB 연결 → 데이터 영속
[ ] 모바일(768px 미만) → 1컬럼 스와이프 동작
```
