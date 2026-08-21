# R-A9 게시 전 체크리스트

> Synthetic GitHub artifact: true

- [x] 초기 PR은 a53d706의 변경만 설명합니다.
- [x] 전체 코드 범위는 허용된 brainwash/algorithms/memory_edit.py, tests/test_memory_edit_runtime.py입니다.
- [x] 최초 PR 변경량은 52줄이며 batch trigger API 한 가지 기능에 집중합니다.
- [x] 최초 PR 시점 전체 test 126건을 통과했습니다.
- [x] 리뷰는 실제 batch contract 5개를 다루며, 필요한 2개만 Change Request로 분류합니다.
- [x] Change Request는 cfdf1df test commit과 cd5e0db fix commit으로 별도 누적했습니다.
- [x] 리뷰 시작 후 rebase, squash, force push를 사용하지 않았습니다.
- [x] Review 1은 [Python mapping types 공식 문서](https://docs.python.org/3/library/stdtypes.html#mapping-types-dict)를 링크하고 duplicate dict key 예시로 cardinality 문제를 설명합니다.
- [x] duplicate input order와 triggered count를 regression test로 고정했습니다.
- [x] tests.test_memory_edit_runtime 15건과 전체 test 128건을 통과했습니다.
- [x] 프롬프트 히스토리는 실제 대화 export가 아닌 가상 리뷰 요청입니다.
- [x] 금지된 raw JSONL/세션/스레드 export 산출물을 만들지 않았습니다.
