# R-A8 작업 지시

review/review-08-memory-snapshots 브랜치의 memory snapshot export PR을 검토합니다.

초기 PR은 lifecycle as-of를 지정해 artifact bundle을 export하고, 그 시점의 active/inactive 수와 이유를 담은 snapshot manifest를 추가합니다. 하나의 snapshot이라는 이름에 맞게 모든 파생 artifact가 같은 시점과 시간대 계약을 따르는지 확인합니다.

## 범위

- 초기 PR: brainwash/memory/ledger.py
- 리뷰 반영: brainwash/memory/ledger.py, tests/test_memory_ledger.py
- 기준 commit: 9605168
- 최초 검토 commit: 1d6cd78

## 기대 산출물

- 실제 snapshot contract를 다루는 리뷰 스레드 5개
- artifact boundary와 timezone input에 대한 Change Request 2개
- Python datetime 공식 문서와 예시를 포함한 정보 공유
- 최초 PR 설명, 가상 프롬프트, 최종 보고서, 캡처 목록, 체크리스트
