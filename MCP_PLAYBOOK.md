# Easy Labeling MCP 작업 시작 안내

새 작업을 받으면 이 파일을 먼저 읽는다. 아래 순서로 대상과 라벨 상태를 확인한 뒤, 해당하는 작업 문서의 절만 읽고 실행한다. MCP 연결이 안 되면 [클라이언트 설정](CLIENTS.md)을 연다.

공통 순서는 **앱 상태 확인 → 이미지별 작업 경로 선택 → 한 장에서 후보 확인 → 요청 범위만 적용 → 저장 결과 확인**이다. 데이터셋의 클래스와 경계 기준이 있으면 작업 전에 읽는다. 근거가 부족한 후보는 위치와 이유를 남긴다.

## 1. 대상과 작업 모드 확인

1. 첫 MCP 호출은 `app_state {}`다. `appVersion`, `mcpVersion`, `folder`, `labelFolder`, `currentImage`, `workflow`, `autoSave`, `unsavedChanges`를 확인한다. 버전이 다르면 `versionWarning`을 보고하고 앱과 함께 배포된 MCP 실행 파일로 재연결한다. 대상이 열리지 않았다면 지정된 대상을 연다. 편집 전에는 [저장 경로와 결과 확인](#저장-경로와-결과-확인)의 조회를 먼저 실행한다.
2. **Detection은 박스, Segmentation은 픽셀 마스크**다. 요청이 박스나 마스크를 지정하면 그 모드를 사용한다. 요청이 모드를 지정하지 않았고 현재 모드로 해석할 수 있으면 그대로 진행한다. 결과가 박스인지 마스크인지에 따라 작업이 달라지는데 판단할 근거가 없으면 확인한다.
3. 모드 전환이 필요한 작업은 `set_workflow {"workflow":"detection"}` 또는 `set_workflow {"workflow":"segmentation"}` 후 상태를 확인한다. 이미지나 모드를 바꾸기 전에 미저장 편집을 확인한다. Auto save가 켜져 있으면 이동이 저장을 일으킬 수 있다. 읽기 전용 폴더 검수는 가능한 한 이동 없이 저장된 라벨을 읽는다.
4. ‘찾아줘’, ‘검사해줘’, ‘보여줘’는 검수다. ‘고쳐줘’, ‘적용해줘’는 명시된 범위의 편집이다. ‘미리보기’가 포함되면 지원되는 미리보기 도구로 후보와 변경 예정 수까지 보고한다. 현재 이미지의 박스 추가만 요청받았으면 편집과 수치 확인에서 끝낸다. 저장 요청이 있으면 [저장 결과를 확인](#저장-경로와-결과-확인)한다.

## 2. 라벨 상태에 따라 문서 선택

라벨링 전 `get_classes {}`로 활성 클래스 이름과 ID를 확인한다. 적절한 파일이 없고 새 정의가 정해졌으면 `create_class_file {"name":"specimen","classes":[{"id":"1","name":"Boundary"}]}`로 새 YAML을 만들고 `get_classes`로 선택 결과를 확인한다. 이름이 겹치면 기존 파일이 보존된다.

- **Detection**에서는 현재 이미지의 `boxCount`를 사용한다. 개수가 없거나 작업 모드를 바꿨으면 필요할 때 `get_boxes {"limit":0}`의 `total`로 확인한다.
- **Segmentation**에서는 `mask.pixelsByClass`에 배경 클래스 `0` 이외의 양수 픽셀 수가 있는지 확인한다. 마스크 요약이 없으면 `get_regions {"limit":0}`을 한 번 호출한다. 마스크가 비었을 때만 Detection 박스 후보를 찾는다. 전환 전 확인한 박스 개수가 있으면 재사용하고, 없으면 `read_labels {"image":"현재 이미지 이름"}`으로 저장된 Detection 박스를 확인한다. `list_images`의 `labeled`와 `boxCount`도 Detection 기준이다.
- 여러 이미지의 상태가 섞여 있으면 `list_images {"storage":true,"offset":0,"limit":20}`으로 두 모드의 디스크 상태를 읽고 **이미지별로 분기**한다. 편집할 때는 해당 이미지를 열고 캔버스 상태도 확인한다. 마스크의 잡티·구멍 등 품질 검사는 `analyze_masks`를 사용한다.

| 요청과 시작 상태 | 다음에 읽을 절 | 첫 행동 |
|---|---|---|
| 폴더 전체 신규 라벨링 · Detection, Segmentation 또는 둘 다 | [폴더 전체 라벨링](workflows/LABEL_DATASET.md) | 앱에 로딩된 파일 목록 확정 → 라벨 경로 충돌 확인 → 첫 결과 적용·저장 |
| Detection · 박스 없음 · 신규 라벨링 | [빈 이미지에서 시작](workflows/DETECTION.md#빈-이미지에서-시작) | 클래스와 첫 물체의 근거를 확인하고 한 장에서 박스 생성 또는 후보 미리보기 |
| Detection · 박스 있음 · 검수, 보완, 크기별 분류 | [기존 박스 검수와 수정](workflows/DETECTION.md#기존-박스-검수와-수정) | 요청한 범위만 수치로 조회하고 확인된 변경만 적용 |
| Segmentation · 마스크 없음 · 신규 라벨링 · Detection 박스 있음 | [빈 마스크에서 시작](workflows/SEGMENTATION.md#빈-마스크에서-시작) | 박스와 마스크의 클래스 매핑을 확인하고 `boxes_to_masks` 미리보기 |
| Segmentation · 마스크 없음 · 신규 라벨링 · Detection 박스 없음 | [빈 마스크에서 시작](workflows/SEGMENTATION.md#빈-마스크에서-시작) | 좌표가 확인된 물체에 `segment_ai` 박스 프롬프트 또는 지정 도형 칠하기 |
| Segmentation · 기존 마스크 있음 | [기존 마스크 검수와 수정](workflows/SEGMENTATION.md#기존-마스크-검수와-수정) | `analyze_masks`로 문제 유형을 확인하고 필요한 곳만 정리 |
| Detection · 기존 박스 경계 Refine | [Refine 판단과 적용](workflows/REFINE.md) | 설정과 대상 확인 → 한 장 또는 폴더 표본 미리보기 |
| 폴더 범위 Reclass·마스크 정리 | [폴더 일괄 작업](workflows/FOLDER.md) | 지정 이미지나 표본부터 미리보기 |
| 템플릿·레이아웃·YOLO 사용 요청 | [자동화와 추론](workflows/AUTOMATION.md) | 한 장에서 프리셋·모델·결과 라벨 폴더 확인 |

검수나 정리가 요청됐는데 해당 라벨이 비어 있으면 빈 상태를 보고한다. 새 라벨 생성이 요청된 경우에만 신규 경로로 진행한다. 기존 라벨을 보완할 때는 누락·오류 영역만 대상으로 한다.

## 3. 진행과 종료

- 캡처 후에는 근거가 충분한 항목을 적용·검증하거나 다음 후보로 이동한다. 판단에 필요한 정보가 여전히 부족하면 미확정 사유를 남긴다. 클래스나 대상 선택을 위임받았으면 확인 가능한 대표 객체와 클래스 기준부터 정한다. 사용자의 답이 필요한 항목은 구체적으로 기록하고 처리 가능한 이미지부터 진행한다.
- 변경 후 도구 응답과 수치로 확인한다. 저장을 요청받았으면 저장 결과를 재조회하고, 검증 수치를 확보하면 바로 보고한다. 이미지 이동 후 박스 ID를 새로 읽고, 마스크 편집 후에는 영역 `seed`를 다시 읽는다. 자세한 공통 규칙은 [AGENTS.md](AGENTS.md)에 있다.

### 캡처와 좌표

1. 수치만으로 판단할 수 있으면 캡처를 생략한다. 새로 그릴 물체의 위치를 화면에서 찾아야 하면 박스가 없는 이미지도 캡처한다. 이미지당 필요한 화면 1회를 기본으로 하며, 검수할 박스는 최대 36개씩 `annotationIds`로 묶는다. 화면에 물체가 잘리면 원본 이미지 픽셀의 `region`으로 필요한 범위를 받는다.
2. `unchanged: true`는 앱이 앞선 캡처와 같은 PNG를 생략했다는 응답이다. `captureId`가 같은 PNG를 실제로 볼 수 있으면 이번 응답의 매핑과 함께 사용한다. 이전 버전에 `captureId`가 없으면 같은 요청에서 해당 PNG를 받았다는 근거가 있을 때 재사용한다. 앞선 PNG를 볼 수 없고 시각 판단이 필요하면 같은 인자에 `force: true`를 넣어 한 번 다시 받는다. `force` 뒤에도 이미지 블록이 안 보이면 클라이언트의 전달 여부를 확인한다. 이미지가 보이지만 윤곽을 판독할 수 없거나 위치 근거가 여전히 부족하면 해당 후보만 보류한다. 좌표, 모델, 템플릿 등으로 확인된 항목은 계속 처리한다.
3. 뷰포트와 단일 영역에서는 `imageX = pngX * scaleX + offsetX`, `imageY = pngY * scaleY + offsetY`로 PNG의 네 경계를 이번 응답의 `imageFromCapture`에 대입한다. 묶음에서는 각 타일 안의 위치에만 `imageX = (pngX - cell.x) * scale + offsetX`, `imageY = (pngY - cell.y) * scale + offsetY`를 적용한다. `scale`과 오프셋은 해당 타일의 `imageFromCell`에서 가져온다. PNG의 `width`, `height` 기준으로 좌표를 읽는다. 예를 들어 PNG가 절반 크기로 표시되면 표시 좌표를 2배로 환산한 뒤 매핑한다.
4. `add_boxes`와 `update_boxes`에는 변환 후의 원본 이미지 픽셀 경계를 보낸다. `0 ≤ x0 < x1 ≤ 이미지 너비`, `0 ≤ y0 < y1 ≤ 이미지 높이`를 확인한다. `x1`, `y1`은 오른쪽·아래쪽 경계라 이미지 크기와 같을 수 있다. `read_labels`의 정규화 경계는 X에 이미지 너비, Y에 이미지 높이를 곱한다. `analyze_boxes`는 응답의 `units`를 확인한다. 편집 후 ID로 `get_boxes`를 조회해 네 변과 클래스·개수를 비교한다.

## 저장 경로와 결과 확인

앱과 MCP 서버가 모두 3.13.0 이상일 때 아래 절차를 사용한다. 배포 버전별 응답 예시는 [MCP 사용 가이드](../easy_labeling/docs/AGENT_MCP_GUIDE_KO.md#이미지별-저장-경로와-디스크-상태)를 참고한다.

1. `list_images {"storage":true,"images":["a.png"]}`에서 해당 이미지의 `storage`를 `workflow`로 찾아 활성 `labelFolder`, 실제 `paths`, `conflictingImages`, `files`를 확인한다. `conflictingImages`에는 페이지나 필터 밖 이미지도 포함된다. 충돌이 있거나 저장 위치가 없거나 읽기 오류가 나면 해당 출력을 보류한다. 대상이 여러 장이면 반환된 `total`까지 페이지를 읽는다. 기존 `labeled`·`unlabeled`·`issues` 필터는 Detection 목록 기준이므로 두 모드의 파일 상태를 확인할 때는 생략한다.
2. `files[].status`는 `missing`(파일 없음), `empty`(빈 라벨, 배경만 있는 마스크, 주석 없는 내보내기), `present`(내용 있음), `read_error`(읽기 또는 해석 오류)다. `images`를 지정하면 저장된 박스 수 또는 PNG의 클래스별 픽셀·연결 영역 수도 조회할 수 있다. `canvas`와 `unsavedChanges`는 캔버스 쪽 상태다. 파일이 비었거나 없으면 대상 없음과 검수 완료는 아직 판정되지 않은 상태다.
3. 저장 요청이면 `save {}`의 `success`, `image`, `workflow`, `writtenPaths`, `action`을 확인한다. 동일 이미지로 `list_images`를 재호출해 경로와 Detection의 `boxCount` 또는 Segmentation PNG의 `pixelsByClass`·`regionsByClass`를 대조한다. `written_empty`는 빈 결과를 파일에 쓴 경우다. 클래스별 Detection 집계가 필요하면 `get_boxes {"limit":0}`을 쓰고, 저장 좌표가 필요하면 `read_labels`로 읽는다.
4. 실패 응답의 `structuredContent`가 있으면 대상과 경로를 확인한다. `partially_written`이나 호출 끊김이 발생하면 캔버스와 디스크를 먼저 재조회하고, 적용 여부가 확인될 때까지 편집을 반복하지 않는다.

## 오류 처리와 결과 보고

- `busy`이면 현재 작업이 끝났는지 `app_state`로 확인하고 대상이 일치할 때 재시도한다. `not_found`가 ID 또는 `resultId`에서 발생하면 현재 이미지의 ID를 다시 읽거나 미리보기를 새로 실행한다.
- `declined`, Stop, Codex 승인 거절이 발생하면 해당 적용을 종료한다. 가능한 저장 라벨 조회는 계속하고 어떤 호출이 막혔는지 보고한다. 편집 결과가 불명확하면 앱 상태와 파일부터 조회한다.
- 끝에 **대상 폴더·이미지 수 / 확인한 후보 / 적용한 변경 / 저장 확인 / 남은 검수**를 짧게 보고한다. 읽기 결과와 실제 저장 결과를 구분한다.
- 전송 직전에 각 숫자의 출처를 확인한다. `review_folder`는 폴더 품질 규칙, `analyze_boxes`는 기하학적 후보, `compare_labels`·`compare_masks`는 정답 비교 결과다. 도구가 반환하지 않은 값은 빈칸으로 남기거나 필요한 조회를 한 번 추가한다.

### 새 채팅에 넣을 요청 예시

> MCP_PLAYBOOK.md부터 읽어. 현재 열린 폴더의 모든 이미지에 Detection과 Segmentation을 진행해. 대상 클래스는 확인 가능한 대표 객체로 정하고, 이미지별 저장 결과와 보류 사유를 로그에 남겨줘.

> MCP_PLAYBOOK.md부터 읽어. `a.png` 한 장에서 지정한 대상의 Detection을 처리해. 먼저 저장 경로와 기존 라벨을 확인해. 검증된 모델이나 대표 템플릿으로 후보를 묶어 만들고, 겹침·빈자리와 명암이 다른 구역의 표본만 확인해. 확정된 수정만 적용·저장하고 경로와 박스 수가 디스크 재조회와 일치하면 종료해. 보류한 위치와 이유를 남겨줘.

> MCP_PLAYBOOK.md부터 읽어. 현재 라벨 폴더에서 우선순위가 높은 이미지 5장의 박스를 검수하고 겹침·누락 후보를 보여줘.

> MCP_PLAYBOOK.md부터 읽어. 현재 이미지에 마스크가 없으면 기존 박스를 마스크로 변환할 수 있는지 미리 봐줘. 마스크가 있으면 연결 영역과 누락 후보만 검사해.
