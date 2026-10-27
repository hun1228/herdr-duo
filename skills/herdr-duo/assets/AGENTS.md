# AGENTS.md

이 프로젝트는 두 AI 에이전트가 [herdr](https://herdr.dev) 터미널 워크스페이스에서 나란히 작업한다.

## 세션 구성

| 위치 | 세션 | 실행 명령 | herdr `agent` 값 |
|---|---|---|---|
| 왼쪽 창 | **Claude** (Claude Code) | `claude` | `claude` |
| 오른쪽 창 | **agy** (Antigravity CLI, Gemini) | `agy` | `agy` |

- 두 세션 모두 이 파일이 있는 폴더를 작업 폴더로 쓴다.
- 상대 세션을 부를 때는 "Claude" / "agy" 로 부른다.

## herdr로 메시지 보내기

### 1. 상대 창의 pane ID 찾기

pane ID(예: `w1:p4`)는 창을 닫았다 열면 바뀌므로 **하드코딩하지 말고 매번 조회**한다.

```sh
# Claude(왼쪽) → agy(오른쪽) 찾기
herdr pane neighbor --current --direction right

# agy(오른쪽) → Claude(왼쪽) 찾기
herdr pane neighbor --current --direction left
```

결과 JSON의 `result.neighbor.neighbor_pane_id` 가 상대 pane ID다.
전체 목록과 각 창의 에이전트 종류는 `herdr pane list` 로 확인할 수 있다 (`agent` 필드가 `claude` / `agy`).

### 2. 메시지 보내기

```sh
# 텍스트 입력 + Enter (프롬프트로 바로 전송)
herdr pane run <PANE_ID> "<메시지>"

# Enter 없이 텍스트만 입력
herdr pane send-text <PANE_ID> "<텍스트>"

# 특수 키 (esc, enter, ctrl+c 등)
herdr pane send-keys <PANE_ID> esc
```

- 메시지 앞에는 보낸 쪽을 밝힌다: `[from Claude] ...` / `[from agy] ...`
- 상대 화면에 안내 배너("esc to dismiss")가 떠 있으면 먼저 `esc` 를 보낸다.
- 상대가 작업 중(`agent_status: working`)이면 입력이 끼어들 수 있으니, `herdr pane list` 로 `idle` 인지 확인한 뒤 보낸다.

### 3. 답장 확인

`pane run` 은 상대 창에 타이핑할 뿐이라 **답장이 자동으로 돌아오지 않는다**. 상대 화면을 읽어서 확인한다.

```sh
# 최근 출력 읽기
herdr pane read <PANE_ID> --lines 40

# 특정 문자열이 나올 때까지 대기 (ms)
herdr pane wait-output <PANE_ID> --match "<문자열>" --timeout 60000
```

답장이 필요한 요청을 보낼 때는 "끝나면 herdr로 내 창에 `[from agy] 완료: ...` 형태로 보내줘" 처럼 회신 방법을 같이 적는다.

## 참고

- agy 실행 파일: `%LOCALAPPDATA%\agy\bin\agy.exe` (Windows) (PATH가 갱신되지 않은 터미널에서는 전체 경로로 실행)
- agy가 herdr 명령마다 승인을 묻지 않도록 `~/.gemini/antigravity-cli/settings.json` 의 `permissions.allow` 에 정규식 규칙이 들어 있다. Windows에서는 따옴표·공백·한글이 섞인 명령에 prefix 규칙이 적용되지 않으므로 `regex:` 규칙이 필요하다 (설정 변경 후 agy 재시작 필요):
  `"command(regex:herdr pane (run|send-text|send-keys|read|wait-output|neighbor|list) .*)"`
- 메시지로 작업을 시킬 때 폴링용 `sleep` 등 허용되지 않은 명령을 쓰지 말라고 같이 적으면 승인 프롬프트에 막히지 않는다.
- agy를 다시 띄우려면: `herdr pane send-keys <PANE_ID> esc` → `ctrl+c` 를 셸 프롬프트(`PS ...>`)가 보일 때까지 반복 (작업 중엔 ctrl+c가 작업만 취소함) → `herdr pane run <PANE_ID> '& "$env:LOCALAPPDATA\agy\bin\agy.exe" -c'` (`-c` = 직전 대화 이어가기. 전체 자동 승인 모드면 `--dangerously-skip-permissions`도 붙인다)
