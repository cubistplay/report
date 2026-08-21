# R-A1 최종 리뷰 보고서 — semantic matcher 설정 경계

## 1. 검토 배경

대상 PR은 semantic matcher의 환경 변수 해석을 `MatcherSettings`로 모으는 refactor였습니다.
초기 head는 기존 전체 테스트 90건을 통과했지만, 새 설정 객체와 process cache의 관계를
직접 검증하는 테스트는 없었습니다.

검토에서는 기능을 새로 추가했는지보다, 설정 객체가 표현하는 model/provider와 실제 cache
인스턴스가 같은 단위로 관리되는지를 우선 확인했습니다. 또한 기본값 중복, 환경 기반 테스트의
격리, 기존 fallback과 explicit override 우선순위를 함께 점검했습니다.

## 2. Commit 및 PR 경계

- base: `660416e93ac5553cde3924441a723b16a098aecc`
  `refactor(implement-05): separate preference export strategies`
- 최초 PR head: `defa2e31381179760634ede131b95e26f7222857`
  `refactor(review-01): centralize matcher settings`
- 리뷰 반영 테스트: `873402beae95a88e8502441884daea9d803e199b`
  `test(review-01): cover matcher setting isolation`
- 리뷰 반영 수정 및 최종 head: `5b8a43262e8cc2afdfc69c04f33ef153c75139af`
  `fix(review-01): isolate matcher setting cache`

리뷰가 시작된 뒤 최초 PR commit은 바꾸지 않았습니다. regression test와 수정 commit을
순서대로 누적해 `main`의 선형 이력을 유지했습니다.

## 3. 발견 사항과 반영 결과

| 심각도 | 발견 사항 | 영향 | 반영 |
| --- | --- | --- | --- |
| blocking | backend 이름만으로 cache key를 구성 | model/provider 변경 뒤 이전 matcher 재사용 | `(backend, setting)` key로 수정 |
| important | 새 환경·cache 계약의 테스트 부재 | host 환경과 실행 순서에 따른 회귀를 발견하기 어려움 | `patch.dict` 기반 regression test 3건 추가 |
| important | 기본 embedding model 문자열 중복 | 두 진입점의 default가 달라질 위험 | `DEFAULT_EMBEDDING_MODEL` 단일 상수로 통합 |
| question | unknown matcher 값의 fallback | 설정 오류 정책이 retrieval에 미치는 영향 | 기존의 `None` fallback을 의도적으로 유지 |
| question | explicit matcher 우선순위 | injected matcher와 environment 설정의 충돌 | `_ACTIVE` 선확인 계약 유지 확인 |

## 4. 리뷰 품질과 협업 방식

각 지적은 변경된 `MatcherSettings.resolve()`와 `_ENV_CACHE`의 실제 호출 경로를 근거로
작성했습니다. 특히 첫 번째 지적은 `model/a → model/b`, `claude → openai`의 연속 호출을
제시해 재현 조건과 영향을 분명히 했습니다.

테스트 지적에서는 [Python `unittest.mock.patch.dict` 공식 문서](https://docs.python.org/3/library/unittest.mock.html#patch.dict)를
근거로, `with patch.dict(...):` 블록에서만 `os.environ`을 바꾸고 블록 종료 뒤 기존 환경을
자동 복구하는 예시를 함께 제시했습니다. 단순히 “테스트를 추가해 달라”는 요청이 아니라 외부
환경 없이 factory seam과 cache 계약을 검증할 수 있는 방향을 제안했습니다.

unknown 값 fallback과 explicit matcher 우선순위는 결함으로 단정하지 않고, 기존의 opt-in
정책과 test-double 주입 경계를 확인하는 질문으로 다뤘습니다. 이를 통해 불필요한 Change
Request를 만들지 않으면서도 변경의 호환성 범위를 검증했습니다.

## 5. 검증

최초 PR head에서는 기존 전체 suite를 실행했습니다.

```bash
python3 -m unittest discover -s tests -q
# Ran 90 tests — OK
```

리뷰 반영 후에는 focused test와 전체 suite를 다시 실행했습니다.

```bash
python3 -m unittest tests.test_semantic -q
# Ran 15 tests — OK

python3 -m unittest discover -s tests -q
# Ran 93 tests — OK
```

전체 suite는 기존 sqlite connection에 대한 `ResourceWarning` 2건을 출력했으나, test 실패는
없었습니다. 이번 변경과 무관한 기존 경고이므로 별도 범위로 분리했습니다.

## 6. 변경 범위

| 구간 | 파일 | 추가 | 삭제 | 합계 |
| --- | --- | ---: | ---: | ---: |
| 최초 PR | `brainwash/semantic.py` | 48 | 19 | 67 |
| 리뷰 반영 | `brainwash/semantic.py`, `tests/test_semantic.py` | 81 | 10 | 91 |

초기 변경량 106줄은 Review PR의 일반적인 50~200줄 범위 안이며, 설정 추출이라는 하나의
검토 단위를 충분히 보여 줍니다. 리뷰 반영은 발견한 cache·default·test 격리 문제만
해결했고 허용된 source/test 파일 밖으로 범위를 넓히지 않았습니다.

## Black 포맷 검증

각 코드 커밋 직전에 Black 26.5.1을 적용했습니다. 최종 변경 파일은 `black --check`를 통과했고, 재작성 전후 변경 Python 파일의 AST가 동일함을 확인했습니다. `#` 주석과 inline comment는 코드에서 제거했으며, 새 docstring은 추가하지 않았습니다.

## Black 포맷 검증

각 코드 커밋 직전에 Black 26.5.1을 적용했습니다. 최종 변경 파일은 `black --check`를 통과했고, 재작성 전후 변경 Python 파일의 AST가 동일함을 확인했습니다. `#` 주석과 inline comment는 코드에서 제거했으며, 새 docstring은 추가하지 않았습니다.

## Black 포맷 검증

각 코드 커밋 직전에 Black 26.5.1을 적용했습니다. 최종 변경 파일은 `black --check`를 통과했고, 재작성 전후 변경 Python 파일의 AST가 동일함을 확인했습니다. `#` 주석과 inline comment는 코드에서 제거했으며, 새 docstring은 추가하지 않았습니다.

