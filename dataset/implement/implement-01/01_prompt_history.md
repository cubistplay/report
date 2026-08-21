# 프롬프트 히스토리 — I-A1

> 실제 대화 export가 아니라, 이 PR을 만들기 위한 가상 개발 프롬프트 시나리오입니다.
> 구현과 테스트 결과를 기준으로 작성했습니다.

## Prompt 1 — locality 문제 확인

**Developer**

> locality가 수정 후 출력에 target text가 없는지만 보고 있어. 이러면 원래 답이
> 바뀌어도 통과할 수 있잖아. baseline과 비교하게 바꾸고 싶은데, Harness에 조건문만
> 더하는 방식은 피하고 현재 책임이 어디에 섞여 있는지 먼저 봐줘.

**Codex**

> 현재는 사례별 평가 규칙, evaluator 설정, 점수와 결과 조립이 한 흐름에 섞여
> 있습니다. baseline 보존은 expected-answer와 다른 규칙이므로, Strategy와
> registry·결과 집계 경계를 테스트로 먼저 고정하겠습니다.

## Prompt 2 — 테스트 우선 계약

**Developer**

> 좋아. locality 통과/실패만 보지 말고 baseline이 없는 경우, 사용자 지정
> evaluator, 잘못된 registry가 model을 아예 실행하지 않는 것까지 테스트로 잡아줘.
> 기존 `test_pipeline.py`도 새 locality 계약에 맞게만 바꿔.

**작업 단계 — Red 테스트**

- baseline 보존과 regression 근거 기록
- baseline이 없을 때의 `unscored` 결과
- reliability/generalization 점수 분리
- registry 검증이 generator 호출보다 먼저 실행되는지
- 사용자 지정 evaluator의 입력과 `behavior` kind 생성

```bash
python3 -m unittest tests.test_pipeline_eval_harness -q
# ImportError: 아직 CaseVerdict가 없습니다.
```

Commit: `c704ecc test(implement-01): specify evaluator registry contracts`

## Prompt 3 — evaluator 확장 경계

**Developer**

> 사용자 지정 evaluator가 나중에 필요한 입력을 받으려면 매개변수가 계속 늘어날 수
> 있잖아. 그 부분도 깔끔하게 잡아줘. 그리고 빈 registry를 넘겼을 때 기본값이
> 조용히 붙으면 안 돼.

**Codex**

> `EvaluationContext`에 사례, 수정 후 출력, baseline 출력을 묶겠습니다.
> `EvaluatorRegistry`는 기본값 선택과 coverage 검증을 담당하게 하겠습니다.
> `None`만 기본 registry를 뜻하고, mapping은 호출자가 넘긴 설정 그대로 검증합니다.

## Prompt 4 — 결과 집계 책임

**Developer**

> Harness가 실행도 하고 실패 결과 dict도 만들면 다시 커질 것 같아. 실패 사례,
> `unscored`, kind별 점수를 정리하는 쪽도 분리해줘. PR에는 왜 이 구조인지와
> 전후 흐름도 넣고.

**작업 단계 — Green 검증**

```bash
python3 -m unittest tests.test_pipeline_eval_harness -q
# Ran 6 tests — OK

python3 -m unittest tests.test_pipeline -q
# Ran 5 tests — OK

python3 -m unittest discover -s tests -q
# Ran 74 tests — OK
```

Commit: `78c27ec feat(implement-01): preserve locality with evaluation strategies`
