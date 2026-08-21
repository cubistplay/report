# R-A6 작업 지시

`review/review-06-lifecycle-audit` 브랜치에 올라온 lifecycle audit import PR을 검토합니다.

초기 PR은 `memory_lifecycle.jsonl`의 audit row를 Update DB에 적재하고 조회할 수 있게 합니다.
리뷰에서는 외부 JSON artifact의 타입 계약과 import 결과의 관찰 가능성을 우선 확인합니다.

## 범위

- 초기 PR: `brainwash/memory/schema.sql`, `brainwash/memory/update_db.py`
- 리뷰 반영: `brainwash/memory/update_db.py`, `tests/test_update_db.py`
- 기준 commit: `6fcf422`
- 최초 검토 commit: `b61a2d0`

## 기대 산출물

- 문제와 확인 질문을 구분한 리뷰 대화 5개
- 실제 수정이 필요한 Change Request 2개와 독립 반영 commit
- 공식 Python 문서를 근거와 짧은 예시로 제공하는 정보 공유 1건 이상
- 최초 PR 설명, 프롬프트 히스토리, 최종 보고서, 캡처 목록, 체크리스트
