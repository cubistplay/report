# R-A7 작업 지시

review/review-07-promotion-candidates 브랜치의 promotion candidate 조회 PR을 검토합니다.

초기 PR은 Update DB의 active memory를 subject/relation/scope 단위로 묶어 promotion threshold를 충족한 후보를 반환합니다. 후보 결과는 training 전 검토, audit, dataset export에서 재사용될 수 있으므로 결정성과 기존 policy의 target 의미를 중점적으로 확인합니다.

## 범위

- 초기 PR: brainwash/memory/update_db.py
- 리뷰 반영: brainwash/memory/update_db.py, tests/test_update_db.py
- 기준 commit: 8e32f99
- 최초 검토 commit: 58a5609

## 기대 산출물

- 중요도와 Change Request 여부를 구분한 리뷰 스레드 5개
- 결정적 결과와 target 정규화를 위한 독립 test/fix commit
- SQLite 공식 문서와 재현 예시를 포함한 정보 공유 1건 이상
- 최초 PR 설명, 가상 프롬프트 히스토리, 최종 보고서, 캡처 목록, 체크리스트
