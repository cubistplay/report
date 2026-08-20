# I-A2 개발 활동 보고서 — benchmark request Factory 분리

## 1. 배경

CounterFact, KnowEdit, MQuAKE, RippleEdits loader는 서로 다른 raw schema를
읽지만, 마지막에는 같은 `CorrectionRequest`를 만들었습니다. 이 과정에서
`subject`/`relation` metadata, answer type, paraphrase/locality prompt를 조립하는
규칙이 네 곳에 반복되어 있었습니다.

이번 변경은 입력 schema를 통합하지 않습니다. 각 loader의 해석 책임은 유지하고,
해석된 값으로 공통 request를 만드는 책임만 Factory로 옮겼습니다.

## 2. Commit 및 PR 경계

- base: `main` / `dcd46fd68822ac3041daa94afe53c80043e2a903`
- Red 테스트: `ae1de36f9993076bcefd278a09063740e22c8419`
  `test(implement-02): specify benchmark request factory`
- 최초 PR 및 최종 head: `29bb11e491889c22f93e01e1f9a344669d860aae`
  `refactor(implement-02): centralize benchmark request construction`
- 최종 `main`: `29bb11e491889c22f93e01e1f9a344669d860aae`

최초 head에서 네 가지 설계·동작 경계를 검토했습니다. 동작 차이나 코드 결함은 발견되지
않았으므로 Change Request나 후속 commit을 추가하지 않았습니다.

## 3. TDD 및 동작 보존 검증

Red 테스트는 아직 없는 `BenchmarkRequestFactory`와 `BenchmarkRequestSpec` import에서
실패했습니다. 새 Factory의 metadata 계약, caller metadata 불변성, adapter별 prompt
보존을 먼저 명세로 고정한 것입니다.

구현 후 아래 검증을 완료했습니다.

```bash
python3 -m unittest tests.test_benchmark_adapters_factory -q
# Ran 3 tests — OK

python3 -m unittest tests.test_benchmark_adapters -q
# Ran 6 tests — OK

python3 -m unittest discover -s tests -q
# Ran 77 tests — OK
```

또한 CounterFact, KnowEdit, MQuAKE, RippleEdits의 대표 입력을 변경 전 commit과
현재 commit에서 JSON으로 출력해 비교했습니다. `diff` 결과가 없어 변환 결과가
보존됐음을 확인했습니다.

## 4. 구조 개선

`BenchmarkRequestSpec`은 각 loader가 raw row에서 해석한 공통 값을 담습니다.
`BenchmarkRequestFactory`는 spec을 받아 `CorrectionRequest`와 공통 metadata를
만듭니다. 이는 Factory 패턴을 실제로 적용한 구조입니다.

Factory는 `extra_metadata`를 복사하고 `subject`, `relation`을 최종적으로 설정합니다.
따라서 caller mapping을 수정하지 않으면서도 공통 식별 정보의 계약을 보장합니다.
answer type resolver도 주입할 수 있어 unit test에서 matcher 상태와 분리해 확인할 수
있습니다.

## 5. 변경 규모와 범위

| 항목 | 결과 |
| --- | ---: |
| 추가 | 186줄 |
| 삭제 | 43줄 |
| 합계 | 229줄 |
| 파일 | 2개 |
| 허용 목록 외 변경 | 없음 |

변경 파일은 `brainwash/benchmarks.py`와
`tests/test_benchmark_adapters_factory.py`입니다. 생성 파일이나 formatting만 한
변경은 포함하지 않았습니다. 229줄 안에서 Factory 도입, 중복 제거, regression
테스트, 동작 보존 증명을 하나의 검토 가능한 단위로 완결했습니다.

## 6. 리뷰 결과

리뷰에서는 metadata 우선순위, matcher 조회 시점, Factory와 loader의 입력 해석
경계를 검토했습니다. Factory 단위 테스트, 기존 adapter 테스트, JSON 출력 비교로
각 결정을 확인했고 추가 코드 변경 없이 승인되었습니다.
