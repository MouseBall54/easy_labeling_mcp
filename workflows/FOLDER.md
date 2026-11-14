# 폴더 일괄 작업

Refine의 표본 선정과 적용 판단은 [Refine 가이드](REFINE.md#폴더-적용)를 함께 따른다.

폴더 전체에 새 박스나 마스크를 만드는 요청은 [폴더 전체 라벨링](LABEL_DATASET.md)을 따른다.

1. `app_state`로 데이터셋·활성 라벨 폴더·미저장 상태를 확인한다. `list_images`의 `storage:true`로 대상 이미지 범위와 충돌·읽기 오류를 확인한다. 수치 미리보기와 검증에는 캡처를 쓰지 않는다.
2. `refine_folder` 또는 `reclass_folder`에 `images` 또는 `sample`과 `dryRun:true`를 보낸다. Reclass 전에는 `get_refine_settings`로 입력·목적 클래스와 밝기 규칙을 확인한다. 마스크 정리는 `analyze_masks {"scope":"folder"}` 후 `clean_masks`에 같은 범위와 `apply:false`를 사용한다.
3. 변경 이미지·박스 수, 제외 항목, 이상치를 확인한다. 표본이 타당해 전체 적용이 요청되면 전체 범위를 다시 미리 본다. 표본의 `resultId`는 표본만 가리킨다.
4. 요청 범위가 맞으면 Refine/Reclass는 동일 도구에 `resultId`를, 마스크 정리는 확인한 범위·옵션과 `apply:true`를 보낸다. 앱 승인 거절이나 Stop이 오면 종료한다. 미리보기가 만료됐다면 최신 상태에서 다시 시작한다.
5. 반환된 변경 수, `skipped`, `saves`, 백업 경로를 기록한다. `clean_masks`의 `saves`는 요청한 결과 페이지의 범위다. `list_images`로 저장 경로와 결과 수치를 대조한다. 테스트 복사본에서 복원을 검증할 때는 해당 백업을 대상으로 복원 전후 해시를 대조한다.
