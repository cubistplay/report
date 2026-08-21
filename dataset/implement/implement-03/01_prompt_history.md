# 프롬프트 히스토리 — I-A3

> 실제 대화 export가 아니라, 이 PR을 만들기 위한 가상 개발 프롬프트 시나리오입니다.
> 구현과 테스트 결과를 기준으로 작성했습니다.

## Prompt 1 — 중복과 위험 확인

**Developer**

> pipeline에서 adapter registry를 그냥 dict로 들고 있고, algorithm 강제 선택할 때
> `AlgorithmPlan` 필드를 전부 손으로 복사하네. plan 필드 하나라도 추가되면 여기서
> 빠질 것 같은데, 구조적으로 정리할 수 있을까?

**Codex**

> adapter 등록과 조회는 `AlgorithmRegistry`로 감싸고, override는
> `dataclasses.replace()`로 algorithm과 reason만 바꾸면 됩니다. 우선 중복 등록,
> 빈 주입, plan 보존을 명세 테스트로 고정하겠습니다.

## Prompt 2 — 테스트 우선 계약

**Developer**

> 빈 registry를 넣었는데 기본값으로 다시 바뀌는 건 막아줘. adapter가 없을 때는
> output 폴더부터 만들고 실패하지 말고, 준비 전에 바로 오류를 내게 해줘.

**작업 단계 — Red 테스트**

- 같은 algorithm의 중복 등록 거부와 명시적 교체
- 빈 registry 주입 보존과 출력 directory 미생성
- algorithm override 뒤 나머지 plan 필드 보존

```bash
python3 -m unittest tests.test_pipeline_registry -q
# ImportError: 아직 AlgorithmRegistry가 없습니다.
```

Commit: `5adc292 test(implement-03): specify algorithm registry contracts`

## Prompt 3 — 기존 설정과 책임 경계

**Developer**

> 지금 registry를 dict로 넣는 호출도 있을 수 있으니 바로 깨지게 만들진 말아줘.
> 대신 adapter 이름이 key랑 다르면 등록 단계에서 바로 알 수 있게 했으면 좋겠어.

**Codex**

> pipeline은 `Mapping`도 받아 `AlgorithmRegistry.from_mapping()`으로 변환하겠습니다.
> key와 adapter 이름이 다르면 실행 시점까지 미루지 않고 변환 단계에서 오류를 냅니다.
> 기본 registry도 같은 등록 경로를 사용하게 하겠습니다.

## Prompt 4 — 동작 보존 확인

**Developer**

> 이건 리팩터링이니까 기본 route랑 `--algorithm` 같은 명시 override 결과가 이전이랑
> 똑같은지 전후 JSON으로 확인해줘. 전체 테스트도 마지막에 돌리고.

**작업 단계 — Green 검증**

```bash
python3 -m unittest tests.test_pipeline_registry -q
# Ran 5 tests — OK

python3 -m unittest tests.test_pipeline -q
# Ran 5 tests — OK

python3 -m unittest discover -s tests -q
# Ran 82 tests — OK
```

fact request의 기본 계획과 DPO override 계획을 변경 전 `1ba3260`와 비교했고,
`diff` 출력은 없었습니다.

Commit: `23fa548 refactor(implement-03): encapsulate algorithm registry`

## Prompt 5 — 리뷰 반영

**Developer**

> RAG adapter는 상속받은 기본 이름 때문에 registry key와 이름이 다시 어긋날 수 있대.
> 지금 명시적으로 이름을 준 동작이 깨지지 않게 테스트 하나 추가하고, 전체 테스트까지
> 다시 확인해줘.

**작업 단계 — 리뷰 반영 검증**

```bash
python3 -m unittest tests.test_pipeline_registry -q
# Ran 6 tests — OK

python3 -m unittest discover -s tests -q
# Ran 83 tests — OK
```

Commit: `9f0ceb9 test(implement-03): cover rag adapter registration`
