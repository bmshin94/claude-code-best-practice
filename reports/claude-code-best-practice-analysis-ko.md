# claude-code-best-practice 레포 분석 (한국어)

<table width="100%">
<tr>
<td><a href="../">← Back to Claude Code Best Practice</a></td>
<td align="right"><img src="../!/claude-jumping.svg" alt="Claude" width="60" /></td>
</tr>
</table>

이 문서는 `claude-code-best-practice` 레포를 전수조사한 분석 결과와, 설치·정체성·토큰·에이전트 구축·수익화·React/PHP 구현 가능성·유튜브 강의 제작 가능성에 대한 Q&A를 한국어로 정리한 기록이다.

## 📎 관련 링크

| 구분 | 주소 |
|---|---|
| 원본 레포 (upstream) | https://github.com/shanraisshan/claude-code-best-practice |
| 작업 레포 (fork) | https://github.com/bmshin94/claude-code-best-practice |
| 작업 브랜치 | `claude/quirky-keller-z1pntj` |
| 라이선스 | MIT (Copyright 2025-2026 Shayan Rais) |
| Claude Code 공식 문서 | https://code.claude.com/docs |

---

## 1. 레포 정체성 — 한 줄 정의

> **Claude Code를 "챗봇"이 아니라 "엔지니어링 도구"로 쓰는 법을 가르치는 교과서 + 실제 작동하는 레퍼런스 구현체**

- 설치해서 돌리는 애플리케이션이나 라이브러리가 **아니다**
- 읽고, 이해하고, 필요한 부분만 **내 프로젝트로 베껴 쓰는** 레퍼런스 자료집
- README 자체가 *"Read this repo as a course, not as a workflow or skill"* 이라고 명시

### 신상 정보

| 항목 | 내용 |
|---|---|
| 원작자 | Shayan Rais (파키스탄) |
| 규모 | README 599줄 + 마크다운 문서 56개(약 5,600줄) + 실동작 설정 30여 개 |
| 실적 | GitHub Trending #1 (2026년 3월) |
| 스폰서 | Disrupt.com, ClaudeKit / 후원: Polar.sh |

### 핵심 철학 — "하네스(harness)"

모델(Opus/Sonnet/Haiku)은 Anthropic이 제공한다. 우리가 통제할 수 있는 것은 모델을 둘러싼 **껍데기 = 하네스**(컨텍스트 · 툴 · 권한 · 훅 · 메모리)뿐이다. 이 레포 전체가 "하네스 설계로 결과가 갈린다"는 하나의 주장을 증명하는 구조다.

- 근거 문서: `reports/why-harness-is-important.md`

---

## 2. 폴더 구조 전수조사

### 2.1 `.claude/` — 심장부 (실제로 작동하는 설정)

```
.claude/
├── settings.json          # 650줄 설정 교보재 (훅 30개 전부 등록)
├── agents/     (10개)     # 서브에이전트 정의
├── commands/   (10개)     # 슬래시 커맨드
├── skills/      (9개)     # SKILL.md 스킬
├── rules/       (2개)     # 경로별 지연 로딩 규칙
├── hooks/                 # hooks.py 483줄 + 사운드 100여 개
└── agent-memory/          # 에이전트가 스스로 누적한 기록
```

| 구성요소 | 실물 | 비고 |
|---|---|---|
| 서브에이전트 | `.claude/agents/*.md` | weather, time, 프레젠테이션 3종, 문서 자동갱신 워크플로 5종 |
| 커맨드 | `.claude/commands/*.md` | `/weather-orchestrator`, `/time-command`, `/workflows:*` |
| 스킬 | `.claude/skills/*/SKILL.md` | weather-fetcher, weather-svg-creator, 프레젠테이션 3종 등 |
| 룰 | `.claude/rules/*.md` | `paths:` 프론트매터 → **해당 경로 건드릴 때만 로딩** |
| 훅 | `.claude/hooks/scripts/hooks.py` | 훅 이벤트 30개 전부 처리, 크로스플랫폼(Win/Mac/Linux) |
| 메모리 | `.claude/agent-memory/weather-agent/` | 실행마다 두바이 기온을 **스스로 누적** (2026-03-06 → 2026-04-16) |

**`rules/` 지연 로딩이 핵심 테크닉**

```yaml
---
paths:
  - "presentation/**"   # 이 경로를 만질 때만 컨텍스트에 로딩됨
---
```

`paths:` 프론트매터가 있으면 해당 파일을 만질 때만 로딩된다. CLAUDE.md처럼 매 세션 전부 로딩되지 않아 **컨텍스트를 절약**할 수 있다.

### 2.2 `orchestration-workflow/` — 레포의 하이라이트 데모

**Command → Agent → Skill** 패턴의 완성 예제.

```
사용자 → /weather-orchestrator (Command, model: haiku)
           ↓ Step 1: AskUserQuestion — 섭씨? 화씨?
           ↓ Step 2: Agent 툴로 weather-agent 호출
                       weather-agent (model: sonnet, 툴: Read + Skill 만)
                         └→ Skill 툴로 weather-fetcher 호출
                              └→ Open-Meteo API에서 두바이 기온 조회
           ↓ Step 3: Skill 툴로 weather-svg-creator 호출
                       → weather.svg + output.md 생성
```

**설계의 교훈**: `weather-agent`의 툴 목록은 `Read`, `Skill` 두 개뿐이고 `WebFetch`가 **의도적으로 빠져 있다**. 에이전트가 "귀찮으니 직접 API를 부르자"고 우회하는 것을 구조적으로 불가능하게 만든 것이다.

- ❌ "직접 API 부르지 마" (프롬프트 — LLM이 무시할 수 있음)
- ✅ 애초에 API 부를 툴을 주지 않음 (**구조 — 어길 수 없음**)

추가 안전장치: `maxTurns: 5`(무한루프 차단), `permissionMode: acceptEdits`(확인창 제거)

에이전트 정의문의 결정적 문장:

> *"Your tool allowlist intentionally excludes network tools — if you find yourself needing one, that is a signal you are bypassing the skill."*

### 2.3 문서 폴더

| 폴더 | 개수 | 내용 |
|---|---|---|
| `best-practice/` | 8 | 기능별 모범사례. `claude-settings.md`가 **1,425줄** 전체 레퍼런스 |
| `implementation/` | 6 | 실제 구현 기록 (Agent Teams, Scheduled Tasks, Goal 등) |
| `reports/` | 12 | 심층 리포트 (Agent SDK vs CLI 340줄, LLM 성능저하 360줄, 모노레포 스킬 등) |
| `tips/` | 9 | Boris Cherny(Claude Code 창시자) + Thariq(Anthropic) 공식 팁 **83개** |
| `videos/` | 8 | Lenny's Podcast, Y Combinator, Karpathy 등 영상 요약 |
| `changelog/` | 12 | 공식 문서 변경 자동 추적 로그 |
| `tutorial/` | day0~1 | 초보자 입문 (OS별 설치 → 로그인 → 프롬프팅/에이전트/스킬 3단계) |
| `presentation/` | 3 덱 | HTML 슬라이드 (GDG 발표 등) |
| `agent-teams/` | — | 에이전트 **팀**(병렬 협업)으로 만든 time 워크플로 |
| `development-workflows/` | — | RPI(Research-Plan-Implement), 크로스모델(Claude+Codex) |
| `!/` | 100여 개 | 뱃지·마스코트 SVG. 폴더명이 `!`라서 정렬 시 항상 최상단 |

### 2.4 기타 설정

| 파일 | 내용 |
|---|---|
| `.mcp.json` | MCP 서버 3개 — `playwright`(브라우저), `context7`(최신 문서), `deepwiki`(레포 분석). 전부 `npx`, **API 키 불필요** |
| `.codex/` | 동일한 훅 시스템을 OpenAI Codex CLI용으로 미러링 (멀티 CLI 대응) |
| `.github/FUNDING.yml` | Polar.sh 후원 링크 |
| `CLAUDE.md` | 프로젝트 지침 (+ 이 포크에는 "카리나" 페르소나 추가됨) |

### 2.5 README 안의 큐레이션 테이블

1. **🧠 CONCEPTS** — Claude Code 기능 18개 × (공식문서 / 모범사례 / 구현예제) 3단 링크
2. **🔥 Hot** — 최신 베타 기능 30여 개 (Ultrareview, Agent Teams, Routines, Fast Mode, Advisor...)
3. **⚙️ DEVELOPMENT WORKFLOWS** — 유명 워크플로 레포 **13개 비교표** (Superpowers 288k★, Matt Pocock 265k★, Spec Kit 138k★ 등) + 단계 흐름도
4. **🔀 CROSS-MODEL** — Claude + Codex/Gemini 연동 3방식 (Plugin / MCP / Router)
5. **🧰 SKILL / 🤖 AGENT COLLECTIONS** — 스킬·에이전트 모음 레포 랭킹
6. **💡 TIPS AND TRICKS (83)** — 창시자 공식 팁
7. **☠️ STARTUPS / BUSINESSES** — Claude 기능이 대체한 스타트업 목록 (Code Review → CodeRabbit/Greptile, Agent SDK → LangChain/CrewAI 등)
8. **💵 Billion-Dollar Questions** — 아직 답 없는 열린 질문 13개
9. **📖 HOW TO USE** — "워크플로가 아니라 강의로 읽어라"

---

## 3. 쉬운 비유 정리

### 레시피북 vs 밀키트

| | |
|---|---|
| 밀키트 | 포장 뜯고 데우면 끝. `npm install` 하고 바로 쓰는 라이브러리 |
| **레시피북** | 읽고 내 주방에 맞게 따라 만드는 것 ← **이 레포** |

### Claude Code 4대 요소 = 주방 비유

| 요소 | 비유 | 실제 역할 |
|---|---|---|
| **커맨드** | 메뉴판 | `/명령` 하나로 정해진 코스가 순서대로 자동 진행 |
| **에이전트** | 전문 요리사 | 역할 고정. 칼만 주고 프라이팬을 안 주면 볶음 요리를 **못 한다** |
| **스킬** | 레시피 카드 | 미리 꽂아두기(`skills:` 필드) vs 그때그때 꺼내기(`Skill` 툴) |
| **훅** | 주방 타이머 | 이벤트마다 소리/알림. git commit 전용 사운드까지 존재 |

### 핵심 메시지 3줄

1. Claude를 챗봇처럼 쓰면 매번 다르게 대답해서 결과가 불안정하다.
2. 커맨드·에이전트·스킬·훅으로 "틀"을 만들어두면 매번 같은 품질이 나온다.
3. 이 레포는 그 틀을 만드는 법을 보여주는 교재다 — 베껴 쓰면 된다.

---

## 4. Q&A

### Q1. 설치 및 사용법?

"설치"라는 개념이 거의 없다. 패키지가 아니라 문서 레포다.

| 방법 | 명령 / 설명 |
|---|---|
| **A. 그냥 읽기** (추천) | README → `tutorial/day0` → `day1` → `best-practice/` 순서 |
| **B. 로컬에서 Claude와 함께 읽기** | `git clone … && cd … && claude` 후 "tips 폴더 읽고 내 CLAUDE.md 개선안 제안해줘" |
| **C. 데모 실행** | `claude` → `/weather-orchestrator` (훅 사운드엔 python3 필요) |
| **D. 내 프로젝트로 이식** (실전) | `.claude/settings.json`, `.claude/rules/`, `.claude/hooks/` 복사 후 CLAUDE.md 신규 작성 |

```bash
git clone https://github.com/bmshin94/claude-code-best-practice
cd claude-code-best-practice
claude
/weather-orchestrator
```

내 프로젝트로 이식:

```bash
mkdir -p .claude
cp ~/claude-code-best-practice/.claude/settings.json .claude/
cp -r ~/claude-code-best-practice/.claude/rules .claude/
cp -r ~/claude-code-best-practice/.claude/hooks .claude/   # 선택
```

> CLAUDE.md는 **200줄 이하**로 유지해야 Claude가 안정적으로 따른다.

**선행 조건**

| 필요 | 용도 |
|---|---|
| Node.js 18+ | Claude Code 본체 |
| Claude Code CLI | `npm i -g @anthropic-ai/claude-code` |
| Claude Pro/Max 구독 또는 API 키 | 로그인 |
| Python 3 (선택) | 훅 사운드 |

훅 끄기: `.claude/settings.local.json`에 `"disableAllHooks": true`

### Q2. 플러그인이야? 스킬이야? MCP야?

**셋 다 아니다. 그러나 셋 다 안에 들어 있다.**

| 질문 | 답 | 설명 |
|---|---|---|
| 플러그인? | ❌ | `plugin.json`도 marketplace 등록도 없다. `/plugin install` 불가 |
| 스킬? | ❌ (레포 자체는) | 안에 SKILL.md 9개가 **예제로** 들어있음 |
| MCP? | ❌ (레포 자체는) | `.mcp.json`에 MCP 서버 3개 **설정**이 들어있음 |

정체는 **교재 + 레퍼런스 구현(Reference Implementation)**. 플러그인화를 의도적으로 하지 않은 것 — 좋은 하네스는 프로젝트마다 다르기 때문이다.

| 카테고리 | 실물 | 개수 |
|---|---|---|
| 스킬 | `.claude/skills/*/SKILL.md` | 9 |
| 서브에이전트 | `.claude/agents/*.md` | 10 |
| 슬래시 커맨드 | `.claude/commands/*.md` | 10 |
| 훅 | `.claude/hooks/scripts/hooks.py` | 30 이벤트 |
| MCP 설정 | `.mcp.json` | 3 서버 |
| 룰 | `.claude/rules/*.md` | 2 |

### Q3. API 토큰을 사용해야 돼?

**이 레포 때문에 필요한 토큰은 0개다.**

| 대상 | 토큰 | 설명 |
|---|---|---|
| 레포 자체 | ❌ 불필요 | 마크다운 문서 |
| Claude Code 로그인 | ⚠️ 둘 중 하나 | Pro/Max 구독(정액, 권장) 또는 API 키(종량) |
| Open-Meteo (날씨 데모) | ❌ 불필요 | 무료 공개 API, 좌표 기반 |
| MCP playwright / context7 / deepwiki | ❌ 불필요 | 로컬 `npx` 실행, 공개 데이터 |
| GitHub 푸시 | ⚠️ | 포크에 커밋할 때만 |

**비용 최적화 테크닉**: 레포는 커맨드에 `model: haiku`, 판단이 필요한 에이전트에 `model: sonnet`을 지정한다. 단순 오케스트레이션은 싼 모델, 판단은 좋은 모델로 분리하는 패턴.

**보안**: API 키는 하드코딩하지 말고 환경변수(`ANTHROPIC_API_KEY`)로. 이 레포도 `settings.local.json`은 git-ignore 처리돼 있다.

### Q4. AI 에이전트 구축에 도움이 될까?

**도움이 크다. 단 "어떤 종류의 에이전트냐"에 따라 다르다.**

도움이 되는 영역:

| 목표 | 도움 | 근거 |
|---|---|---|
| Claude Code 기반 개발 자동화 | ★★★★★ | 레포의 주제 자체 |
| 멀티 에이전트 오케스트레이션 | ★★★★★ | Command→Agent→Skill + `agent-teams/` 병렬 협업 |
| 에이전트 안전/권한 설계 | ★★★★★ | "툴을 안 주면 못 한다" + allow/ask 분리 |
| 컨텍스트 엔지니어링 | ★★★★★ | `rules/` 지연로딩, progressive disclosure, 200줄 룰 |
| Agent SDK 제품화 | ★★★★ | `reports/claude-agent-sdk-vs-cli-system-prompts.md` |
| 에이전트 메모리/상태 | ★★★★ | `agent-memory/` 실제 누적 데이터 |
| 평가/디버깅 | ★★★ | `/doctor`, 훅 로깅, `reports/llm-day-to-day-degradation.md` |

한계:

| 못 하는 것 | 이유 |
|---|---|
| LangChain/CrewAI 코드 | 이 레포는 마크다운 설정 기반, 파이썬 프레임워크 코드 없음 |
| OpenAI/Gemini 전용 구축 | Claude 특화 (연동법은 `development-workflows/cross-model-workflow/`에 있음) |
| 벡터DB/RAG 파이프라인 | 다루지 않음 |
| 프로덕션 배포/스케일링 | 다루지 않음 |

**바로 쓸 수 있는 설계 원칙 6개**

1. **최소 권한** — 필요한 툴만. 우회 경로 제거
2. **역할 분리** — 오케스트레이터(싼 모델) / 전문가(좋은 모델) / 실행기(스킬)
3. **점진적 공개** — SKILL.md는 요약, 상세는 `reference.md`로 분리해 필요 시 로딩
4. **Fail-closed 가드레일** — 기대값이 안 오면 다음 단계 금지, 추측 진행 금지
5. **턴 제한** — `maxTurns`로 무한루프·비용 폭탄 차단
6. **관측성** — 훅으로 모든 이벤트에 신호. 블랙박스 금지

### Q5. 수익화 아이디어가 있어?

→ 아래 **5장 수익화 아이디어 상세** 참조.

### Q6. React나 PHP로 만들 수 있어?

"이 레포를 React/PHP로 다시 만들기"는 의미가 없다(마크다운 문서 모음). 그러나 **이 내용을 담은 웹서비스**는 충분히 만들 수 있다.

**React (Next.js) 방향**

| # | 아이템 | 스택 |
|---|---|---|
| 1 | 한국어 Claude Code 레퍼런스 포털 | Next.js App Router + MDX + Tailwind + Pagefind/Algolia |
| 2 | **하네스 설정 생성기 (Harness Builder)** ⭐ | 체크박스 선택 → settings.json/CLAUDE.md/agents ZIP 다운로드 |
| 3 | CLAUDE.md 린터/스코어러 | Monaco Editor + 규칙 엔진 |
| 4 | 워크플로 비교 대시보드 | GitHub API로 스타 수 자동 갱신 |
| 5 | 인터랙티브 학습 플랫폼 | tutorial 기반 + 퀴즈/진행도/수료증 |

**PHP 방향**

| # | 아이템 | 스택 |
|---|---|---|
| 1 | WordPress 플러그인/테마 (한국어 가이드 + 멤버십) | WP + WooCommerce |
| 2 | Harness Builder 백엔드 (API + 과금) | Laravel + Cashier + Sanctum |
| 3 | 사내 하네스 중앙 관리 대시보드 (B2B 온프레미스) | Laravel |
| 4 | 간단 정적 생성기 | Parsedown |

**추천 조합**: 프론트 Next.js + Tailwind/shadcn → 백엔드 Laravel 또는 Next.js API Routes → DB Supabase → 결제 Stripe/토스페이먼츠 → 배포 Vercel + Railway

선택 기준: **콘텐츠/SEO 중심이면 Next.js + MDX**(마크다운 그대로 사용 가능), **결제·회원·관리자 중심이면 Laravel**이 빠르다. Agent SDK는 TypeScript/Python 공식 지원이라 AI 기능까지 넣으려면 Next.js(TS)가 유리하다.

### Q7. 유튜브 강의 영상으로 제작 가능할까?

**가능하다.** 라이선스가 MIT라 상업적 이용/2차 창작이 허용되며, 출처 표시만 하면 된다.

> 설명란 예: `Based on github.com/shanraisshan/claude-code-best-practice (MIT License)`
> 주의: 이미지·뱃지 애셋은 원작자 제작물이므로 직접 제작하거나 출처를 표기한다.

**왜 지금인가**: 한국어 Claude Code **심화** 콘텐츠가 거의 없고, 원본 레포는 GitHub Trending #1 이력으로 공신력이 있으며, 기업의 AI 코딩 도구 도입 수요가 커지는 중이다. 게다가 레포가 이미 커리큘럼 구조(tutorial → best-practice → reports)로 짜여 있다.

**커리큘럼 12편 설계안**

| # | 제목 | 길이 | 소스 |
|---|---|---|---|
| 1 | 설치부터 로그인까지 (Win/Mac/Linux) | 12분 | `tutorial/day0/` |
| 2 | 챗봇처럼 쓰면 손해 — 프롬프팅의 한계 | 10분 | `tutorial/day1/` Level 1 |
| 3 | CLAUDE.md 제대로 쓰기 (200줄의 법칙) | 15분 | `best-practice/claude-memory.md` |
| 4 | settings.json 완전정복 — 확인창 지옥 탈출 | 20분 | `best-practice/claude-settings.md` |
| 5 | 서브에이전트 만들기 | 18분 | `claude-subagents.md` + `.claude/agents/` |
| 6 | 슬래시 커맨드로 워크플로 자동화 | 15분 | `claude-commands.md` |
| 7 | 스킬(SKILL.md)과 점진적 공개 | 18분 | `claude-skills.md` |
| 8 | 훅으로 소리 알림 만들기 | 15분 | `.claude/hooks/` |
| 9 | **실전: Command→Agent→Skill 오케스트레이션** ⭐ | 25분 | `orchestration-workflow/` |
| 10 | MCP 서버 연결 (Playwright/Context7) | 15분 | `.mcp.json` |
| 11 | 유명 워크플로 13개 비교 | 20분 | README 워크플로 표 |
| 12 | Boris Cherny 공식 팁 83개 총정리 | 25분 | `tips/` |

보너스: "Claude가 대체한 스타트업들" (README STARTUPS 섹션) — 조회수 후킹용

**제작 팁**

| 항목 | 추천 |
|---|---|
| 포맷 | 터미널 녹화(asciinema/OBS) + 화면 확대, 폰트 18pt↑ |
| 오프닝 훅 | "Claude Code를 챗봇으로 쓰고 있다면 90%를 못 쓰고 있는 겁니다" |
| 차별점 | 실제로 **돌려 보여주기**. 9편은 결과 SVG가 화면에 생성됨 |
| 길이 전략 | 롱폼 12편 + Shorts 30~50개(팁 1개씩) 파생 |
| 수익 경로 | 애드센스 → 강의 전환 → **기업 교육 인바운드** → 멤버십/스폰서 |

**승부처는 9편**이다. 1~8편은 설정 설명이지만 9편은 커맨드가 에이전트를 부르고, 에이전트가 스킬을 부르고, 결과물이 눈앞에 생성되는 과정을 보여준다. 추상 개념이 시각적 결과물로 바뀌는 지점에서 유료 전환이 일어난다. 따라서 9편을 무료 공개/홍보용으로 쓰고 나머지를 유료로 묶는 전략이 유효하다.

---

## 5. 수익화 아이디어 상세

### 5.0 전제 조건 (왜 기회인가)

| 요소 | 상태 | 의미 |
|---|---|---|
| 라이선스 | MIT | 번역·강의·상품화·판매 전부 합법 |
| 한국어 심화 콘텐츠 | 거의 없음 | 블루오션 |
| 원본 공신력 | GitHub Trending #1 | 마케팅 자산 |
| 기업 수요 | AI 코딩 도구 도입 러시 | B2B 예산 존재 |
| 콘텐츠 준비도 | 이미 커리큘럼 구조 | 제작 기간 단축 |

### 5.1 TIER 1 — 지금 바로 (저비용·고확률)

**① 한국어 레퍼런스 허브 (`claude-code-best-practice-kr`)**

| 항목 | 내용 |
|---|---|
| 초기 비용 | 0원 (GitHub + Vercel 무료) |
| 기간 | 2~4주 |
| 난이도 | ★★ |
| 수익 모델 | Polar/토스 후원, 스폰서 헤더, 제휴 링크, 리드 수집 |
| 예상 | 직접 수익은 작지만 **모든 수익의 유입 관문** |

체크리스트: 포크 → README 전면 번역 → 원작자 크레딧 + MIT 고지 → 한국 전용 섹션(국내 사례/스택/커뮤니티) → Next.js + MDX docs 사이트화(SEO) → 원작자에게 KR 버전 알리고 역링크 요청

**② 유튜브 채널 + 온라인 강의** (가장 추천)

| 항목 | 내용 |
|---|---|
| 초기 비용 | 10~50만원 (마이크/녹화) |
| 기간 | 첫 영상 1주, 12편 2~3개월 |
| 난이도 | ★★★ |
| 수익 모델 | 애드센스 + 인프런/클래스101/유데미 + 멤버십 |

현실적 시뮬레이션 (6~12개월 안착 기준)

| 채널 | 월 수익 |
|---|---|
| 유튜브 애드센스 (구독 5천~2만) | 20~100만원 |
| 인프런 강의 (8만원 × 월 50명, 수수료 차감) | 약 280만원 |
| 클래스101/유데미 추가 | 50~150만원 |
| **합계** | **약 350~500만원** |

차별화 3개: (1) 실제로 돌아가는 걸 보여준다 (2) 비용 최적화(haiku/sonnet 분리)를 강조 — 기업 수요와 직결 (3) 두바이 날씨 대신 **네이버 API / 공공데이터포털 / 토스페이먼츠** 등 한국 실무 예제로 교체

**③ 기업 교육 / 컨설팅** (단가 최강)

| 항목 | 내용 |
|---|---|
| 초기 비용 | 0원 (자료 이미 존재) |
| 난이도 | ★★★ (영업이 관문) |
| 단가 | 하루 150~500만원 |

| 패키지 | 내용 | 기간 | 가격대 |
|---|---|---|---|
| A. 입문 워크샵 | 설치~CLAUDE.md 작성 + 실습 | 1일(4h) | 150~300만원 |
| B. 하네스 구축 | 팀 전용 settings/agents/commands/hooks 설계·납품 | 2~4주 | 800~2,500만원 |
| C. 리테이너 | 월 유지보수 + 신기능 반영 + Q&A | 월 | 200~500만원 |

단가 근거: 개발자 10명 × 생산성 20% 향상 = 연 수억 가치. 기업 입장에서 1천만원은 저렴한 투자다.

영업 루트: 유튜브/블로그 인바운드 → LinkedIn 사례 연재 → **사내 선적용 후 성과 데이터를 레퍼런스화** → 개발자 밋업 발표(`presentation/`에 덱 존재) → 테크 기업 Dev Rel/플랫폼팀 직접 컨택

왜 컨설팅이 가장 수익성 높은가: 판매하는 것이 "Claude Code 사용법"이 아니라 **"조직의 개발 프로세스를 하네스로 코드화하는 능력"**이고, 이는 회사마다 달라 복제가 불가능해 가격 경쟁에 휘말리지 않는다. 강의는 복제되므로 가격이 하락한다. 따라서 **강의로 신뢰 → 컨설팅으로 수익**이 정석이다.

### 5.2 TIER 2 — 중기 (3~6개월)

**④ 스킬/플러그인 상품 판매** (원본 스폰서 ClaudeKit이 이미 이 모델)

| 상품 | 구성 | 가격 |
|---|---|---|
| 한국 스타트업 스타터 팩 | React/Next.js + Supabase 기준 CLAUDE.md + 에이전트 5종 + 커맨드 10종 | ₩49,000 |
| PHP/Laravel 레거시 리팩토링 팩 | 레거시 분석 에이전트 + 점진적 마이그레이션 워크플로 | ₩69,000 |
| 코드리뷰 자동화 팩 | 한국어 리뷰 에이전트 + 컨벤션 룰 + PR 템플릿 | ₩89,000 |
| 훅 사운드 프리미엄 팩 | 한국어 TTS 알림 | ₩29,000 |

채널: Gumroad / Lemon Squeezy / Polar / 자체몰(Laravel + 토스페이먼츠) · 예상 월 30~200만원

**⑤ Harness Builder SaaS** (확장성 최대)

```
입력: 프로젝트 타입(Next.js/Laravel/Spring/RN) · 팀 규모 · 보안 수준 · 필요 기능
출력: settings.json / CLAUDE.md / agents/*.md / commands/*.md / rules/*.md / hooks/
      → ZIP 다운로드 또는 GitHub PR 자동 생성
```

| 항목 | 내용 |
|---|---|
| 스택 | Next.js + Laravel/Supabase + Stripe/토스 |
| 기간 | MVP 1개월, 정식 2~3개월 |
| 난이도 | ★★★★ |
| 과금 | Free / Pro $19월(팀 프리셋·버전관리) / Team $99월(SSO·감사로그) |
| 예상 | 유료 100명 ≈ 월 $1,900 (약 260만원) MRR |

락인 기능: CLAUDE.md 린터(200줄 초과 경고, 모순 규칙 탐지) · 공식 문서 변경 자동 추적 알림(원본 `changelog/` 구조 활용) · 팀 하네스 버전관리/롤백

**⑥ 유료 뉴스레터 / 멤버십**

무료: 주간 업데이트 요약 / 유료 ₩9,900월: 신기능 심층 분석 + 월 1회 라이브 Q&A + 프리미엄 템플릿 + 디스코드
구독 500명 ≈ 월 495만원. 콘텐츠 파이프라인은 원본 `changelog/` 추적 구조를 자동화해 확보 가능. 관문은 **꾸준한 발행**.

### 5.3 TIER 3 — 장기 / 고난도

| # | 아이디어 | 규모 | 비고 |
|---|---|---|---|
| ⑦ | AI 에이전트 수주 개발 (SI) | 프로젝트당 2,000만~1억원 | Agent SDK(TS/Python) + React 대시보드 + Laravel 백엔드 |
| ⑧ | **사내 적용 → 간접 수익** | 연봉/승진/이직 몸값 | **리스크 0, 즉시 실행 가능. 모든 아이디어의 레퍼런스 데이터원** |
| ⑨ | 책 / 전자책 출판 | 전자책 ₩19,900 × 500부 ≈ 1,000만원 | 종이책은 인지도 → 강의/컨설팅 단가 인상 |
| ⑩ | 오픈소스 → 투자/채용 | — | KR 레포 스타 1,000+ → Anthropic Community Ambassador 지원 등 |

### 5.4 실행 로드맵

| 기간 | 할 일 |
|---|---|
| 0~1개월 | 사내 적용(⑧) — 리스크 0, 성과 데이터 확보 + KR 레포 번역 착수(①) |
| 1~3개월 | 유튜브 첫 5편(②) + KR 레포 공개 → SEO 시동 |
| 3~6개월 | 인프런 강의 출시(②) + 스킬 팩 판매(④) + 기업 교육 인바운드 대응(③) |
| 6~12개월 | Harness Builder MVP(⑤) + 컨설팅 리테이너(③) + 뉴스레터 론칭(⑥) |

### 5.5 우선순위 Top 3

| 순위 | 아이디어 | 이유 |
|---|---|---|
| 1 | ⑧ 사내 적용 → ③ 기업 컨설팅 | 리스크 0으로 시작, 레퍼런스가 쌓이면 단가 최강 |
| 2 | ② 유튜브 + 강의 | 모든 수익의 유입 채널. 한국어 심화 콘텐츠 공백 선점 |
| 3 | ⑤ Harness Builder SaaS | React/PHP 둘 다 활용 가능, Q6과 직결 |

### 5.6 핵심 원칙

수익화 순서는 **실적(⑧) → 콘텐츠(①②) → 신뢰 → 고단가(③⑤)** 다. 많은 사람이 반대로(유료 강의부터) 시작해 실패한다. 이 레포가 가르치는 것은 기술 자체가 아니라 **"팀 프로세스를 하네스로 코드화하는 능력"**이고, 그것은 실제로 해본 사람만 판매할 수 있다.

따라서 1순위는 자기 프로젝트에 `.claude/` 하네스를 제대로 구축해 **"커밋 전 체크 자동화, 리뷰 시간 40% 감소"** 같은 **숫자**를 만드는 것이다. 그 숫자가 유튜브 썸네일이 되고, 강의 소개문이 되고, 컨설팅 제안서가 된다.

---

## 6. 요약 — 이 레포가 주는 가치

| 가치 | 구체적 내용 |
|---|---|
| ⏰ 시간 | 검증된 settings.json / 에이전트 구조를 복사 → 삽질 수 주 절약 |
| 📈 실력 | "프롬프트 엔지니어" → "에이전틱 엔지니어"로 포지션 상승 |
| 🗺️ 정보 | 워크플로 13개 + 스킬/에이전트 컬렉션 랭킹 = 업계 지형도 |
| 🔒 안전 | allow/ask 분리, 최소 권한, maxTurns, fail-closed 가드레일 |
| 💰 기회 | MIT 라이선스 + 한국어 시장 공백 = 번역/강의/컨설팅/SaaS 선점 |

### 적용 체크리스트

```
[ ] CLAUDE.md 200줄 이하로 정리
[ ] .claude/rules/*.md + paths: 로 조건부 로딩 도입
[ ] settings.json allow/ask 분리 설정
[ ] 반복 작업 1개를 슬래시 커맨드로 추출
[ ] 에이전트에 최소 툴만 부여 (우회 경로 제거)
[ ] maxTurns 설정으로 무한루프/비용 차단
[ ] 오케스트레이션(커맨드→에이전트→스킬) 1개 직접 구현
[ ] 컨텍스트 50%에서 수동 /compact 습관화
```
