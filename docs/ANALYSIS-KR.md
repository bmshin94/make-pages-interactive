# make-pages-interactive 전수조사 & 활용/수익화 분석 (한국어 정리)

> 작성: Claude Code 세션 (카리나 페르소나) 대화 정리
> 정리일: 2026-09-27
> 분석 대상: `make-pages-interactive` 레포지토리 전체 (코드 2,244줄)

## 🔗 관련 GitHub 주소

| 구분 | 주소 |
|---|---|
| **원본(Upstream)** | https://github.com/paraschopra/make-pages-interactive |
| **포크(이 레포)** | https://github.com/bmshin94/make-pages-interactive |
| 원본 통계 | ⭐ Stars 478 / 🍴 Forks 56 (2026-09 기준) |
| 라이선스 | MIT (Copyright 2026 Paras Chopra) |

---

## 1. 이게 뭐하는 프로젝트인가

**"정적 HTML 페이지를 Claude가 알아듣는 라이브 댓글판으로 바꿔주는 Claude Code 스킬"**

페이지에서 텍스트를 드래그하거나 요소를 클릭해 코멘트를 남기면, 그 코멘트가 로컬 인박스 파일에 쌓이고,
Claude가 그것을 읽어 HTML을 직접 수정한다. 페이지는 자동 리로드되며 무엇이 바뀌었는지 투어로 보여준다.

원작자는 긴 HTML 연구 리포트를 반복 수정하기 위해 만들었으나, 문서·디자인 목업·생성된 리포트·프로토타입 UI 등
모든 HTML 폴더에 적용 가능하다.

### 1.1 폴더 구조

```
make-pages-interactive/
├── SKILL.md          (8.1KB) 에이전트가 읽는 스킬 명세 = 핵심
├── README.md         (8.0KB) 사람이 읽는 문서
├── CLAUDE.md         (1.4KB) 카리나 페르소나 가이드 (이 포크에서 추가)
├── LICENSE                   MIT
├── screenshot.png            README용 스크린샷
├── lib/
│   ├── feedback.js   (1,127줄) 브라우저 주입용 댓글 UI 라이브러리
│   ├── feedback.css   (620줄) UI 스타일
│   └── server.py      (324줄) 파이썬 표준 라이브러리 전용 초경량 HTTP 서버
├── scripts/
│   ├── inject.py      (124줄) HTML 태그 멱등 삽입/제거
│   └── update.py       (49줄) git pull --ff-only
└── docs/
    └── ANALYSIS-KR.md        본 문서
```

**외부 의존성 0개.** `pip install` 불필요 (http.server, json, threading 등 표준 모듈만 사용).

### 1.2 동작 파이프라인

```
[1] "이 페이지들 인터랙티브하게 만들어줘"
[2] inject.py → 모든 *.html의 </head>에 CSS, </body>에 JS 태그 멱등 삽입
[3] server.py → localhost:5050 에서 해당 폴더 서빙
[4] 브라우저 접속 → feedback.js 로드 → 댓글 UI 표시
[5] 텍스트 선택 / 요소 클릭 / 일반 메모로 코멘트 작성
[6] submit batch → POST /feedback → feedback/inbox.jsonl 에 한 줄 append
[7] Claude가 Monitor로 감지 → HTML 직접 수정
[8] Claude가 feedback/history.json 에 변경 내역 append
[9] 페이지가 4초마다 history.json 폴링 → 자동 리로드(스크롤 보존) → 변경 투어
```

### 1.3 3가지 코멘트 방식

| 방식 | 조작 | 내부 처리 |
|---|---|---|
| 텍스트 선택 | 글자 드래그 | `selectionchange` → 말풍선 → 인용문 + 안정적 CSS selector |
| 요소 선택 | 🎯 버튼 후 클릭 (Shift로 다중) | `COMMENTABLE_TAGS`(P/IMG/TABLE/FIGURE/SECTION 등) 외곽선 |
| 페이지 단위 | `+ general` | 특정 요소에 묶이지 않는 자유 메모 |

각 코멘트 페이로드: `cf_id`(고유 ID), `selector`, `outerHTML`(600자 컷), 텍스트 스니펫(220자 컷), 타임스탬프.

### 1.4 설계 하이라이트

1. **배치 전송** — 코멘트를 모아 한 번에 POST. 에이전트가 맥락 전체를 보고 일관되게 수정.
2. **localStorage 복원** — 작성 중 코멘트가 새로고침에도 유지 (`cf-state-v1`).
3. **탭 제목 상태 표시** — 처리 중 `⏳ `, 완료 `🔔 ` 접두사.
4. **3중 자동 종료** — ① 부모 프로세스 사망 감지(`os.getppid() == 1`, 5초 폴링) ② 유휴 10분 타임아웃 ③ 수동 kill.
5. **Stale 경고** — 180초 무응답 시 "에이전트가 안 잡음" 배너. 단, history 변경 또는 `/status` 갱신이 오면 데드라인 연장.
6. **`/status` 엔드포인트** — 에이전트가 진행 상황 문자열을 기록(10분 후 자동 pruning).
7. **경로 탐색 방어** — `/lib/*` 서빙 시 `resolve()` 후 LIB_DIR 밖이면 403.
8. **UTF-8 강제** — 미설정 시 브라우저가 Latin-1로 폴백해 이모지 깨짐.

### 1.5 커밋 히스토리

| 커밋 | 작성자 | 내용 |
|---|---|---|
| `7435b9a` | Paras Chopra | 최초 릴리스 (2026-05-26) |
| `4c5ea20` | Paras Chopra | README 스크린샷 추가 |
| `951a2e0` | Sam Gobrail | `/status` 엔드포인트 (외부 기여 PR #9) |
| `053b056` | Paras Chopra | 투어/히스토리 현재 페이지 한정, 오탐 배너 수정 |
| `8cafd46` | bmshin94 | CLAUDE.md 카리나 페르소나 추가 |

### 1.6 활용처

- 데이터 분석 리포트 / 대시보드 리뷰
- 문서·매뉴얼·기술 스펙 검수
- 디자인 목업, 프로토타입 UI 피드백
- AI 생성 HTML 결과물 다듬기
- 코드 워크스루 문서에 질문 남기기

### 1.7 사용자 입장의 이점

1. 위치를 말로 설명하는 비용 제거 (드래그 → 지시)
2. selector + outerHTML 동반 전달로 컨텍스트 정확도 상승
3. 터미널 ↔ 브라우저 전환 불필요 (자동 리로드 + 투어)
4. 로컬 완결형 — 외부 전송 0 / API 토큰 0 / 의존성 0
5. 에이전트-사람 피드백 루프 설계 학습용 최고 예제

---

## 2. 더 쉬운 설명 (비유)

### 한 문장

> "내 웹페이지에 포스트잇을 붙이면, 그 포스트잇을 읽고 페이지를 직접 고쳐주는 비서를 붙이는 장치."

구글 독스의 댓글 기능을 **로컬 HTML 파일**에서 쓸 수 있게 하고, **반영하는 주체가 Claude**인 것.

### 식당 비유

| 식당 | 프로젝트 |
|---|---|
| 손님(주문) | 사용자 — 페이지 드래그해서 코멘트 |
| 주문서 | 코멘트 1개 (어디를, 무엇을) |
| 주문 전표 꽂이 | `feedback/inbox.jsonl` |
| 요리사 | Claude — HTML 수정 |
| 완성 벨 | `feedback/history.json` |
| 서빙 | 페이지 자동 리로드 + 투어 |
| 홀 직원 | `server.py` (전달만, 요리 안 함) |

**서버는 HTML을 수정하지 않는다.** 브라우저와 파일 사이의 다리일 뿐, 실제 수정은 100% Claude가 한다.

### 기억할 파일 3개

```
내폴더/
├── report.html            사용자 페이지 (태그 2줄만 추가)
└── feedback/
    ├── inbox.jsonl        사용자 → Claude (요청)
    ├── history.json       Claude → 사용자 (완료된 변경)
    └── status.json        Claude → 사용자 (진행 상황)
```

DB도 API도 없이 파일 3개로 통신한다.

### HTML에 실제 추가되는 것

```html
<head>
  <link rel="stylesheet" href="/lib/feedback.css">   <!-- 추가 -->
</head>
<body>
  <script src="/lib/feedback.js" defer></script>     <!-- 추가 -->
</body>
```

`--remove` 옵션으로 완전 원복 가능.

### 주의할 함정

두 태그가 `/lib/...` **절대 경로**를 가리키며, 이는 `server.py`만 해석 가능하다.
따라서 **HTML 파일을 더블클릭해서 직접 열면 댓글 위젯이 조용히 로드 실패**한다(페이지 자체는 정상 렌더).
반드시 `http://localhost:5050/report.html` 로 접속해야 한다.

### 체감 시나리오

```
0초    "매출 그래프" 드래그 → 💬 comment
5초    "막대 대신 선 그래프, 색은 파랑" 입력 → add
7초    다른 표 클릭 → "합계 행 추가" → add
9초    submit batch → 탭 제목 "⏳ 매출 리포트"
10초   Claude가 자동 감지 → 배너에 진행 상황 표시
40초   수정 완료 + history.json 기록 → 탭 제목 "🔔 매출 리포트"
44초   R 키 → 스크롤 보존 리로드 → 변경 2곳 하이라이트 + 투어
```

### 가장 영리한 아이디어 3개

1. **HTML을 입력 UI로 쓴다** — 새 프롬프트 창 대신 기존 페이지 자체가 입력 인터페이스.
2. **파일이 메시지 큐다** — Kafka/Redis/WebSocket 없이 `.jsonl` append + 4초 폴링.
3. **프로세스를 절대 남기지 않는다** — 부모 감시 + 유휴 타임아웃으로 좀비 서버 0개.

---

## 3. 핵심 질문 7개

### Q1. 설치 및 사용법

**설치 (1줄)**

```bash
git clone https://github.com/paraschopra/make-pages-interactive \
  ~/.claude/skills/make-pages-interactive

# 이 포크를 쓰려면
git clone https://github.com/bmshin94/make-pages-interactive \
  ~/.claude/skills/make-pages-interactive
```

Claude Code가 `~/.claude/skills/` 아래 `SKILL.md`를 가진 폴더를 자동 발견한다. 설정 파일 수정 불필요.
사전 준비물: Claude Code + Python 3.9+ (`list[Path]` 타입 힌트) + 브라우저.

**자연어 사용법**

| 목적 | 발화 |
|---|---|
| 시작 | "이 페이지들 인터랙티브하게 만들어줘" / "make these pages interactive" |
| 특정 폴더 | "`./reports` 폴더에 피드백 붙여줘" |
| 서버 종료 | "피드백 서버 꺼줘" |
| 원상복구 | "피드백 레이어 제거해줘" |
| 업데이트 | "make-pages-interactive 스킬 업데이트해줘" |

**수동 CLI**

```bash
# 태그 주입 (하위 폴더 포함: -r)
python ~/.claude/skills/make-pages-interactive/scripts/inject.py ./내폴더 -r

# 서버 실행 (--idle-timeout 0 = 자동종료 비활성화)
python ~/.claude/skills/make-pages-interactive/lib/server.py ./내폴더 --port 5050

# 서버 상태 확인 / 종료
curl -s http://localhost:5050/info
lsof -ti:5050 | xargs kill

# 원복
python ~/.claude/skills/make-pages-interactive/scripts/inject.py ./내폴더 --remove
```

**페이지 내 단축키:** `F` 패널 · `P` Pending · `H` History · `T` 투어 · `R` 리로드 · `⌘↵` 코멘트 추가 · `Esc` 취소

### Q2. 플러그인 / 스킬 / MCP 중 무엇인가

**정답: 스킬(Agent Skill)이다.** 근거는 `SKILL.md`의 YAML frontmatter(`name`, `description`, 트리거 문구).
Claude는 평소 `description`만 읽다가(토큰 절약), 트리거 문구를 만나면 본문 전체를 로드한다(Progressive Disclosure).

| | **Skill** (이것) | **MCP** | **Plugin** |
|---|---|---|---|
| 정체 | 마크다운 지침서 + 스크립트 | 실행되는 서버(프로토콜) | Skill/MCP/커맨드 묶음 배포 단위 |
| 핵심 파일 | `SKILL.md` | `.mcp.json` + 서버 프로세스 | `plugin.json` + marketplace |
| 통신 | 없음 (파일 읽기) | JSON-RPC (stdio/SSE/HTTP) | 없음 (구성만) |
| 설치 | `~/.claude/skills/` | 설정 파일 등록 | `/plugin` 명령 |
| 도구 목록 노출 | 안 됨 | `mcp__xxx__yyy`로 노출 | 내용에 따라 |
| 비유 | 레시피북 | 새 주방기구 | 밀키트 세트 |

**왜 헷갈리는가:** `server.py`라는 서버가 있어서 MCP처럼 보이지만, 이 서버는 **브라우저와 HTTP로** 대화하고
Claude는 **파일만 읽는다**. MCP 서버는 Claude와 JSON-RPC로 대화하며 도구 목록에 등록된다.

**판단 기준:** 절차/노하우를 가르치려면 Skill(가장 가볍고 우선 검토), 새 능력(외부 API/DB)을 주려면 MCP,
팀에 배포하려면 Plugin.

### Q3. API 토큰이 필요한가

**이 스킬 자체는 토큰이 전혀 필요 없다.** 코드 전수조사 결과:

| 확인 항목 | 결과 |
|---|---|
| 외부 HTTP 요청 코드 | 없음 |
| `Authorization` 헤더 | 없음 |
| 환경변수/시크릿/`.env` | 없음 (`os.getppid()`만 사용) |
| 외부 패키지 | 없음 (표준 라이브러리만) |
| `feedback.js` fetch 대상 | `/feedback`, `/status`, `feedback/history.json` — 전부 localhost |

데이터가 로컬을 벗어나지 않으므로 사내 기밀 문서에도 사용 가능.

**전제:** Claude Code 자체는 인증 필요(Pro/Max 구독 또는 Anthropic API 키) — 단 이건 스킬과 무관한 기본 요건.

**주의사항 2가지**
1. **인증이 전혀 없음** — `localhost:5050`에 접근 가능한 누구나 코멘트 POST 가능하며 `Access-Control-Allow-Origin: *`.
   공개 노출(0.0.0.0 바인딩/포트포워딩) 금지. 로컬 전용. 상용화 시 1순위 보완 지점.
2. **비용은 Claude 쪽에서 발생** — 코멘트 처리마다 HTML 읽기/수정으로 토큰 소비. 배치 전송이 유리.

### Q4. 왜 GitHub에서 유명한가 (⭐478 / 🍴56)

1. **타이밍** — 2026년 5월, Claude Code Skills 생태계 폭발기에 킬러 데모로 등장.
2. **명확한 페인포인트** — "위에서 세 번째, 아니 그 밑에..." 위치 설명 고통을 드래그 한 번으로 해결.
3. **설치 장벽 0** — `git clone` 한 줄, 의존성 0, 설정 0.
4. **시각적 임팩트** — 스크린샷 한 장으로 가치 전달, 소셜 확산에 최적.
5. **원작자 영향력** — Paras Chopra(Wingify/VWO 창업자)의 초기 확산력.
6. **읽고 싶은 코드** — 2,244줄로 완독 가능하고, 주석이 '무엇'이 아니라 '왜'를 설명(학습 자료 가치).
7. **발상의 전환** — "사람이 텍스트로 설명" → "사람이 결과물 위에 표시, AI가 읽음" 패러다임 전환.
8. **범용성** — 연구 리포트용으로 만들었지만 문서/디자인/대시보드/프로토타입 전부 커버.

### Q5. 로컬 에이전트 구축에 도움이 되는가

도움이 된다. 사용보다 **설계 패턴 이식**이 더 큰 가치.

| # | 패턴 | 내용 |
|---|---|---|
| 1 | **파일 = 메시지 큐** | `.jsonl` append-only. 작은 write는 원자적, 한 줄=한 메시지, `tail -f`로 디버깅 가능. 로컬 에이전트 90%는 이걸로 충분 |
| 2 | **양방향 채널 분리** | inbox(요청) / history(영구 기록, source of truth) / status(임시 상태, TTL) 분리 |
| 3 | **원자적 쓰기** | `.tmp` 작성 후 `os.replace()` — 읽는 쪽이 반쪽 JSON을 절대 안 봄 |
| 4 | **워치독** | 부모 PID 감시(+nohup 예외) + 유휴 타임아웃 → 좀비 프로세스/포트 충돌 영구 해결 |
| 5 | **TTL 자기 치유** | 쓰기마다 10분 초과 엔트리 pruning → 에이전트 크래시에도 "작업중"이 영구 고착되지 않음 |
| 6 | **멱등 변환** | 마커 검사로 중복 삽입 방지 + `--remove`로 완전 원복. 파일 수정 도구의 절대 원칙 |
| 7 | **생존 신호 + Stale 감지** | 타임아웃은 넉넉히(180초), 생존 신호 도착 시 데드라인 연장 → 헛경고로 신뢰 상실 방지 |
| 8 | **Monitor + 폴링 하이브리드** | 에이전트는 이벤트 기반 감시, 브라우저는 폴링(파일 감시 불가). 환경에 맞게 혼용 |

**SKILL.md 작성법 학습 포인트**
1. frontmatter에 트리거 문구 명시 (없으면 발동 안 함)
2. "When to invoke"로 상황별 플로우 분기 (Setup/Stop/Removal/Update)
3. 각 플로우를 번호 매긴 단계로 서술
4. 판단 분기를 명확히 ("artifact_dir이 같으면 재사용, 다르면 5051 시도")
5. JSON 스키마를 예시와 함께 정확히 제시
6. **"Gotchas" 섹션** — 에이전트가 실수할 지점을 문서로 미리 차단

**바로 적용 가능한 아이디어**

| 대상 | 방법 |
|---|---|
| 승인 게이트 | `approvals.jsonl`에 질의, 사람이 `approved.jsonl`에 응답 |
| 진행 대시보드 | `status.json` 패턴 + 로컬 웹 UI |
| 작업 히스토리 | `history.json` append-only → 감사 로그 확보 |
| 크론 에이전트 | 워치독 패턴으로 좀비 방지 |
| 사람 개입(HITL) | 코멘트 UI를 결과 검토 UI로 개조 |

### Q6. 수익화 가능성 (요약 — 상세는 4장)

| # | 아이디어 | 한 줄 | 난이도 |
|---|---|---|---|
| 1 | AI 리뷰 SaaS | Markup.io/Pastel 포지션 + "AI가 즉시 수정" 차별화 | 높음 |
| 2 | WordPress 플러그인 | PHP, 4천만 사이트 시장, 무료 유통 채널 | 중간 |
| 3 | 에이전시 화이트라벨 | 클라이언트 피드백 지옥 해결, 즉시 현금화 | 낮음 |
| 4 | 배포 프리뷰 붙박이 | Vercel/Netlify 프리뷰마다 리뷰 레이어 자동 장착 | 중간 |
| 5 | 버티컬 특화 | 교육/번역/법률/의료/IR/게임기획 | 중간 |

**핵심 차별점:** 기존 툴은 피드백을 *수집*해서 사람에게 넘긴다. 우리는 피드백을 *실행*한다.

### Q7. React나 PHP로 만들 수 있는가

가능하다. 아키텍처가 4덩어리로 분리되어 언어 교체가 쉽다.

```
[A] 브라우저 UI  → React / Vue / Vanilla
[B] HTTP 서버    → PHP / Node / Python
[C] 저장소       → 파일 / MySQL / Redis
[D] AI 실행부    → Claude Code / Claude API
```

**React 버전 설계**

```
@our/feedback-react
├── <FeedbackProvider>       Context: pending, history, status
├── <FeedbackLauncher />     플로팅 버튼 + 패널
├── <SelectionCatcher />     selectionchange 감지
├── <ElementPicker />        오버레이 + 요소 하이라이트
├── <CommentEditor />        Portal 렌더
├── <ChangeTour />           변경사항 워크스루
└── hooks/
    ├── useSelectionAnchor() Range → 안정적 selector
    ├── useFeedbackQueue()   배치 + localStorage
    └── useHistoryStream()   SSE 또는 폴링
```

장점: 컴포넌트 트리를 알기 때문에 불안정한 CSS selector 대신 React Fiber 경로/`data-testid`로 앵커링 가능,
DOM 재렌더링에 강함, **source map으로 "이 버튼" → `src/Button.tsx:42` 매핑 가능**, npm 배포 용이.
주의: 재렌더 중 Range 소실 → `useLayoutEffect`로 선택 복원, 상태 복잡도 → Zustand/Jotai 분리,
라이브러리 DOM은 **반드시 Portal + Shadow DOM** 내부에서만 렌더.

**PHP 버전 설계**

```
POST /api/feedback   inbox 저장 (DB 또는 JSONL)
POST /api/status     진행상황 기록
GET  /api/history    변경 이력
GET  /api/events     SSE 스트림 (폴링 대체)

src/
├── Controller/FeedbackController.php
├── Service/InboxService.php        큐 관리
├── Service/AgentBridge.php         Claude API 호출
├── Repository/CommentRepository.php
└── Middleware/AuthMiddleware.php   원본에 없던 인증 보강
```

장점: **WordPress 플러그인으로 직행 가능**(웹사이트 40% 점유, 강력한 유통 채널), 공유 호스팅 호환(국내 중소기업 접근성),
Laravel의 인증/큐/브로드캐스트로 원본의 약점(인증 부재) 해결, Laravel Reverb로 4초 폴링을 실시간으로 업그레이드.
주의: 요청-응답 모델이므로 긴 AI 작업은 **Queue + Worker 분리 필수**, 파일 큐는 `flock()`/DB 트랜잭션,
Claude API 스트리밍 처리 필요.

**추천 조합**

```
프론트  : React (TypeScript) + Shadow DOM + Portal
백엔드  : Laravel (PHP) 또는 Fastify (Node)
실시간  : SSE (WebSocket은 오버스펙)
저장소  : MySQL/Postgres (audit log 필수)
AI      : Claude API (claude-opus-5 / claude-sonnet-5)
배포형태: ① npm 패키지(셀프호스팅) ② SaaS 스크립트 태그 ③ WP 플러그인
```

**로드맵**
1. 1주차 — 원본을 그대로 사용하며 UX 감각 익히기
2. 2~3주차 — React MVP (텍스트 선택 + 코멘트 + 배치 전송)
3. 4주차 — Laravel 백엔드 + 인증
4. 5~6주차 — Claude API 연동으로 실제 수정 완성
5. 7주차~ — WordPress 플러그인 포장 및 유통

**가장 어려운 지점:** *앵커가 페이지 변경 후에도 살아남기.* 상용화에는 다중 전략 앵커링
(selector + 텍스트 퍼지 매칭 + 주변 컨텍스트 해시 + 위치 백업)이 필요하다.

---

## 4. 수익화 아이디어 상세

### 4.1 시장 분석 — 경쟁자는 이미 과금 중

| 서비스 | 하는 일 | 가격 | 한계 |
|---|---|---|---|
| Markup.io | 웹사이트 댓글 | $8~/mo | 수정은 사람이 |
| Pastel | 디자인/웹 리뷰 | $29~/mo | 피드백 수집만 |
| BugHerd | 버그 리포트 + 스크린샷 | $49~/mo | Jira 티켓 생성까지 |
| Usersnap | 피드백 위젯 | $49~/mo | 수집 + 분류 |
| Ruttl | 웹사이트 편집 + 코멘트 | $12~/mo | CSS만 편집, 코드 반영 X |

시장은 검증되었고 월 $8~$49 지불 의사가 존재한다. 그러나 전부 "수집"에서 멈추며 실제 수정은 개발자 수작업이다.

**유일한 무기**

```
기존: 피드백 → 티켓 → 개발자 대기열 → 3일 후 수정 → 재검토
우리: 피드백 → 30초 후 수정 완료 → 즉시 재검토
```

Feedback → Fix 시간을 3일에서 30초로 줄이는 것이 프리미엄 가격의 근거다.

### 4.2 아이디어 1: AI Review Loop SaaS (메인 추천)

**컨셉:** "고치라고 말하면, 고쳐져 있는 리뷰 툴"

```html
<script src="https://cdn.서비스.com/w.js" data-key="pk_xxx"></script>
```

→ 리뷰 레이어 장착 → 클라이언트 코멘트 → **AI가 GitHub PR 생성** → 개발자는 리뷰/머지만.

| 플랜 | 월 요금 | 내용 |
|---|---|---|
| Free | $0 | 1 프로젝트, 코멘트 무제한, AI 수정 월 20회 |
| Pro | $29 | 5 프로젝트, AI 수정 월 300회, GitHub PR 연동 |
| Team | $99 | 무제한 프로젝트, AI 수정 2,000회, 팀 권한, SSO |
| Enterprise | $499+ | 온프레미스, 감사 로그, 전용 지원, SLA |
| 추가 크레딧 | $0.10/회 | 초과분 |

가격 근거: 개발자 1시간 인건비 $30~50 대비 월 $29에 수정 300회는 설득이 쉽다.

12개월 목표: 1~2개월 MVP + 베타 20명 → 3~4개월 Product Hunt 런치, 유료 30명(MRR $900)
→ 6개월 100명(MRR $3,500) → 12개월 400명 + Team 10팀(MRR $12,000~15,000).

난이도: 높음 / 잠재력: 최상

### 4.3 아이디어 2: WordPress 플러그인 (PHP, 국내 최적)

웹사이트 40%가 WordPress(4천만+ 사이트). 플러그인 저장소가 **무료 유통 채널**이며,
국내 공유 호스팅(카페24, 가비아)에 그대로 설치된다.

**컨셉:** "AI Page Feedback for WordPress" — 클라이언트가 댓글 → AI가 본문/CSS/이미지 alt 수정 → 관리자 승인.

| | 가격 | 내용 |
|---|---|---|
| Free (저장소) | $0 | 코멘트 수집 + 관리자 대시보드 (AI 없음) |
| Pro | $49/년 | 1 사이트, AI 수정 월 100회 |
| Agency | $149/년 | 10 사이트, AI 무제한급, 화이트라벨 |
| Lifetime | $299 | 초기 자금 확보용 (WP 시장에서 잘 팔림) |

목표: 무료 설치 5,000 → 유료 전환 2%(100명) × $49 = 연 $4,900 + Agency 20곳 × $149 = 연 $2,980
→ 합계 약 연 $8,000. 상위 노출 시 트래픽이 자동 유입되어 유지보수 대비 수익이 좋다.

난이도: 중간 / 잠재력: 상

### 4.4 아이디어 3: 에이전시 화이트라벨 (가장 빠른 현금화)

디자인/웹 에이전시의 실제 고통: "3번째 섹션 폰트 키워주세요" → "아니 그 위요" → 카톡 스크린샷 →
수정 횟수 추적 불가. 시간이 곧 돈인 업계라 지불 의사가 높다.

**제품:** 클라이언트 전용 리뷰 링크(로그인 불필요 토큰 URL), 에이전시 로고 화이트라벨,
**수정 이력 자동 기록 → "추가 수정 요청" 과금 근거 자료**(킬러 기능), AI 1차 수정 후 디자이너 마감.

| 플랜 | 요금 |
|---|---|
| Studio | $79/mo (5 프로젝트 동시) |
| Agency | $199/mo (무제한 + 화이트라벨 + 커스텀 도메인) |
| 프로젝트 단건 | $29/프로젝트 |

B2B 직접 영업이 통하므로 국내 웹 에이전시 100곳 접촉 → 10곳 확보 시 즉시 MRR $2,000.

난이도: 낮음 (원본 코드 + 인증 + 테넌시) / 잠재력: 최상

### 4.5 아이디어 4: 배포 프리뷰 붙박이 (Vercel/Netlify)

```
PR 생성 → Vercel 프리뷰 URL → 리뷰 레이어 자동 주입
→ PM/디자이너가 프리뷰에서 코멘트 → AI가 같은 PR 브랜치에 커밋 추가
→ PR 코멘트로 "3개 피드백 반영 완료" 자동 알림
```

개발 워크플로우에 내장되어 팀 전체가 매일 사용 → **이탈률(churn)이 극히 낮음**.

| | 요금 |
|---|---|
| Free | 공개 레포, 월 50 수정 |
| Team | $20/seat/mo |
| Business | $40/seat/mo (SSO, 감사로그) |

Seat 기반이라 확장성이 좋다 (10명 $200/mo, 50명 $1,000/mo).

난이도: 중간 / 잠재력: 최상

### 4.6 아이디어 5: 버티컬 특화 (경쟁 희박)

| 버티컬 | 고객 | 킬러 기능 | 가격 |
|---|---|---|---|
| 교육/이러닝 | 강사, 인강업체 | 강의자료 질문 → AI가 보충 설명 삽입 | $39/mo |
| 번역/현지화 | 번역 에이전시 | 원문 대조 검수 → AI가 문맥 수정 | $89/mo |
| 법률/컴플라이언스 | 로펌, 법무팀 | 조항별 코멘트 + 감사 추적 | $299/mo |
| 의료 문서 | 병원, 제약사 | 규제 문서 검수 + 변경 이력 | $499/mo |
| IR/재무 리포트 | IR, 회계법인 | 숫자 검증 + 표 자동 수정 + 버전 관리 | $399/mo |
| 게임 기획서 | 게임사 기획팀 | 코멘트 → AI가 밸런스 표 재계산 | $149/mo |

규제 산업은 예산이 크고 경쟁이 적지만 **감사 로그 + 온프레미스**가 필수다.
이 레포의 `history.json` append-only 패턴이 감사 로그와 궁합이 좋다.

### 4.7 아이디어 6~9: 부수익

| # | 아이디어 | 모델 |
|---|---|---|
| 6 | 스킬 마켓플레이스 | 프리미엄 스킬 번들 $19~49 (일회성) |
| 7 | 유료 강의/책 | "Claude Code 스킬 만들기" $99 (인프런/유데미) |
| 8 | 컨설팅/구축 대행 | 기업 맞춤 에이전시 구축 건당 500~3,000만원 |
| 9 | 오픈코어 | OSS 무료 + 클라우드/SSO/감사로그 유료 |

### 4.8 추천 전략

```
STEP 1 (1~2개월) 에이전시 화이트라벨
  → 난이도 낮음 + 직접 영업 + 즉시 현금. 목표 10곳 × $199 = MRR $2,000
  → 진짜 고객의 목소리 확보가 핵심

STEP 2 (3~5개월) WordPress 플러그인
  → STEP 1 코드 재활용 + 무료 유통 채널. 목표 무료 5,000 설치 → 유료 100명

STEP 3 (6~12개월) 배포 프리뷰 SaaS (본편)
  → 앞 단계 수익으로 개발 자금 충당, Seat 기반 확장. 목표 MRR $10,000+
```

**성공 조건**
1. "수집"이 아니라 "수정"을 판다 — 유일한 무기이므로 메시지를 흐리지 않는다.
2. 앵커링 정확도가 제품력이다 — 엉뚱한 곳을 고치면 신뢰가 즉사. 개발 리소스 70% 투자.
3. 사람의 승인 단계를 반드시 넣는다 — "AI가 초안, 사람이 승인" 구조.
4. "수정 이력"을 자산으로 판다 — 감사 로그 / 과금 근거 / 성과 리포트로 전환.

**리스크와 대응**

| 리스크 | 대응 |
|---|---|
| 기존 강자(BugHerd 등)의 AI 기능 추가 | 버티컬로 좁혀 방어, 속도로 승부 |
| AI 수정 품질 불안정 | 사람 승인 게이트 + 원클릭 롤백 |
| Claude API 비용 | 수정 횟수 기반 과금으로 원가 전가, 캐싱 활용 |

**첫걸음:** 원본 스킬을 그대로 사용해 디자이너/에이전시 3명에게 시연한다.
"이거 나 필요해!" 반응이면 진행, "괜찮네?" 수준이면 버티컬을 더 좁힌다.
코드 작성 전 3명의 검증이 3개월 개발보다 가치 있다.

---

## 5. 한 줄 결론

의존성 0, 코드 2,244줄의 이 스킬은 **"사람이 결과물 위에 직접 표시하고 AI가 그것을 실행한다"**는
패러다임 전환을 가장 단순한 형태로 증명한 레퍼런스이며, 그 설계 패턴(파일 큐 / 원자적 쓰기 / 워치독 /
멱등 변환 / 생존 신호)은 로컬 에이전트를 만들 때 그대로 이식할 수 있다.
수익화는 "피드백 수집"이 아닌 **"피드백 실행"**을 파는 방향에서만 프리미엄이 성립한다.

---

### 참고 링크

- 원본 레포: https://github.com/paraschopra/make-pages-interactive
- 이 포크: https://github.com/bmshin94/make-pages-interactive
