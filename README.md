# herdr-duo

[herdr](https://herdr.dev) 터미널 워크스페이스에서 **Claude Code(왼쪽 창)** 와 **agy(Antigravity CLI, Gemini — 오른쪽 창)** 가
같은 폴더에서 나란히 작업하고, `herdr pane run`으로 서로 메시지를 주고받게 만드는 Claude Code 스킬입니다.

Claude에게 "herdr 세팅 불러와" 또는 "agy 오른쪽에 띄워줘"라고 하면 아래를 알아서 점검·세팅합니다.

1. 오른쪽 창이 없으면 분할하고 agy 실행
2. agy가 herdr 명령마다 승인을 묻지 않도록 허용 규칙 추가 (`~/.gemini/antigravity-cli/settings.json`, 기존 설정은 유지)
3. 프로젝트에 메시지 프로토콜 `AGENTS.md` / `CLAUDE.md` 생성 (없을 때만)
4. 양방향 메시지 테스트
5. 필요하면 agy 안전하게 재시작

처음 쓸 때 agy 권한 모드를 고르게 됩니다.

- **허용 규칙 모드 (기본)**: herdr 명령만 자동 허용, 나머지는 agy가 승인을 물음
- **전체 자동 승인 모드**: `agy --dangerously-skip-permissions`. 멈추지 않는 대신 모든 명령이 실행되므로, 스킬이 agy에게 보내는 지시문을 범위·금지 명령·회신 형식까지 엄격하게 씁니다

## 필요한 것

- [herdr](https://herdr.dev)
- [Claude Code](https://claude.com/claude-code)
- agy (Antigravity CLI)
- Windows에서 만들고 테스트했습니다. 다른 OS에서는 agy 실행 경로 등을 조정해야 할 수 있습니다.

## 설치

### 방법 1: 플러그인으로 설치 (추천)

Claude Code에서:

```
/plugin marketplace add hun1228/herdr-duo
/plugin install herdr-duo@herdr-duo
```

### 방법 2: 스킬 폴더 복사

```sh
git clone https://github.com/hun1228/herdr-duo.git
# Windows (PowerShell)
Copy-Item -Recurse herdr-duo\skills\herdr-duo "$HOME\.claude\skills\"
# macOS / Linux
cp -r herdr-duo/skills/herdr-duo ~/.claude/skills/
```

## 사용

herdr 안에서 `claude`를 실행한 뒤:

- "herdr 세팅 불러와"
- "agy 오른쪽에 띄워줘"
- "agy가 승인 계속 물어봐"
- 또는 `/herdr-duo`

## 라이선스

MIT
