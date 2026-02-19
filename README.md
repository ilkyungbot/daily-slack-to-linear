# Daily Slack-to-Linear

슬랙 멘션을 수집하여 Linear 이슈로 자동 생성하는 **Claude Code 스킬**.

퇴근 전 `/daily-slack-to-linear` 한 번이면, 오늘 하루 나를 멘션한 슬랙 메시지 중 액션이 필요한 건만 골라 Linear 이슈로 만들어줍니다.

## 동작 방식

```
슬랙 멘션 수집 → 액션 아이템 판별 → 프로젝트 매칭 → Linear 이슈 생성 → 결과 요약
```

1. 오늘 날짜 기준 나를 멘션한 슬랙 메시지를 전체 수집
2. 요청/부탁/결정 필요한 건만 필터링 (완료 보고, 단순 알림 등 제외)
3. 설정된 키워드와 프로젝트에 매칭
4. Linear 이슈 자동 생성 (제목, 설명, 우선순위, Due Date 포함)
5. 생성 결과를 테이블로 요약

## 사전 요구사항

- [Claude Code](https://claude.ai/code) 설치
- **Slack MCP 서버** 연결 ([Slack MCP](https://github.com/anthropics/claude-code/tree/main/packages/slack-mcp))
- **Linear MCP 서버** 연결 ([Linear MCP](https://github.com/anthropics/claude-code/tree/main/packages/linear-mcp))

## 설치

### 1. 스킬 파일 복사

`SKILL.md`를 Claude Code 스킬 디렉토리에 복사합니다:

```bash
mkdir -p ~/.claude/skills/daily-slack-to-linear
cp SKILL.md ~/.claude/skills/daily-slack-to-linear/SKILL.md
```

### 2. 최초 실행 (셋업)

Claude Code에서 아래 명령어를 입력하면 대화형 셋업이 시작됩니다:

```
/daily-slack-to-linear
```

셋업에서 설정하는 항목:
- **Slack 계정** - 이름 또는 이메일로 검색하여 확인
- **Linear 팀** - 소속 팀 선택 (복수 가능)
- **담당 키워드** - 팀 프로젝트를 분석하여 키워드 제안, 매칭되는 프로젝트 확인
- **Due Date 규칙** - 우선순위별 마감일 (기본값: Urgent=당일, High=+2, Medium=+4, Low=+5 영업일)

설정은 `~/.claude/daily-slack-to-linear-config.json`에 저장되며, 이후 실행 시 셋업 없이 바로 동작합니다.

## 사용법

Claude Code에서 아래 중 하나를 입력:

```
/daily-slack-to-linear
슬랙 정리
오늘 멘션
퇴근 전 정리
```

### 출력 예시

```
## 오늘의 슬랙 → Linear 정리

| # | 이슈 ID | 제목                              | 프로젝트       | 우선순위 | Due Date |
|---|---------|-----------------------------------|---------------|----------|----------|
| 1 | PLT-123 | [GWS] 고객 환불 처리               | GWS 자동화     | High     | 2/23     |
| 2 | PLT-124 | 갤럭시북 소구점 정리                 | 네고왕 프로젝트 | High     | 2/23     |
| 3 | PLT-125 | 채널톡 구독 문의 태그 추가           | CX 빌드업      | Medium   | 2/25     |

### 제외된 항목
- "전달하고 완료 했습니다" → 이미 완료 보고
- "승인 처리 해두었습니다" → 이미 완료 보고

총 3건 생성 완료.
```

## 설정 변경

실행 중 아래와 같이 요청하면 설정을 수정할 수 있습니다:

```
설정 변경
프로젝트 추가
설정 초기화
```

## 알려진 제한사항

- Slack MCP의 `response_format: "concise"` 옵션은 에러를 발생시키므로 `detailed`만 사용
- Slack MCP의 `to:me` 쿼리는 에러를 발생시키므로 `<@{user_id}>` 형태로 검색
- Due Date 계산 시 한국 공휴일은 기본 제외 (설정에서 변경 가능)

## 라이선스

MIT
