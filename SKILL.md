---
name: daily-slack-to-linear
description: Use when collecting daily Slack mentions for a user and creating Linear issues for actionable items. Trigger on "슬랙 정리", "daily slack", "오늘 멘션", "슬랙 리니어", "퇴근 전 정리", "/daily-slack-to-linear".
---

# Daily Slack-to-Linear

슬랙에서 오늘 하루 동안 나를 멘션한 메시지를 수집하고, 액션이 필요한 건을 Linear 이슈로 자동 생성한다.

## Step 0: MCP 서버 연결 확인 (필수)

스킬 실행 시 **가장 먼저** Slack과 Linear MCP 서버 연결 상태를 확인한다.

`ToolSearch`로 아래 두 도구가 사용 가능한지 확인:
- `mcp__slack__slack_search_public_and_private` (Slack 검색)
- `mcp__linear__create_issue` (Linear 이슈 생성)

**둘 다 사용 가능** → 다음 단계로 진행
**하나라도 없는 경우** → 즉시 중단하고 안내:

```
이 스킬을 사용하려면 Slack과 Linear MCP 서버가 모두 연결되어 있어야 합니다.

현재 상태:
- Slack MCP: ✅ 연결됨 / ❌ 연결 안 됨
- Linear MCP: ✅ 연결됨 / ❌ 연결 안 됨

`.claude.json`의 mcpServers에 누락된 서버를 추가해주세요.
```

---

## Step 1: 설정 확인

MCP 확인을 통과한 후 `~/.claude/daily-slack-to-linear-config.json` 파일 존재 여부를 확인한다.

### 설정 파일이 없는 경우 → 셋업 실행 (최초 1회)

사용자에게 아래 정보를 순서대로 질문하여 설정 파일을 생성한다:

1. **Slack 사용자 확인**: "슬랙에서 사용하는 이름 또는 이메일이 무엇인가요?"
   - 답변을 받으면 `mcp__slack__slack_search_users`로 검색하여 Slack User ID를 확인
   - 검색 결과를 보여주고 맞는지 확인

2. **Linear 팀 선택**: "Linear에서 소속된 팀을 선택해주세요 (복수 선택 가능)"
   - `mcp__linear__list_teams`로 팀 목록을 보여주고 선택하게 함 (1개 이상)

3. **담당 업무 키워드**:
   - 먼저 `mcp__linear__list_projects`로 선택된 팀의 **전체 프로젝트 목록을 조회**한다
   - 프로젝트 이름들을 분석하여 공통 테마/키워드를 추출한다
   - 추출된 키워드를 예시로 제안하며 질문: "본인이 주로 담당하는 업무 키워드를 입력해주세요. 현재 팀 프로젝트를 보면 {추출된 키워드 예시} 등이 있습니다."
   - 입력받은 키워드로 프로젝트를 필터링하여 매칭되는 프로젝트 목록을 보여줌
   - "이 프로젝트들이 맞나요? 추가/제거할 것이 있나요?" 확인
   - 사용자가 확인하면 최종 프로젝트 목록과 키워드를 저장
   - 이 키워드는 이후 슬랙 멘션 필터링에도 사용됨 (키워드에 해당하는 건만 이슈 생성)

4. **Due Date 규칙**: "우선순위별 마감일 규칙을 설정합니다. 기본값을 사용할까요?"
   - 기본값: Urgent=당일, High=+2영업일, Medium=+4영업일, Low=+1주(5영업일)
   - 영업일 = 토·일·한국 공휴일 제외
   - 사용자가 커스텀 값을 원하면 각각 입력받음

**설정 파일 형식** (`~/.claude/daily-slack-to-linear-config.json`):
```json
{
  "slack_user_id": "<Slack User ID>",
  "slack_user_name": "<사용자 이름>",
  "linear_teams": [
    { "id": "<Team ID>", "name": "<팀 이름>" }
  ],
  "linear_user_identifier": "me",
  "keywords": ["<키워드1>", "<키워드2>"],
  "projects": [
    { "id": "<project-uuid>", "name": "<프로젝트 이름>", "team": "<팀 이름>" }
  ],
  "due_date_rules": {
    "urgent": 0,
    "high": 2,
    "medium": 4,
    "low": 5
  },
  "exclude_weekends": true,
  "exclude_korean_holidays": true
}
```

### 설정 파일이 있는 경우 → 바로 실행

설정을 읽고 아래 워크플로우를 즉시 시작한다. 시작 전 "설정을 확인했습니다: [이름], [팀], 프로젝트 [N]개" 형태로 간단히 알린다.

---

## 워크플로우

### Step 2: 슬랙 멘션 수집

검색을 실행한다:

- `mcp__slack__slack_search_public_and_private`: query = `<@{slack_user_id}> on:{오늘 날짜}`, sort = `timestamp`, limit = `20`
- 결과가 20건이면 `cursor`로 다음 페이지도 조회하여 전체 수집

> **주의**: `to:me` 쿼리와 `response_format: "concise"`는 Slack MCP 버그로 에러 발생 → 사용하지 않는다.

결과에서 **본문(Text)만 추출**하고, Context before/after는 무시한다. 각 메시지에서 아래 필드만 사용:
- Channel, From, Time, Message_ts, Permalink, Text

### Step 3: 액션 아이템 판별

각 메시지를 분석하여 **액션이 필요한 건**만 필터링한다.

**포함**:
- 요청/부탁 ("~해주세요", "확인 부탁", "의견 주세요", "전달 부탁")
- 결정 필요 ("어떻게 할까요", "의견이 어떠신지")
- 작업 요청 ("업데이트 필요", "정리해주세요", "세팅 필요")

**제외**:
- 단순 알림/공지
- 이미 완료된 건 ("완료했습니다", "처리했습니다")
- 캘린더 초대, 미팅 알림
- 감사/인사 메시지
- 내가 보낸 메시지

### Step 4: 프로젝트 매칭

`keywords`가 설정되어 있으면, 액션 아이템 중 **키워드에 매칭되는 건만** 남긴다.

각 액션 아이템의 내용을 분석하여 `projects` 목록 중 가장 적합한 프로젝트에 매핑한다:
- 메시지 내용과 프로젝트 이름/성격을 비교하여 가장 적합한 프로젝트 배정
- 매칭되는 프로젝트가 없으면 사용자에게 질문하여 배정

### Step 5: Linear 이슈 생성

각 액션 아이템마다 `mcp__linear__create_issue`로 이슈를 생성한다:

- **team**: 프로젝트의 `team` 필드에 해당하는 `linear_teams`의 ID
- **assignee**: 설정의 `linear_user_identifier`
- **project**: Step 4에서 매핑된 프로젝트
- **title**: 핵심 액션을 간결하게 (예: "[GWS] 고객 환불 법인 계좌 안내")
- **priority**: 내용의 시급도를 판단하여 1(Urgent)~4(Low) 설정
- **description**: 아래 형식으로 작성

```markdown
## 배경
{요청자}({팀/역할})님이 #{채널명}에서 요청 ({날짜 시간})

## 내용
* {핵심 내용 정리}
* [슬랙 원문]({permalink})

## 액션
* {구체적으로 해야 할 일}
```

### Step 6: Due Date 산정

각 이슈의 due date를 설정한다:

1. 슬랙 메시지에 **마감일이 명시**된 경우 → 해당 날짜 사용 (주말/공휴일이면 직전 영업일로 조정)
2. **명시되지 않은 경우** → 우선순위별 규칙 적용:
   - 설정의 `due_date_rules`에 따라 오늘 기준 +N 영업일
   - 영업일 계산 시 토·일 제외
   - `exclude_korean_holidays`가 true이면 한국 공휴일도 제외

### Step 7: 결과 요약

모든 이슈 생성 완료 후 아래 형식으로 요약을 출력한다:

```
## 오늘의 슬랙 → Linear 정리

| # | 이슈 ID | 제목 | 프로젝트 | 우선순위 | Due Date |
|---|---------|------|----------|----------|----------|
| 1 | PLT-XXX | ... | ...      | High     | M/DD     |

### 제외된 항목
- {제외 사유와 함께 나열}

총 {N}건 생성 완료.
```

---

## 설정 변경

사용자가 "설정 변경", "설정 초기화", "프로젝트 추가" 등을 요청하면:
- `~/.claude/daily-slack-to-linear-config.json`을 읽어 현재 설정을 보여준다
- 변경할 항목만 수정하고 다시 저장한다
