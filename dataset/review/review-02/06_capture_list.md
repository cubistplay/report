# R-A2 캡처 목록

> 실제 원격 PR 화면은 생성하지 않았습니다. 아래는 로컬 합성 PR 기록을 게시할 때 사용할
> 캡처 이름과 근거입니다.

| 캡처명 | 대상 | 근거 |
| --- | --- | --- |
| `R-A2-01-initial-pr.png` | 최초 PR 설명 | `0509d92` 시점, history API test 없음 |
| `R-A2-02-mutable-history-review.png` | internal list 노출 Change Request | `history.clear()` / `history.append()` 재현 code |
| `R-A2-03-frozen-record-guidance.png` | nested list mutation 안내 | frozen field assignment 설명과 `.append()` 예시 |
| `R-A2-04-response-commits.png` | test → fix 리뷰 반영 이력 | `ae1d70b` → `9d3eda3` |
| `R-A2-05-final-verification.png` | focused·전체 test | 9/9, 96/96 통과 |
| `R-A2-06-mainline-log.png` | 선형 Git 이력 | Review-01 → initial PR → review response |
