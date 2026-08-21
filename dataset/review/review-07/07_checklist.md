# R-A7 게시 전 체크리스트

> Synthetic GitHub artifact: true

- [x] 초기 PR은 5be9d0c의 변경만 설명합니다.
- [x] 전체 코드 범위는 허용된 brainwash/memory/update_db.py, tests/test_update_db.py입니다.
- [x] 최초 PR 변경량은 61줄이며 promotion candidate read model 한 가지 기능에 집중합니다.
- [x] 최초 PR 시점 전체 test 122건을 통과했습니다.
- [x] 리뷰는 실제 계약 5개를 다루며, 필요한 2개만 Change Request로 분류합니다.
- [x] Change Request는 b9db88c test commit과 9aae539 fix commit으로 별도 누적했습니다.
- [x] 리뷰 시작 후 rebase, squash, force push를 사용하지 않았습니다.
- [x] Review 1은 [SQLite aggregate 함수 공식 문서](https://www.sqlite.org/lang_aggfunc.html)를 링크하고 GROUP_CONCAT(id) 예시로 순서 문제를 설명합니다.
- [x] stable record order와 normalized target contract를 regression test로 고정했습니다.
- [x] tests.test_update_db 12건과 전체 test 124건을 통과했습니다.
- [x] 프롬프트 히스토리는 실제 대화 export가 아닌 가상 리뷰 요청입니다.
- [x] 금지된 raw JSONL/세션/스레드 export 산출물을 만들지 않았습니다.
