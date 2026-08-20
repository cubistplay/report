# I-A3 개발 활동 보고서 — algorithm registry 캡슐화

## 1. 배경

기존 pipeline은 algorithm adapter를 일반 dictionary로 보관하고, 없을 때의 오류를
직접 처리했습니다. 또한 사용자가 algorithm을 명시하면 router가 만든 `AlgorithmPlan`의
모든 필드를 수동으로 복사했습니다. 이 방식은 registry 설정의 의미가 분산되고,
`AlgorithmPlan` 필드가 늘어날 때 override에서 값을 빠뜨릴 위험이 있었습니다.

이번 변경에서는 adapter 등록 정책과 조회 정책을 `AlgorithmRegistry`로 분리했습니다.
사용자 선택 override는 변경이 필요한 두 필드만 갱신하도록 정리했습니다. router 규칙과
adapter별 artifact 생성 방식은 유지하되, RAG adapter가 상속받은 기본 이름으로 잘못
기록될 수 있던 등록 구성은 `RAG_CONTEXT_PATCH` 이름을 명시해 바로잡았습니다.

## 2. Commit 및 PR 경계

- base: `main` / `29bb11e491889c22f93e01e1f9a344669d860aae`
- Red 테스트: `5655dde4cdca26cf542bb74b96c3cd1e4410fd7d`
  `test(implement-03): specify algorithm registry contracts`
- 최초 PR head: `3cb00d9d558281c241fdd0ed387df5c71fe86bc1`
  `refactor(implement-03): encapsulate algorithm registry`
- 리뷰 반영: `3c11c0234491b57001cd620ba284071c4ad4957b`
  `test(implement-03): cover rag adapter registration`
- 최종 `main`: `3c11c0234491b57001cd620ba284071c4ad4957b`

최초 head에서 다섯 가지 설계·동작 경계를 검토했습니다. RAG adapter 이름이 상속
기본값으로 다시 어긋나지 않도록 회귀 테스트가 필요하다는 Change Request 1건을 받고,
일반 push 가능한 새 test commit으로 반영했습니다.

## 3. TDD 및 동작 보존 검증

Red 테스트는 아직 없는 `AlgorithmRegistry` import에서 실패했습니다. 이후 중복 등록
거부, 명시적 교체, 빈 registry 주입, output directory 생성 전 실패, override의 전체
plan 보존을 명세로 확인했습니다.

구현 후 아래 검증을 완료했습니다.

```bash
python3 -m unittest tests.test_pipeline_registry -q
# Ran 5 tests — OK

python3 -m unittest tests.test_pipeline -q
# Ran 5 tests — OK

python3 -m unittest discover -s tests -q
# Ran 82 tests — OK
```

변경 전 commit과 최초 PR head에서 동일한 fact request의 기본 계획 및 DPO override
계획을 JSON으로 출력해 비교했습니다. `diff` 결과가 없어 해당 계획 결과가 보존됐음을
확인했습니다. RAG adapter 이름은 의도적으로 수정한 범위이므로 동등성 비교 대상에서
제외했고, 리뷰 반영 후에는 RAG identity 회귀 테스트로 별도로 고정했습니다.

리뷰 반영 후에는 registry/pipeline 테스트 6건, 기존 pipeline 테스트 5건, 전체 테스트
83건을 다시 통과했습니다.

## 4. 구조 개선

`AlgorithmRegistry`는 `Mapping` 인터페이스와 호환되는 Registry 패턴 구현입니다.
adapter 추가는 `register()`로만 수행하므로 중복 등록은 기본적으로 거부되고, 교체는
`replace=True`를 명시해야 합니다. 이전 dictionary 주입은 `from_mapping()`으로 받아
호환성을 유지하면서 key와 adapter 이름의 불일치를 구성 단계에서 확인합니다.

`BrainwashPipeline`은 registry를 선택하고 route를 실행하는 역할만 맡습니다. adapter
필수 조회는 `require()`로 위임했고, adapter가 없는 경우 output directory 생성 전에
실패하도록 순서를 정리했습니다. 사용자 선택 override에는 `dataclasses.replace()`를
사용해 algorithm과 reason만 갱신하므로, 추가되는 `AlgorithmPlan` 필드가 자동으로
보존됩니다.

## 5. 변경 규모와 범위

| 항목 | 결과 |
| --- | ---: |
| 추가 | 203줄 |
| 삭제 | 39줄 |
| 합계 | 242줄 |
| 파일 | 3개 |
| 허용 목록 외 변경 | 없음 |

변경 파일은 `brainwash/algorithms/registry.py`, `brainwash/pipeline.py`,
`tests/test_pipeline_registry.py`입니다. 242줄 안에서 Registry 패턴 도입,
수동 plan 복사 제거, RAG identity regression 테스트, 변경 전후 출력 비교를 하나의
검토 단위로 완결했습니다.

## 6. 리뷰 결과

리뷰에서는 빈 registry와 기본값의 구분, `replace()`가 보존하는 plan 필드의 범위,
중복 등록 정책, mapping 이름 계약, 읽기·변경 API 경계를 확인했습니다. RAG identity
회귀 테스트를 Change Request로 추가한 뒤 전체 테스트를 다시 통과했고 최종 승인되었습니다.
