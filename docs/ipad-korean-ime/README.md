# iPad Korean IME Terminal Support

이 fork는 iPadOS WebKit 브라우저에서 code-server 통합 터미널에 한글을 입력할 때 자모가 분리되어 shell로 전달되는 문제를 보정합니다.

## Scope

- 지원 대상은 iPadOS Safari/Chrome의 한글 IME입니다.
- 일본어/중국어 IME 후보 변환은 지원 범위가 아닙니다.
- macOS, Windows, Linux desktop 브라우저는 기존 xterm.js 입력 경로를 그대로 사용합니다.
- 목표는 터미널 입력 보정이며 editor, search input, chat input의 IME 동작은 변경하지 않습니다.

## How It Works

code-server는 VS Code 소스를 `lib/vscode` submodule로 가져오고 `patches/`의 quilt patch를 적용합니다. 이 기능도 같은 방식으로 관리합니다.

핵심 patch는 `patches/ipad-korean-ime-terminal.diff`입니다. 이 patch는 VS Code 통합 터미널 생성 시점에 `IpadKoreanImeBridge`를 연결합니다.

브리지는 다음 조건에서만 활성화됩니다.

- `codeServer.terminal.iPadKoreanImeMode`가 `on`
- 또는 기본값 `auto`이고 브라우저가 iPadOS WebKit으로 감지되는 경우

활성화되면 브리지는 xterm.js를 교체하지 않고 입력 경계만 보정합니다.

- `compositionstart`, `compositionupdate`, `compositionend`, `beforeinput`, `input` 이벤트를 추적합니다.
- xterm.js의 `raw.onData` 앞에서 한글 자모 입력을 가로챕니다.
- `ㄱㅏㄴㅏㄷㅏ`처럼 compatibility jamo가 직접 들어오면 두벌식 규칙으로 `가나다`를 조합한 뒤 shell로 보냅니다.
- 조합 중복을 피하기 위해 bridge가 이미 보낸 한글 데이터는 짧은 시간 xterm.js raw data에서 무시합니다.
- Enter, Tab, Escape, 공백, ASCII 입력 전에 pending 한글 조합을 flush합니다.
- Enter 같은 boundary key는 pending 한글과 같은 write로 묶어 보냅니다. 이렇게 해야 raw-mode TUI가 Enter를 먼저 처리해 마지막 글자를 잃는 일을 막을 수 있습니다.

## Setting

```json
{
  "codeServer.terminal.iPadKoreanImeMode": "auto"
}
```

지원 값:

- `auto`: iPadOS WebKit에서만 bridge를 켭니다. 기본값입니다.
- `on`: 현재 브라우저 세션에서 항상 bridge를 켭니다.
- `off`: bridge를 끄고 기존 xterm.js 입력을 사용합니다.

## Changed Files

```text
patches/
  series
  ipad-korean-ime-terminal.diff

docs/
  ipad-korean-ime/
    README.md

lib/vscode/src/vs/platform/terminal/common/
  terminal.ts

lib/vscode/src/vs/workbench/contrib/terminal/browser/
  ipadKoreanImeBridge.ts
  terminalInstance.ts

lib/vscode/src/vs/workbench/contrib/terminal/common/
  terminalConfiguration.ts
```

`lib/vscode` 아래 파일은 직접 commit하지 않고 `patches/ipad-korean-ime-terminal.diff` 안에 보관합니다.

## Test Cases

실기기 acceptance에서 확인할 입력:

- `한글`
- `안녕하세요`
- `값`
- `괜찮다`
- `한글 test 123`
- `cd 한글폴더`
- 공백, Enter, Backspace가 섞인 입력

터미널 앱 시나리오:

- bash/zsh prompt
- Python REPL
- vim insert mode
- nano
- tmux pane

회귀 확인:

- desktop 브라우저의 기존 xterm.js 입력은 변경되지 않아야 합니다.
- Ctrl+C, Ctrl+D, Tab, arrow keys, paste, bracketed paste, selection/copy가 유지되어야 합니다.

## Current Verification

이 작업에서 확인한 결과:

- `npm run typecheck-client`: 통과
- `npm run compile-client`: 통과
- iPad Pro 13-inch Simulator Safari:
  - `ㄱㅏㄴㅏㄷㅏ` raw 자모 입력이 `가나다`로 shell에 전달됨
  - `ㅎㅏㄴㄱㅡㄹ test 123 ㄱㅏㅂㅅ ㄱㅗㅐㄴㅊㅏㄴㅎㄷㅏ` raw 자모/ASCII 혼합 입력이 `한글 test 123 값 괜찮다`로 shell에 전달됨
  - `ㄴ`과 `ㅏ` 사이를 약 1.8초 지연해도 `나`로 전달됨
  - `값싼닭갈비가나다라`를 자모 단위 key action으로 입력해도 같은 문자열로 전달됨
  - raw-mode TUI stdin reader에서도 `값싼닭갈비가나다라`가 같은 문자열로 전달됨
  - partial composition 최종값이 `ㅏ`만 제공되고 이전 partial 이벤트가 `ㄴ`인 경우에도 `아`가 아니라 `나`로 전달됨
  - raw-mode TUI에서 Backspace가 조합 중/조합 완료 후 각각 자연스럽게 동작함
  - Enter submit 직전 pending 글자가 `각각가`, `괜찮다`, `읽다`, `앉다`에서 누락되지 않음

Simulator 결과는 smoke/regression 증거입니다. 최종 acceptance는 실제 iPadOS Safari/Chrome의 software keyboard와 hardware keyboard에서 확인합니다.

## Development Workflow

patch를 수정할 때:

```bash
git submodule update --init
quilt push -a

# edit lib/vscode files

quilt refresh
npm run typecheck-client --prefix lib/vscode
```

실기기 테스트용 dev server:

```bash
npm run watch -- --bind-addr=0.0.0.0:18080 --auth=none --disable-workspace-trust /Users/m1ns2o128/code-server-ipad
```

같은 Wi-Fi의 iPad에서 Mac의 LAN IP와 포트로 접속합니다.

## Maintenance Notes

- 한글 조합기는 compatibility jamo 범위 `U+3131..U+3163`만 처리합니다.
- 완성형 한글이 composition event로 정상 전달되는 경우에는 그대로 전달합니다.
- 같은 자모가 실제로 반복 입력되는 `닭갈` 같은 케이스를 보존하기 위해 raw 자모 입력은 최근 duplicate cache에 넣지 않습니다.
- 마지막 조합 글자와 Enter는 한 번의 terminal data write로 전달합니다. 별도 async write로 보내면 raw-mode 앱이 Enter를 먼저 처리해 마지막 글자를 놓칠 수 있습니다.
- 후보 선택, 일본어/중국어 변환, punctuation 변환 보정은 의도적으로 구현하지 않았습니다.
- bridge 실패 시 다음 대안은 xterm.js 교체가 아니라 `ghostty-web` API 호환성 검토입니다.
