# R-A9 작업 지시

review/review-09-memory-trigger-batch 브랜치의 memory trigger batch API를 검토합니다.

초기 PR은 여러 prompt의 trigger decision과 summary를 반환합니다. evaluation consumer가 입력과 결과를 순서대로 비교할 수 있어야 하므로 input cardinality, 결과 순서, summary count를 중점적으로 검토합니다.

## 범위

- 초기 PR: brainwash/algorithms/memory_edit.py
- 리뷰 반영: brainwash/algorithms/memory_edit.py, tests/test_memory_edit_runtime.py
- 기준 commit: f8993d9
- 최초 검토 commit: 9a9d26d

## 기대 산출물

- batch result 계약을 다루는 리뷰 스레드 5개
- duplicate prompt와 triggered count에 대한 Change Request 2개
- Python dict 공식 문서와 예시를 포함한 정보 공유
- 최초 PR 설명, 가상 프롬프트, 최종 보고서, 캡처 목록, 체크리스트
