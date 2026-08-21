# I-A7 개발 활동 보고서 — memory retrieval Strategy 분리

## 1. 현황 및 이슈

`UpdateDb.retrieve_memory_edit()`는 exact, alias, pattern, FTS의 SQL 조회, route별 score 계산,
semantic rerank, answer-type validation, threshold 판정, audit log 기록을 한 제어 흐름에서 처리했습니다.
메서드를 읽을 때 candidate를 “어떻게 찾는지”와 “왜 수락하거나 거절하는지”를 동시에 추적해야 했습니다.

가독성 측면에서는 네 route의 SQL 분기가 공통 selection 흐름을 가려 각 route의 차이를 파악하기
어려웠습니다. 유지보수성 측면에서는 조회 route 하나를 추가하거나 FTS 예외 정책을 바꿀 때 threshold,
answer-type gate, log schema까지 실수로 변경할 가능성이 있었습니다. 조회 확장과 최종 안전 정책을
독립적으로 검토할 수 있는 구조가 필요했습니다.

## 2. Commit 및 PR 경계

- base: `main` / `2749f66ecb6830e1d77228e8dc34d75718bd8d3b`
- Red 테스트: `b4b5c0ac68ce4299290622f9501b544bd2a394ec`
  `test(implement-07): specify memory retrieval strategies`
- 최초 PR 및 최종 head: `5b7ce2f912960aeac1a2e65136e4b9c298931af5`
  `refactor(implement-07): extract memory retrieval strategies`
- 최종 `main`: `5b7ce2f912960aeac1a2e65136e4b9c298931af5`

최초 head에서 Strategy와 final selection의 경계, default/empty/custom strategy 설정 의미,
FTS fallback 및 audit contract 보존을 검토했습니다. 코드 결함은 발견되지 않아 Change Request나
후속 commit은 만들지 않았습니다.

## 3. 활동 내용

구현 의도는 route별 candidate 수집만 Strategy로 추출하고, semantic rerank와 최종 수락·거절 정책은
하나의 공통 경로에 유지하는 것이었습니다. 먼저 존재하지 않는 retrieval Strategy를 사용하는 Red
테스트를 작성해 default route 순서, 빈 Strategy 목록의 lookup disable, injected Strategy의
raw/normalized query, abstract contract를 명세로 고정했습니다.

`RetrievalStrategy`와 exact·alias·pattern·FTS 구현체는 각자의 SQL prefilter와 initial score만
반환합니다. route·row·score는 `RetrievalCandidate`로 전달되며, `UpdateDb.retrieve_memory_edit()`는
first non-empty route의 candidate를 semantic rerank한 뒤 priority, threshold, answer type으로 최종
선택합니다. audit trace와 retrieval log도 selection 이후 한 곳에서 기록합니다.

또한 constructor 설정에서 `None`은 default order, 빈 tuple은 explicit disable로 구분했습니다.
이 명시적 계약은 설정값의 의미를 호출부만 읽어도 이해할 수 있게 하고 custom Strategy가 공통 safety
gate를 우회하지 못하게 합니다. 구현 후 다음 검증을 완료했습니다.

```bash
python3 -m unittest tests.test_update_db -q
# Ran 8 tests — OK

python3 -m unittest tests.test_memory_ledger -q
# Ran 7 tests — OK

python3 -m unittest discover -s tests -q
# Ran 110 tests — OK
```

전체 suite는 기존 sqlite connection `ResourceWarning` 2건을 출력했으나 test 실패는 없었습니다.

## 4. 기대 효과

새 retrieval route는 Strategy 구현과 route 단위 테스트로 범위를 제한할 수 있습니다. 반대로 threshold,
answer-type, semantic rerank, audit 형식을 변경할 때는 `UpdateDb`의 공통 selection 경로만 검토하면 됩니다.
route별 SQL과 공통 정책의 변경 이유가 분리되어 코드 탐색 시간과 회귀 테스트 범위가 줄어듭니다.

리뷰 과정에서는 retrieval을 “candidate collection”과 “final selection”으로 구분해 인식을 맞췄습니다.
팀원은 새로운 backend를 논의할 때 SQL·score 생성과 수락 정책을 별도 결정으로 다룰 수 있고,
`RetrievalCandidate`를 공통 교환 형식으로 사용할 수 있습니다. 그 결과 route 추가 자체가 audit와 safety
정책 변경으로 확대되는 것을 방지하고, 리뷰에서도 공통 계약이 유지되는지 빠르게 확인할 수 있습니다.

## 5. 변경 규모와 범위

| 항목 | 결과 |
| --- | ---: |
| 추가 | 388줄 |
| 삭제 | 153줄 |
| 합계 | 541줄 |
| 파일 | 2개 |
| 허용 목록 외 변경 | 없음 |

변경 파일은 `brainwash/memory/update_db.py`, `tests/test_update_db.py`입니다. 300줄을 넘었지만
네 SQL route의 중복 제어 흐름을 Strategy로 옮기고, Candidate Value Object·constructor injection·
rerank delegation·네 계약 test를 함께 완료한 하나의 coherent refactor입니다.

## 6. 리뷰 결과

리뷰에서는 candidate collection과 final selection의 책임 경계, default/empty/custom Strategy의 설정
의미, FTS fallback과 audit log 공통 경로 보존을 확인했습니다. 세 스레드의 설계 검토와 Update DB·
Memory Ledger·전체 suite 검증을 근거로 승인했습니다.

## Black 포맷 검증

각 코드 커밋 직전에 Black 26.5.1을 적용했습니다. 변경된 Python 파일의 comment token과 module/class/function docstring은 제거했으며, SQL·script template·test fixture 같은 실행용 multiline string 값은 보존했습니다. 전체 51개 커밋의 변경 Python blob 57개가 `black --check`를 통과했고, 원본과 재작성본의 실행 AST도 동일합니다.
