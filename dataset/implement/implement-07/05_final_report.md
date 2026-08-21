# I-A7 개발 활동 보고서 — memory retrieval Strategy 분리

## 1. 배경

`UpdateDb.retrieve_memory_edit()`는 exact, alias, pattern, FTS의 SQL 조회와 score 계산, semantic
rerank, answer-type validation, threshold 선택, retrieval audit log를 모두 수행했습니다. 조회 route를
추가하거나 SQL을 변경할 때 final selection 정책까지 함께 검토해야 했습니다.

이번 변경은 candidate 수집을 `RetrievalStrategy`로 분리하고, route·row·initial score를
`RetrievalCandidate` Value Object로 전달하게 했습니다. semantic rerank와 최종 수락/거절은
`UpdateDb`가 계속 단일 책임으로 처리합니다.

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

## 3. TDD 및 동작 보존 검증

Red 테스트는 존재하지 않는 retrieval Strategy import에서 실패했습니다. default route order, 빈
Strategy 목록의 lookup disable, injected Strategy의 raw/normalized query, abstract contract를 먼저
명세로 고정했습니다.

구현 후 아래 검증을 완료했습니다.

```bash
python3 -m unittest tests.test_update_db -q
# Ran 8 tests — OK

python3 -m unittest tests.test_memory_ledger -q
# Ran 7 tests — OK

python3 -m unittest discover -s tests -q
# Ran 110 tests — OK
```

전체 suite는 기존 sqlite connection `ResourceWarning` 2건을 출력했으나 test 실패는 없었습니다.

## 4. 구조 개선

`RetrievalStrategy`는 DB connection, raw prompt, normalized prompt를 받아 한 route의
`RetrievalCandidate` 목록을 반환하는 Strategy 인터페이스입니다. exact, alias, pattern, FTS 구현체는
각 SQL prefilter와 초기 score만 소유합니다.

`UpdateDb`는 first non-empty route의 후보를 semantic rerank한 뒤 priority, threshold, answer-type으로
최종 선택합니다. audit trace와 retrieval log도 이 공통 경로에서 한 번만 기록합니다. 따라서 route를
추가해도 final policy와 log schema가 route별 분기로 복제되지 않습니다.

`None` Strategy 설정은 default order를, 빈 tuple은 explicit disable을 뜻합니다. custom Strategy는
raw와 normalized query 모두를 받으므로 SQL key lookup 외의 확장도 가능하지만, final selection은 여전히
공통 contract를 따릅니다.

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
