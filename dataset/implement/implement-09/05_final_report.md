# I-A9 개발 활동 보고서 — memory match Strategy 분리

## 1. 현황 및 이슈

`MemoryEditStore.evaluate_trigger()`는 exact/paraphrase 비교, subject-aware token overlap, best score 선택,
threshold 판정, answer-type validation을 한 loop에서 처리했습니다. 후보를 찾는 규칙과 correction을
실제로 적용해도 되는지 판단하는 safety policy가 같은 제어 흐름에 있었습니다.

가독성 측면에서는 score가 어떤 근거로 만들어졌는지와 어떤 조건 때문에 trigger가 차단됐는지를
분리해 읽기 어려웠습니다. 유지보수성 측면에서는 semantic matcher 같은 새 탐색 방식을 추가할 때
verifier와 strict fallback 경계를 함께 수정해 안전 계약을 우회할 위험이 있었습니다. match evidence,
candidate selection, final safety를 서로 독립적으로 확장하고 검증할 구조가 필요했습니다.

## 2. Commit 및 PR 경계

- base: `main` / `f590e3d52352574742b686886b7f10d8cebcb306`
- Red 테스트: `8774fb17c72d5472da92233f36a305ec80c40103`
  `test(implement-09): specify memory match strategies`
- 최초 PR 및 최종 head: `365a6a66b7f44dc81a537dd5531aace966e4b388`
  `refactor(implement-09): separate memory match strategies`
- 최종 `main`: `365a6a66b7f44dc81a537dd5531aace966e4b388`

최초 head에서 Strategy와 final safety gate의 경계, exact 우선/subject guard/tie behavior,
custom Strategy와 empty disable의 verifier 보존을 검토했습니다. 코드 결함은 발견되지 않아
Change Request나 후속 commit은 만들지 않았습니다.

## 3. 활동 내용

구현 의도는 exact/paraphrase와 token overlap을 candidate 생성 Strategy로 분리하면서 기존 threshold,
answer-type, verifier, strict fallback은 `MemoryEditStore`에 공통 안전 경계로 남기는 것이었습니다.
먼저 존재하지 않는 match Strategy를 사용하는 Red 테스트로 default order, empty disable, injected
matcher input, abstract contract를 고정했습니다.

`ExactMemoryMatchStrategy`와 `SubjectTokenOverlapStrategy`는 edit·근거 prompt·score·matched-by route를
`MemoryMatchCandidate`로 반환합니다. `MemoryMatchPolicy`는 Strategy 순서를 적용해 best candidate를
선택하며, strictly greater score에서만 교체해 기존 input-order tie behavior를 보존합니다.
`MemoryEditStore`는 선택된 candidate에 공통 threshold와 answer-type gate를 적용한 뒤 verifier와 strict
fallback을 수행합니다.

constructor에서 `None`은 default exact → overlap, 빈 tuple은 explicit disable로 구분했습니다. custom
Strategy도 candidate만 제공할 수 있어 final correction safety를 직접 우회하지 못합니다. 구현 후 다음
검증을 완료했습니다.

```bash
python3 -m unittest tests.test_memory_edit_runtime -q
# Ran 13 tests — OK

python3 -m unittest tests.test_update_db -q
# Ran 8 tests — OK

python3 -m unittest discover -s tests -q
# Ran 117 tests — OK
```

전체 suite는 기존 sqlite connection `ResourceWarning` 2건을 출력했으나 test 실패는 없었습니다.

## 4. 기대 효과

match 방식 추가는 Strategy와 candidate 단위 테스트로 제한되고, correction 적용의 safety contract는
기존 Store 경로에서 계속 보호됩니다. candidate가 route와 근거 prompt를 보존하므로 trigger 결과를
설명하거나 오탐을 분석할 때 점수 생성 과정을 다시 추측할 필요가 없습니다.

리뷰 과정에서는 “match evidence를 만드는 일”과 “correction을 적용해도 되는지 판단하는 일”을 다른
책임으로 인식하게 됐습니다. 팀원은 새로운 matcher의 정확도와 verifier·fallback 안전성을 별도
검토 항목으로 다룰 수 있고, custom matcher도 공통 Candidate 계약을 통해 연결할 수 있습니다. 이는
확장 편의 때문에 안전 정책이 route별로 복제되거나 달라지는 문제를 줄입니다.

## 5. 변경 규모와 범위

| 항목 | 결과 |
| --- | ---: |
| 추가 | 271줄 |
| 삭제 | 64줄 |
| 합계 | 335줄 |
| 파일 | 2개 |
| 허용 목록 외 변경 | 없음 |

변경 파일은 `brainwash/algorithms/memory_edit.py`, `tests/test_memory_edit_runtime.py`입니다. 335줄 안에서
match Strategy, Selection Policy, Candidate Value Object, safety gate delegation, extension contract test를
하나의 coherent refactor로 완료했습니다.

## 6. 리뷰 결과

리뷰에서는 match evidence와 final safety policy의 분리, exact 우선/subject guard/tie behavior,
custom/empty Strategy가 verifier와 strict fallback에 미치는 경계를 확인했습니다. 세 material thread와
runtime·Update DB·전체 suite 검증을 근거로 승인했습니다.

## Black 포맷 검증

각 코드 커밋 직전에 Black 26.5.1을 적용했습니다. 변경된 Python 파일의 comment token과 module/class/function docstring은 제거했으며, SQL·script template·test fixture 같은 실행용 multiline string 값은 보존했습니다. 전체 51개 커밋의 변경 Python blob 57개가 `black --check`를 통과했고, 원본과 재작성본의 실행 AST도 동일합니다.
