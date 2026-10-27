---
name: herdr-duo
description: herdr 터미널 워크스페이스에서 왼쪽 Claude Code + 오른쪽 agy(Antigravity CLI, Gemini) 2인 협업 세팅을 다시 만들고 점검한다. 창 분할·agy 실행, agy의 herdr 명령 자동 허용 규칙, AGENTS.md 메시지 프로토콜, 양방향 메시지 테스트, agy 재시작까지 포함. 사용자가 "herdr 세팅 불러와", "제미나이/agy 오른쪽에 띄워줘", "agy랑 같이 작업할 수 있게 해줘", "herdr 듀오", "agy한테 메시지 보내기 세팅", "agy가 승인 계속 물어봐" 같은 말을 하거나, 새 프로젝트 폴더에서 Claude와 agy를 나란히 쓰고 싶어할 때 반드시 이 스킬을 사용한다.
---

# herdr-duo: Claude(왼쪽) + agy(오른쪽) 협업 세팅

herdr 안에서 Claude Code(왼쪽 창)와 agy(오른쪽 창, Gemini)가 같은 폴더에서 나란히 작업하고
`herdr pane run`으로 서로 메시지를 주고받는 세팅을 재현한다.

아래 단계는 **이미 되어 있는 것은 건너뛰도록** 점검 → 필요한 것만 수정하는 방식으로 진행한다.
사용자의 기존 설정을 덮어쓰지 않는 것이 중요하다 (특히 agy settings.json에는 다른 설정도 들어 있다).

## 0. 전제 확인

```sh
herdr pane current      # herdr 안에서 실행 중인지 + 내 pane ID
```
실패하면 herdr 밖이다. 사용자에게 herdr를 먼저 띄우고(`herdr`) 그 안에서 `claude`를 실행해 달라고 안내하고 멈춘다.

작업 폴더(이하 `<CWD>`)는 사용자가 지정한 폴더, 없으면 현재 작업 디렉터리다.

## 1. 오른쪽 agy 창 준비

pane ID(예: `w4:p2`)는 창을 닫았다 열면 바뀐다. 절대 하드코딩하지 말고 매번 조회한다.

```sh
herdr pane neighbor --current --direction right   # result.neighbor.neighbor_pane_id
herdr pane list                                    # 각 pane의 agent / agent_status
```

- 오른쪽 이웃이 있고 `agent: "agy"` → 이미 준비됨. 그 ID를 `<AGY>`로 쓴다.
- 오른쪽 이웃이 있지만 agy가 아님(빈 셸) → 그 창에서 agy를 실행한다.
- 오른쪽 이웃이 없음 → 분할 후 실행:
  ```sh
  herdr pane split --current --direction right --cwd "<CWD>" --no-focus
  herdr pane neighbor --current --direction right   # 새 pane ID 확인
  ```

### 권한 모드 정하기

agy를 실행하기 전에 권한 모드를 정한다. 사용자의 메모리·CLAUDE.md에 이미 선호가 적혀 있으면 그대로 따르고, 없으면 두 가지를 설명하고 고르게 한 뒤 메모리에 기록한다.

- **허용 규칙 모드 (기본)**: 2번의 허용 규칙으로 herdr 명령만 자동 허용하고, 나머지 명령은 agy가 승인을 묻는다. 안전하지만, 규칙에 없는 명령(예: `sleep`)을 쓰면 승인 대기로 멈춘다.
- **전체 자동 승인 모드**: `--dangerously-skip-permissions`로 실행해 모든 권한 요청을 묻지 않고 승인한다. 승인 대기로 멈추는 일은 없어지지만 파일 삭제 같은 명령도 그대로 실행되므로, 6번 기준으로 지시문을 엄격하게 쓰는 것이 유일한 안전장치가 된다.

실행 명령 (Windows 기준. PATH가 갱신 안 된 터미널이 있으므로 전체 경로를 쓴다. 다른 OS에서는 `agy`):
```sh
# 허용 규칙 모드
herdr pane run <AGY> '& "$env:LOCALAPPDATA\agy\bin\agy.exe"'          # 새 대화
herdr pane run <AGY> '& "$env:LOCALAPPDATA\agy\bin\agy.exe" -c'       # 직전 대화 이어가기
# 전체 자동 승인 모드: 위 명령 끝에 --dangerously-skip-permissions 를 붙인다
herdr pane run <AGY> '& "$env:LOCALAPPDATA\agy\bin\agy.exe" --dangerously-skip-permissions -c'
```
전체 자동 승인 모드를 골랐는데 이미 떠 있는 agy가 승인 프롬프트를 띄우면, 옵션 없이 실행된 것이므로 5번 절차로 재시작한다.
다른 워크스페이스의 agy는 그쪽 Claude 세션의 짝이므로 가져다 쓰거나 재시작하지 않는다.
몇 초 뒤 `herdr pane read <AGY> --lines 20`으로 `Antigravity CLI` 배너가 떴는지 확인한다.

## 2. agy 자동 허용 규칙 점검

허용 규칙 모드에서는 이 규칙이 핵심이다. 전체 자동 승인 모드에서도 옵션 없이 실행됐을 때를 대비한 안전망으로 넣어 둔다.

파일: `~/.gemini/antigravity-cli/settings.json` → `permissions.allow` 배열.

먼저 읽고, 아래 규칙 중 **빠진 것만** 배열에 추가한다 (다른 키·규칙은 그대로 둔다).
Windows에서는 따옴표·공백·한글이 섞인 명령에 prefix 규칙이 안 먹기 때문에 `regex:` 규칙이 핵심이다.

```json
"command(herdr pane list)",
"command(herdr pane neighbor)",
"command(herdr pane read)",
"command(herdr pane wait-output)",
"command(herdr pane run)",
"command(herdr pane send-text)",
"command(herdr pane send-keys)",
"command(regex:herdr pane (run|send-text|send-keys|read|wait-output|neighbor|list) .*)",
"command(regex:herdr pane (run|send-text|send-keys|read|wait-output|neighbor|list))",
"command(regex:herdr --help.*)",
"command(regex:herdr pane [a-z-]+ --help)"
```

규칙을 바꿨으면 agy가 이미 떠 있는 경우 **재시작해야 적용된다** (5번 절차).

`powershell -Command "Start-Sleep ...; herdr ..."` 같은 복합 명령은 일부러 허용하지 않는다
(임의 명령 실행을 열어주게 됨). 대신 메시지에 "다른 명령은 실행하지 마"라고 적어 agy가 폴링하지 않게 한다.

## 3. 프로젝트 프로토콜 파일

`<CWD>/AGENTS.md`가 없으면 `assets/AGENTS.md`를 복사한다. 있으면 herdr 메시지 섹션이 들어 있는지만 보고, 없으면 사용자에게 추가할지 묻는다.
`<CWD>/CLAUDE.md`가 없으면 `@AGENTS.md` 한 줄로 만든다 (Claude Code가 AGENTS.md를 읽게 하는 import).
agy는 AGENTS.md를 직접 읽는다.

## 4. 양방향 연결 테스트

agy가 `idle`인지 `herdr pane list`로 확인한 뒤 보낸다 (작업 중이면 입력이 끼어든다).
화면에 "esc to dismiss" 배너가 있으면 먼저 `herdr pane send-keys <AGY> esc`.

```sh
herdr pane run <AGY> "[from Claude] 연결 테스트야. herdr pane run <MY_PANE> 으로 '[from agy] 연결 OK' 라고만 답장하고, 다른 명령(sleep 등)은 실행하지 마."
```

`pane run`은 타이핑만 할 뿐이라 답장이 자동으로 돌아오지 않는다. agy의 답장은 내 입력창에
새 사용자 메시지처럼 들어온다 (내가 작업 중이면 큐에 쌓였다가 전달됨).
진행 상황은 `herdr pane read <AGY> --lines 20`으로 본다.

- 답장이 왔다 → 성공.
- agy 화면에 `Requesting permission for:`가 떠 있다 → 허용 규칙에 빠진 명령이다. 사용자가 대신 승인해 달라고 했으면 `herdr pane send-keys <AGY> enter`(1번 "Yes" 선택). 반복될 명령이면 2번 규칙에 추가하고 재시작한다.

테스트 메시지를 여러 번 보내면 답장도 여러 번 온다. 앞 요청이 지연 처리되면 중복 답장이 생길 수 있으니 테스트는 한 번만 보내고 기다린다.

## 5. agy 재시작 절차

agy가 작업 중(명령 실행·승인 대기)일 때 `ctrl+c`는 **현재 작업만 취소**하고 종료되지 않는다.
그 상태에서 바로 실행 명령을 보내면 셸이 아니라 agy 입력창에 메시지로 들어가 버린다.

```sh
herdr pane send-keys <AGY> esc       # 승인 프롬프트 닫기
herdr pane send-keys <AGY> ctrl+c    # 작업 중단
herdr pane send-keys <AGY> ctrl+c    # 종료
herdr pane read <AGY> --lines 5      # 마지막 줄이 "PS C:\...>" 셸 프롬프트인지 확인
```
셸 프롬프트가 보일 때까지 `ctrl+c`를 반복한 뒤에만 1번의 실행 명령(정한 권한 모드에 맞게, 보통 `-c`)을 보낸다.

## 6. agy에게 보내는 지시문 작성 기준

지시가 모호하면 agy가 "알아서" 범위를 넓히고, 그 결과는 되돌리기 어렵다.
전체 자동 승인 모드에서는 지시문이 사실상 유일한 안전장치다. 허용 규칙 모드에서도 승인 대기로 멈추는 일을 줄여 주므로 같은 기준으로 쓴다. 매 요청에 아래를 담는다.

1. **보낸 쪽 표시**: `[from Claude]`로 시작.
2. **작업 폴더**: 절대 경로를 적는다. 그 폴더 밖의 파일은 건드리지 말라고 명시한다.
3. **할 일을 구체적으로**: 무엇을, 어떤 파일에, 어떤 결과물로 할지. "정리해줘", "고쳐줘"처럼 범위가 열린 표현을 피한다.
4. **건드려도 되는 것 / 안 되는 것**: 수정할 파일을 나열하고, 그 외 파일 수정·삭제는 금지라고 적는다. 읽기만 하는 작업이면 "파일 생성·수정·삭제는 하지 마"라고 적는다.
5. **금지 명령**: 작업과 관계없으면 기본으로 넣는다. 파일·폴더 삭제(`rm`, `Remove-Item`, `del`), `git push` / `git reset --hard` / `git clean` / 브랜치 삭제, 패키지 전역 설치·제거, 시스템·레지스트리·환경변수 변경, 다른 herdr 창 조작(답장용 `pane run` 제외), 폴링용 `sleep` 반복.
6. **애매하면 멈추기**: "계획과 다르거나 확실하지 않으면 실행하지 말고 먼저 나한테 물어봐"라고 적는다.
7. **회신 방법**: `herdr pane run <MY_PANE> "[from agy] 완료: <요약>"` 형태로 끝나면 보고하게 한다. 실패했을 때도 `[from agy] 실패: <이유>`로 보고하게 한다.

예시:
```
[from Claude] 작업 폴더: C:\dev\my-game (이 폴더 밖은 건드리지 마).
할 일: assets/ 에 32x32 픽셀아트 기사 아이콘 knight.png 1개를 생성해.
허용: assets/knight.png 새로 만들기만. 다른 파일 수정·삭제, git 명령, 패키지 설치는 하지 마.
확실하지 않으면 실행하지 말고 먼저 물어봐.
끝나면 herdr pane run <MY_PANE> "[from agy] 완료: <파일 경로>", 실패하면 "[from agy] 실패: <이유>" 로 보내줘.
```

agy가 끝났다고 보고하면 결과를 직접 확인한다. 바뀐 파일을 보고, git 저장소면 `git status`/`git diff`로 지시 범위 밖의 변경이 없는지 본다.
범위 밖 변경이 있으면 되돌리기 전에 사용자에게 알린다.

## 마무리 보고

사용자에게 짧게 알린다: agy pane ID, 새로 추가한 허용 규칙(있으면), 생성한 파일(있으면), 연결 테스트 결과.
