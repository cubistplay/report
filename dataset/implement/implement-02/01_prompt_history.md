# 프롬프트 히스토리 — I-A2

> 실제 대화 export가 아니라, 이 PR을 만들기 위한 가상 개발 프롬프트 시나리오입니다.
> 구현과 테스트 결과를 기준으로 작성했습니다.

## Prompt 1 — 중복 확인

**Developer**

> benchmark loader 네 개를 보니까 마지막에 request 만들고 metadata 붙이는 코드가
> 거의 반복돼. 입력 schema는 각각 달라서 억지로 하나로 합치고 싶진 않은데, 공통된
> 규칙만 안전하게 뺄 방법이 있을까?

**Codex**

> 각 loader는 row를 해석해 normalized 값으로 만드는 역할만 두고,
> `CorrectionRequest` 생성과 metadata 조립은 `BenchmarkRequestFactory`로 분리할 수
> 있습니다. 먼저 Factory의 metadata 계약과 adapter 결과 보존 테스트를 작성하겠습니다.

## Prompt 2 — 테스트 우선 계약

**Developer**

> Factory가 answer type까지 넣는다면 matcher가 없을 때도 기존처럼 동작해야 하고,
> caller metadata를 건드리면 안 돼. KnowEdit랑 RippleEdits처럼 서로 다른 입력도
> 기존 prompt와 locality prompt가 그대로 나오는지 같이 확인해줘.

**작업 단계 — Red 테스트**

- 공통 metadata와 answer type 주입
- caller metadata의 불변성
- KnowEdit/RippleEdits의 adapter별 prompt 유지

```bash
python3 -m unittest tests.test_benchmark_adapters_factory -q
# ImportError: 아직 BenchmarkRequestFactory가 없습니다.
```

```bash
python3 -m unittest tests.test_benchmark_adapters -q
# Ran 6 tests — OK
```

Commit: `5f3a5fb test(implement-02): specify benchmark request factory`

## Prompt 3 — 책임 경계

**Developer**

> Factory가 loader의 입력 해석까지 먹어버리면 나중에 더 복잡해질 것 같아.
> CounterFact나 MQuAKE가 자기 row를 읽는 방식은 그대로 두고, 완성된 값으로 request를
> 만드는 부분만 책임지게 해줘.

**Codex**

> `BenchmarkRequestSpec`은 loader가 해석한 정규화 값을 담고,
> `BenchmarkRequestFactory`는 그 spec을 `CorrectionRequest`로 바꾸겠습니다.
> MQuAKE의 `mquake_case_id`처럼 adapter 고유 metadata는 `extra_metadata`로 유지합니다.

## Prompt 4 — 동작 보존 확인

**Developer**

> 리팩터링이면 결과가 달라지면 안 돼. CounterFact, KnowEdit, MQuAKE, RippleEdits를
> 같은 입력으로 전후 비교해서 JSON 차이 없는 것도 확인하고 PR에 적어줘.

**작업 단계 — Green 검증**

```bash
python3 -m unittest tests.test_benchmark_adapters_factory -q
# Ran 3 tests — OK

python3 -m unittest tests.test_benchmark_adapters -q
# Ran 6 tests — OK

python3 -m unittest discover -s tests -q
# Ran 77 tests — OK
```

CounterFact, KnowEdit, MQuAKE, RippleEdits의 동일 입력 JSON을 변경 전 `ebec589`와
비교했고 `diff` 출력은 없었습니다.

Commit: `1ba3260 refactor(implement-02): centralize benchmark request construction`
