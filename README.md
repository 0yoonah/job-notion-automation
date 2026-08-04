# 채용공고 분석 자동화 파이프라인 (Notion × MCP × Claude)

채용공고를 넣으면 Claude가 기술 스택·업무·요구역량을 분석해 Notion에 정리하고, 보드로 지원 현황을 관리하는 도구.

---

## 문제 → 해결

| 반복 작업 | 자동화 |
|-----------|--------|
| 공고를 읽고 요구 기술·업무를 손으로 정리 | Claude가 구조화 추출 (회사·포지션·경력·필수/우대 기술·요구역량·주요업무) |
| 스프레드시트에 지원 현황 수기 관리 | Notion 보드 뷰(상태 기준 칸반)로 관리 |
| 공고가 쌓이면 정리를 미루게 됨 | 예약 routine이 인박스(미분석) 공고를 주기적으로 자동 정리 |

## 아키텍처

```
[입력]  채용공고 URL 또는 본문
  ├─ 온디맨드:  /analyze-job <URL|본문>        (즉시 처리)
  └─ 예약:      Notion 상태="인박스" 행 스캔     (cron으로 주기 처리)
        │
        ▼
[분석]  Claude(MCP 세션)가 구조화 추출  ── 별도 OpenAI API 키·서버 불필요
        회사 · 포지션 · 경력 · 필수기술[] · 우대기술[] · 요구역량[] · 채용전형[] · 주요업무
        │
        ▼
[저장]  Notion MCP (create-pages / update-page)
        속성 채우기 + 본문에 "AI 요약 / 원본 공고" 작성, 상태=관심
        │
        ▼
[관리]  Notion 보드 뷰(상태 기준) = 칸반 지원관리
        관심 → 지원예정 → 지원완료 → 서류통과 → 면접진행 → 최종합격 / 불합격
```

## Notion 데이터베이스 스키마

**채용공고 트래커** (개인 Notion 워크스페이스)

> 데모 스크린샷을 넣으려면 이 자리에 보드 뷰 이미지를 추가하세요. 예: `![지원 보드](./docs/board.png)`

| 속성 | 타입 | 설명 |
|------|------|------|
| 회사명 | Title | |
| 포지션 | Text | |
| 상태 | Select | 인박스·관심·지원예정·지원완료·서류통과·면접진행·최종합격·불합격 |
| 경력 | Select | 무관·신입·1년+ … 5년+ |
| 필수기술 / 우대기술 / 요구역량 | Multi-select | AI 추출, 미존재 옵션 자동 생성 |
| 채용전형 | Multi-select | AI가 추출한 전형 절차 (서류·코딩테스트·과제·1차·2차·임원면접·컬처핏) |
| 마감일 | Date | AI가 추출한 지원 마감일 (상시채용·미기재면 비움) |
| 주요업무 | Text | AI 요약 |
| URL / 지원일 / 메모 | URL · Date · Text | |
| 등록일 | Created time | 자동 |

**뷰**: `지원 보드`(board, 상태 그룹) · `인박스(미분석)`(table, 상태=인박스 필터) · `마감 임박`(table, 마감일 오름차순)

## 사용법

### 1) 온디맨드 단건 — `/analyze-job`
```
/analyze-job https://www.wanted.co.kr/wd/336520
/analyze-job (공고 본문 붙여넣기)
```
→ 공고 1건을 분석해 Notion에 카드 생성, 생성된 페이지 링크를 반환.

### 2) 온디맨드 일괄 — `/process-inbox`
1. 바쁠 땐 Notion에서 공고 **URL만** 새 행에 담고 상태를 `인박스`로 둔다.
2. 나중에 `/process-inbox` 한 번이면 인박스(미분석) 행을 모아 한꺼번에 분석·정리하고 상태를 `관심`으로 바꾼다.

### 3) 예약(선택) — 인박스 자동 처리
- 위 일괄 처리를 매일 정해진 시각에 **자동 실행**하는 claude.ai 클라우드 routine. **현재는 중단(비활성)** 상태.
- 설정·재활성화: [`routines/inbox-processor.md`](./routines/inbox-processor.md) 참고.

## 설계 노트

- **Notion 활용** — 칸반·필터·모바일 UI를 이미 제공하므로 프론트를 새로 만들지 않고 자동화에 집중.
- **Claude + MCP** — 별도 OpenAI 키·서버 없이 이미 쓰는 Claude로 분석(사용량은 소모), 온디맨드/예약이 [같은 추출 스펙](./prompts/analysis-spec.md)을 공유.

## 프로젝트 구조

```
job-notion-automation/
├── README.md
├── .gitignore
├── .claude/
│   └── commands/
│       ├── analyze-job.md      # 온디맨드 단건 /analyze-job
│       └── process-inbox.md    # 온디맨드 일괄 /process-inbox
├── prompts/
│   └── analysis-spec.md        # 추출·매핑 규칙 (Single Source of Truth)
└── routines/
    └── inbox-processor.md      # 예약 routine 프롬프트 + 설정법
```
> 명령을 쓰려면 위 파일들을 Claude Code 명령 경로
> (프로젝트 `.claude/commands/` 또는 `~/.claude/commands/`)에 두면 된다.

## 한계

- 현재 Notion 워크스페이스가 **무료 플랜**이라 인박스 행 조회(`query_*`)에 호출 한도가 있다.
  정리·수정은 제한이 없고, 조회는 fetch 폴백으로 우회한다.
- SPA(JS 렌더링) 채용 사이트는 페이지 본문 추출이 불완전해서, **JSON API를 우선 사용해 우회**한다
  (예: Wanted `/wd/{id}` → `/api/chaos/jobs/v4/{id}/details`). API도 없으면 본문 붙여넣기로 처리.
- 멀티셀렉트(기술/역량 등) 새 옵션은 자동 추가되지 않아, 값 설정 전에 `update-data-source`로 먼저 옵션을 추가한다.
