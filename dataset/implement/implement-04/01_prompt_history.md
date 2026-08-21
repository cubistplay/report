# 프롬프트 히스토리 — I-A4

> 실제 대화 export가 아니라, 이 PR을 만들기 위한 가상 개발 프롬프트 시나리오입니다.
> 구현과 테스트 결과를 기준으로 작성했습니다.

## Prompt 1 — loop 책임 분리

**Developer**

> dynamic evidence executor가 너무 길어. action 기록, dependency 중복 제거, ask랑
> answer 처리, 실패 응답까지 한 loop에 다 있어서 하나 건드리면 다른 경로가 불안해.
> 흐름은 그대로 두고 구조만 좀 나눌 수 있을까?

**Codex**

> action 종류별로 실행 Strategy를 두고, 반복 중 바뀌는 값은 `DynamicEvidenceState`로
> 모을 수 있습니다. 우선 stage 선택, dependency 중복 제거, invalid action 기록을
> 테스트로 고정하겠습니다.

## Prompt 2 — 테스트 우선 계약

**Developer**

> `give_up` 같은 잘못된 action도 planner가 뭘 냈는지 plan에 남아야 해. 그리고 같은
> memory candidate를 두 번 봐도 dependency가 중복되면 안 돼.

**작업 단계 — Red 테스트**

- `answer`·`ask`·기타 action의 stage 선택
- action 기록과 memory dependency 중복 제거
- invalid action의 기록 후 실패

```bash
python3 -m unittest tests.test_decomposition_stages -q
# ImportError: 아직 DynamicActionStageRegistry 등이 없습니다.
```

```bash
python3 -m unittest tests.test_decomposition -q
# Ran 22 tests — OK
```

Commit: `b0e7fe8 test(implement-04): specify dynamic evidence stages`

## Prompt 3 — 행동 보존

**Developer**

> 이건 루프 구조만 바꾸는 거니까 scripted planner로 memory step 하나 밟고 answer하는
> 결과가 전이랑 완전히 같은지 JSON으로 비교해줘. 기존 dynamic 테스트도 다 돌리고.

**작업 단계 — Green 검증**

```bash
python3 -m unittest tests.test_decomposition_stages -q
# Ran 3 tests — OK

python3 -m unittest tests.test_decomposition -q
# Ran 22 tests — OK

python3 -m unittest discover -s tests -q
# Ran 86 tests — OK
```

동일한 scripted planner와 memory를 Implement-03 head와 현재 commit에서 실행해 JSON을
비교했고 `diff` 출력은 없었습니다.

Commit: `067cbce refactor(implement-04): decompose dynamic evidence actions`
