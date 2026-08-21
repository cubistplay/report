# 프롬프트 히스토리 — I-A7

> 실제 대화 export가 아니라, 이 PR을 만들기 위한 가상 개발 프롬프트 시나리오입니다.
> 구현과 테스트 결과를 기준으로 작성했습니다.

## Prompt 1 — 조회 경로 책임 분리

**Developer**

> Update DB memory retrieval이 너무 한 메서드에 몰려 있어. exact, alias, phrase, FTS는
> 후보 찾는 것만 따로 빼고, 최종 선택이나 answer type 체크, log는 지금처럼 DB에서 한 번만
> 하게 정리해줘. 순서는 절대 바뀌면 안 돼.

**Codex**

> `RetrievalStrategy`와 route별 구현체를 두겠습니다. 각 구현체는 `RetrievalCandidate` 목록만
> 반환하고, `UpdateDb`가 첫 non-empty route를 semantic rerank·threshold·answer-type gate·audit
> log로 공통 처리하게 하겠습니다.

## Prompt 2 — 테스트 우선 계약

**Developer**

> 기본 순서가 exact → alias → pattern → FTS인지 확실히 보고, 빈 목록이면 검색 자체를 끄는
> 걸로 해. 나중에 전략 주입할 수도 있으니까 원본 prompt와 normalize된 값이 둘 다 넘어가는지도
> test로 남겨줘. 기존 DB 조회 테스트는 당연히 안 깨져야 해.

**작업 단계 — Red 테스트**

- default Strategy 구현체의 조회 순서
- 빈 Strategy 목록의 lookup disable 계약
- injected Strategy의 prompt/normalized prompt 입력
- abstract Strategy의 candidate lookup 계약

```bash
python3 -m unittest tests.test_update_db -q
# ImportError: AliasRetrievalStrategy 등이 아직 없습니다.
```

Commit: `b4b5c0a test(implement-07): specify memory retrieval strategies`

## Prompt 3 — 기존 결과 보존

**Developer**

> refactor 뒤에도 기존 exact match, semantic rerank, answer-type block, FTS 실패 시 빈 결과
> 처리까지 결과가 같아야 해. Update DB가 candidate를 최종 선택하는 경계도 유지하고 전체
> 테스트까지 돌려줘.

**작업 단계 — Green 검증**

```bash
python3 -m unittest tests.test_update_db -q
# Ran 8 tests — OK

python3 -m unittest tests.test_memory_ledger -q
# Ran 7 tests — OK

python3 -m unittest discover -s tests -q
# Ran 110 tests — OK
```

정책은 `UpdateDb`에 남기고 candidate 수집만 Strategy로 옮겼습니다. full suite는 기존 sqlite
connection `ResourceWarning` 2건을 출력했으나 실패는 없었습니다.

Commit: `5b7ce2f refactor(implement-07): extract memory retrieval strategies`
