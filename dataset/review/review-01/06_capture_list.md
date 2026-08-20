# R-A1 캡처 목록

> 실제 원격 PR 화면은 생성하지 않았습니다. 아래는 로컬 합성 PR 기록을 게시할 때 사용할
> 캡처 이름과 근거입니다.

| 캡처명 | 대상 | 근거 |
| --- | --- | --- |
| `R-A1-01-initial-pr.png` | 최초 PR 설명 | `c5797e4` 시점, 신규 설정 test 없음 |
| `R-A1-02-cache-key-review.png` | model/provider cache key Change Request | `embedding`/`llm` key와 재현 경로 |
| `R-A1-03-test-guidance.png` | 환경 격리 테스트 안내 | 블록 안에서만 환경을 바꾸고 종료 뒤 자동 복구하는 `patch.dict` 예시와 factory seam |
| `R-A1-04-response-commits.png` | test → fix 리뷰 반영 이력 | `743003b` → `63f8afb` |
| `R-A1-05-final-verification.png` | focused·전체 test | 15/15, 93/93 통과 |
| `R-A1-06-mainline-log.png` | 선형 Git 이력 | Implement-05 → initial PR → review response |
