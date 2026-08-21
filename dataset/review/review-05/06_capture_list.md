# R-A5 캡처 목록

> 실제 원격 PR 화면은 생성하지 않았습니다. 아래는 로컬 합성 PR 기록을 게시할 때 사용할
> 캡처 이름과 근거입니다.

| 캡처명 | 대상 | 근거 |
| --- | --- | --- |
| `R-A5-01-initial-pr.png` | 최초 PR 설명 | `403cf27` 시점, trace contract test 없음 |
| `R-A5-02-score-review.png` | semantic score Change Request | threshold `0.6`과 observed `0.93` 구분 코드 |
| `R-A5-03-trace-test-review.png` | trace contract Change Request | fixed-score 및 batch alignment test |
| `R-A5-04-response-commits.png` | test → fix 리뷰 반영 이력 | `502ddbc` → `6536ec4` |
| `R-A5-05-final-verification.png` | focused·전체 test | 24/24, 101/101 통과 |
| `R-A5-06-mainline-log.png` | 선형 Git 이력 | Review-04 → initial PR → review response |
