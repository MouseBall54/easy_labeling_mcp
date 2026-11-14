# Windows 클라이언트 설정

Easy Labeling을 먼저 실행하고 이 작업 폴더에서 클라이언트를 시작한다. 기본 서버 파일은 `$env:LOCALAPPDATA\Easy Labeling\easy-labeling-mcp.exe`다. 실제 경로는 앱의 **MCP → Connect**에서도 확인할 수 있다. 어느 클라이언트든 첫 검증은 `app_state`를 호출하고 앱 Activity의 호출과 대조하는 것이다.

## Claude Code

`CLAUDE.md`가 `AGENTS.md`를 가져온다. 프로젝트 범위 로컬 등록 예시는 PowerShell 기준이다.

```powershell
$elMcp = Join-Path $env:LOCALAPPDATA 'Easy Labeling\easy-labeling-mcp.exe'
claude mcp add --transport stdio --scope local easy-labeling -- "$elMcp"
claude mcp get easy-labeling
```

채팅의 `/mcp`에서 연결을 확인한다. 이미지 없이 실행할 작업만 하는 세션은 아래 명령으로 캡처를 숨길 수 있다. 이 세션에서는 시각적 판정이 필요한 후보를 보류한다.

```powershell
claude --disallowedTools 'mcp__easy-labeling__capture_view'
```

프로젝트 내에서 계속 수치 전용으로 쓰려면 `.claude/settings.local.json`의 기존 `permissions.deny` 배열에 `mcp__easy-labeling__capture_view`를 추가한다. 일반 검수 세션은 이 제한이 없는 설정을 사용한다. [MCP 등록](https://code.claude.com/docs/en/mcp) · [프로젝트 지침](https://code.claude.com/docs/en/memory) · [도구 권한](https://code.claude.com/docs/en/permissions)

## OpenCode V2

`AGENTS.md`를 읽는다. 이 프로젝트의 `opencode.jsonc`에서 아래 서버 항목을 기존 설정에 합친다. 경로는 앱에 표시된 실제 경로로 바꾼다. `codemode:false`로 Easy Labeling 도구를 모델에 개별 노출한다.

```json
{
  "mcp": {
    "servers": {
      "easy-labeling": {
        "type": "local",
        "command": ["C:/Users/YOUR_NAME/AppData/Local/Easy Labeling/easy-labeling-mcp.exe"],
        "codemode": false
      }
    }
  }
}
```

수치 전용 프로젝트 구성에서는 최상위 `permissions` 배열에 다음 항목을 추가한다. 이 항목이 있는 동안 `capture_view`를 사용할 수 없다.

```json
{"action":"easy-labeling_capture_view","resource":"*","effect":"deny"}
```

`opencode mcp list`로 연결 상태를 확인하고, 프로젝트 채팅에서 `app_state`도 호출한다. Windows V2 `2.3.0` 시험에서는 목록 명령이 서버를 표시하지 않았지만, 프로젝트에서 실행한 모델은 `app_state`와 `analyze_boxes`를 성공적으로 호출했다. 모델은 캡처 도구가 목록에 없다고 보고했다. [V2 MCP 설정 및 권한](https://opencode.ai/docs/mcp-servers/) · [프로젝트 지침](https://opencode.ai/docs/instructions/)

### OpenCode V1

`AGENTS.md`를 읽는다. V1은 `mcp` 바로 아래에 서버를 등록한다.

```json
{
  "mcp": {
    "easy-labeling": {
      "type": "local",
      "command": ["C:/Users/YOUR_NAME/AppData/Local/Easy Labeling/easy-labeling-mcp.exe"]
    }
  }
}
```

V1에서 수치 전용 설정이 필요하면 최상위 `permission` 객체에 `"easy-labeling_capture_view": "deny"`를 추가한다. [V1 MCP 설정](https://opencode.ai/v1/docs/mcp-servers/) · [V1 권한](https://opencode.ai/v1/docs/permissions/)

Windows에서 V1 `1.19.5`로 검증했다. `opencode debug agent build`에서 캡처 도구의 거부 규칙이 적용됐고, 모델은 `app_state`와 `analyze_boxes`만 호출했다. [실측 기록](BENCHMARK.md#opencode-수치-전용-검증)

## Codex

`codex mcp get easy-labeling`으로 등록을 확인한다. 필요하면 앱 MCP 패널의 **Connect → Codex CLI** 설정을 사용한다. 새 세션에서 `app_state`를 호출한다. 편집이 승인 단계에서 차단되면 해당 적용을 종료하고 조회 결과까지 보고한다.

Codex의 편집 경로를 검증하려면 승인 요청을 받을 수 있는 대화형 세션에서 테스트 복사본을 먼저 연다. 시드 `1` 검증 복사본을 사용하는 요청 예시는 아래와 같다. 기존 검증의 `never` 정책에서는 `update_boxes`가 서버에 전달되기 전에 거절됐다.

> 이 테스트 복사본의 `sample_metal_cross_section.png`만 처리해. `app_state`로 대상 확인 → `get_boxes`로 클래스 11 조회 → 확인된 8개만 `update_boxes` 한 번으로 클래스 10으로 변경 → `save` → `list_images {"storage":true,"images":["sample_metal_cross_section.png"]}`로 디스크 경로와 박스 수를 대조하고 바로 보고해. 캡처는 하지 마.

## 공통 확인 요청

> `app_state`를 한 번 호출하고 현재 앱·서버 버전, 폴더, 작업 모드를 보고해줘.

> 수치 전용으로 현재 이미지의 겹침과 빈자리 후보 수만 알려줘. 캡처·편집·저장은 제외해.

앱을 다시 열었거나 연결이 끊겼으면 클라이언트에서 다시 연결하거나 새 세션을 시작한다. 이미지 결과를 표시할 수 있는지는 클라이언트와 선택한 모델의 조합에서 확인한다. 이미지 내용을 판독할 수 없으면 수치로 확정 가능한 범위까지 진행한다.
