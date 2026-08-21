# 프롬프트 히스토리 — I-A9

> 실제 대화 export가 아니라, 이 PR을 만들기 위한 가상 개발 프롬프트 시나리오입니다.
> 구현과 테스트 결과를 기준으로 작성했습니다.

## Prompt 1 — trigger candidate와 safety gate 분리

**Developer**

> MemoryEditStore trigger가 exact 확인, token overlap, subject guard, threshold, answer type을
> 다 같이 처리하고 있어. exact/paraphrase와 subject-aware overlap은 후보를 찾는 쪽으로 빼고,
> threshold랑 answer type block은 지금 store가 공통으로 하게 정리해줘.

**Codex**

> `MemoryMatchStrategy`가 edit별 match evidence를 반환하고, `MemoryMatchPolicy`가 score가 가장
> 높은 candidate를 고르도록 하겠습니다. `MemoryEditStore`는 기존 threshold·answer-type gate와
> verifier를 유지해 모든 Strategy에 동일한 safety policy를 적용하겠습니다.

## Prompt 2 — 테스트 우선 계약

**Developer**

> exact가 overlap보다 먼저인 건 꼭 유지하고, Strategy를 아예 비우면 trigger가 꺼지는 걸로 하자.
> 나중에 custom matcher를 붙일 수도 있으니까 query랑 edit을 받는지도 test로 남겨줘. 기존
> strict fallback이나 city/country block은 깨지면 안 돼.

**작업 단계 — Red 테스트**

- default exact → subject overlap Strategy 순서
- 빈 Strategy 목록의 trigger disable
- injected Strategy의 query/edit 입력
- abstract Strategy의 match contract

```bash
python3 -m unittest tests.test_memory_edit_runtime -q
# ImportError: ExactMemoryMatchStrategy 등이 아직 없습니다.
```

Commit: `8774fb1 test(implement-09): specify memory match strategies`

## Prompt 3 — verifier 흐름 보존

**Developer**

> candidate 찾는 구조만 바꾸는 거니까 trigger reason, threshold block, answer-type block,
> verifier의 context_verified/strict_fallback 결과는 이전과 같아야 해. Update DB랑 전체 test도
> 같이 확인해줘.

**작업 단계 — Green 검증**

```bash
python3 -m unittest tests.test_memory_edit_runtime -q
# Ran 13 tests — OK

python3 -m unittest tests.test_update_db -q
# Ran 8 tests — OK

python3 -m unittest discover -s tests -q
# Ran 117 tests — OK
```

exact/paraphrase Strategy가 먼저 candidate를 만들고, overlap Strategy는 subject token이 있을 때만
fallback으로 candidate를 만듭니다. full suite는 기존 sqlite connection `ResourceWarning` 2건을
출력했으나 실패는 없었습니다.

Commit: `365a6a6 refactor(implement-09): separate memory match strategies`
