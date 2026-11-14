# Easy Labeling MCP 운영 작업 공간

> 현재 라벨 폴더에서 검수가 필요한 이미지 5장을 찾고, 누락·중복 후보와 근거를 보여줘.

새 세션에 먼저 제공할 파일은 [MCP_PLAYBOOK.md](MCP_PLAYBOOK.md)입니다. 작업 대상과 원하는 결과를 알려주면, 안내에 따라 앱의 모드와 라벨 상태를 확인하고 필요한 절만 읽습니다. [Claude Code·OpenCode·Codex 연결 설정](CLIENTS.md)에는 수치 작업에서 `capture_view`를 도구 목록에서 막는 설정이 있습니다.

## 연결과 재연결

1. Easy Labeling 데스크톱 앱을 켜고 왼쪽의 **MCP** 패널을 연다.
2. `codex mcp get easy-labeling`으로 설치된 사이드카 등록을 확인한다. 등록이 필요하면 MCP 패널의 **Connect → Codex CLI** 예제를 사용한다.
3. 새 Codex 세션에서 `Easy Labeling의 app_state를 호출해줘`라고 요청한다. 앱 버전·서버 버전과 활성 폴더를 확인하고 MCP 패널의 **Activity**에서 같은 호출을 찾는다.
4. 앱을 닫으면 사이드카도 종료된다. 앱을 다시 열고 새 세션에서 `app_state`로 재연결을 확인한다. `busy`일 때는 현재 앱 작업이나 대화상자가 끝난 뒤 다시 읽는다.

## 검수 절차

3.13.0의 `list_images`로 쓰기 전 저장 상태와 충돌을 조회하고, `save` 응답을 저장 후 재조회와 대조한다. 호출 예시는 [저장 경로와 결과 확인](MCP_PLAYBOOK.md#저장-경로와-결과-확인)을 참고한다.

| 작업 | 순서 |
|---|---|
| Detection 읽기 전용 | `review_folder`로 이미지 우선순위 확인 → 이미지별 `analyze_boxes` → 필요하면 박스 묶음 또는 빈자리 영역 캡처 → 후보와 근거 보고 |
| Detection 수정 | 확정된 박스 편집을 묶어 호출 → 새 박스 위주로 `run_refine` 미리보기 → 결과 재조회 → 저장 및 다시 읽기 → 실제 검수 완료 이미지에 `Reviewed` |
| Segmentation | 거친 마스크는 `analyze_masks` → `clean_masks` 미리보기. 박스만 있으면 `boxes_to_masks`, 박스도 없으면 `segment_ai`의 박스 프롬프트. 면적과 인접 영역의 합침을 검사한 뒤 저장 |
| 폴더 작업 | `refine_folder` 또는 `reclass_folder`를 표본으로 `dryRun` → 요약 확인 → `resultId` 적용과 앱 승인 → 파일 및 백업 확인 |
| 템플릿 / YOLO | 기존 라벨 처리 정책을 확인하고 한 장에서 검증. 추론 후 `app_state`로 활성화된 `inference-*` 폴더와 클래스 매핑 확인 |

Reclass 전에 `get_refine_settings`로 대상·목적 클래스와 밝기 규칙을 확인한다. 새 클래스 파일은 `create_class_file`로 만들고 `get_classes`로 확인한다. 라벨 폴더 탭 전환, Refine Lab·Preprocess 편집, 템플릿 프리셋 생성은 앱에서 한다. 단일 이미지 편집은 Undo/Redo, 폴더 변경은 앱이 만든 백업에서 복구한다. 테스트용 복구는 해당 데이터셋의 백업 파일만 대상으로 한다.

- [MCP 사용 안내](../easy_labeling/docs/AGENT_MCP_GUIDE_KO.md)
- [앱 기능 및 옵션](../easy_labeling/docs/FEATURES_AND_OPTIONS_KO.md)
- [원본 스킬 및 측정](../easy_labeling/docs/agent-skills/easy-labeling-mcp/SKILL.md)
- [이 작업 공간의 검증 기록](BENCHMARK.md)
