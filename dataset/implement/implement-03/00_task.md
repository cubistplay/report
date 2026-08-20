# Implement-03 — algorithm registry와 계획 override 책임 정리

## 개발 요청

`default_registry()`가 단순 dictionary를 반환하고, `BrainwashPipeline.plan()`은
algorithm override 때 `AlgorithmPlan`의 모든 필드를 손으로 다시 복사하고 있습니다.
등록·교체·누락 처리의 계약을 한 객체로 모으고, 계획의 나머지 값은 그대로 보존하는
리팩터링을 진행해 주세요.

## 완료 조건

- `AlgorithmRegistry`가 adapter 등록, 명시적 교체, 조회, 누락 오류를 책임집니다.
- 기존 dictionary 주입 API는 mapping 변환으로 계속 받을 수 있습니다.
- 빈 registry를 주입했을 때 기본 registry로 바뀌지 않습니다.
- algorithm override는 algorithm과 reason만 바꾸고, 나머지 `AlgorithmPlan` 필드는
  라우터 결과와 같아야 합니다.
- adapter가 없으면 출력 directory를 만들기 전에 실패해야 합니다.
- 기본 경로와 명시 override의 계획 출력은 변경 전과 같아야 합니다.

## 구현 제약

- 테스트를 먼저 작성하고 `test → refactor` 두 commit으로 남깁니다.
- 변경은 `brainwash/algorithms/registry.py`, `brainwash/pipeline.py`,
  `tests/test_pipeline_registry.py`에 한정합니다.
- adapter 구현, router 규칙, CLI, artifact 형식은 바꾸지 않습니다.

## 작업 단위

이번 작업은 Registry 패턴으로 adapter 선택 책임을 명시하고, `dataclasses.replace()`로
불변 `AlgorithmPlan`을 안전하게 갱신하는 refactor입니다. 새로운 plan 필드가 추가되어도
override 코드가 그 필드를 잊어버릴 위험을 없앱니다. RAG adapter는 registry key와 같은
이름으로 등록해 기존 상속 기본값의 identity 불일치도 바로잡습니다.
