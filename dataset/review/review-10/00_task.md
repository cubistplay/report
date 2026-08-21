# R-A10 작업 지시

review/review-10-routing-lanes 브랜치의 mixed correction routing lane PR을 검토합니다.

초기 PR은 behavioral correction과 memory correction을 분리해 각 lane에 routing plan을 부여합니다. domain/broad correction이 factual memory policy를 덮어쓰지 않는지, 분리된 결과가 원래 input을 다시 찾을 수 있는지를 확인합니다.

## 범위

- 초기 PR: brainwash/router.py
- 리뷰 반영: brainwash/router.py, tests/test_router.py
- 기준 commit: 6b169f2
- 최초 검토 commit: 46f99d5

## 기대 산출물

- routing lane 책임 경계를 다루는 리뷰 스레드 5개
- broad lane 분리와 input position 복원을 위한 Change Request 2개
- Python enumerate 공식 문서와 예시를 포함한 정보 공유
- 최초 PR 설명, 가상 프롬프트, 최종 보고서, 캡처 목록, 체크리스트
