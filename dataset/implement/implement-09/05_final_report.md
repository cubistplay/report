# I-A9 개발 활동 보고서 — memory match Strategy 분리

## 1. 배경

`MemoryEditStore.evaluate_trigger()`는 edit 후보 탐색과 safety policy를 한 loop에서 처리했습니다.
exact/paraphrase, subject-aware token overlap, best score 선택, threshold, answer-type validation이
결합돼 있어 새로운 match 방식 추가 시 strict fallback 경계까지 함께 변경해야 했습니다.

이번 변경에서는 match evidence 생성을 `MemoryMatchStrategy`로, best candidate 선택을
`MemoryMatchPolicy`로 분리했습니다. `MemoryEditStore`는 threshold·answer-type·verifier·strict fallback을
계속 한 곳에서 적용합니다.

## 2. Commit 및 PR 경계

- base: `main` / `ec723b789b0df00ee92ccf3dff053650442f9a96`
- Red 테스트: `975afcab3e9a5380595d906f3d077f3c9a58da63`
  `test(implement-09): specify memory match strategies`
- 최초 PR 및 최종 head: `1fe0233c8be14d36b847804b5e820d98b9ef2ebf`
  `refactor(implement-09): separate memory match strategies`
- 최종 `main`: `1fe0233c8be14d36b847804b5e820d98b9ef2ebf`

최초 head에서 Strategy와 final safety gate의 경계, exact 우선/subject guard/tie behavior,
custom Strategy와 empty disable의 verifier 보존을 검토했습니다. 코드 결함은 발견되지 않아
Change Request나 후속 commit은 만들지 않았습니다.

## 3. TDD 및 동작 보존 검증

Red 테스트는 존재하지 않는 match Strategy import에서 실패했습니다. default Strategy order, empty
disable, injected matcher input, abstract contract를 먼저 명세로 고정했습니다.

구현 후 아래 검증을 완료했습니다.

```bash
python3 -m unittest tests.test_memory_edit_runtime -q
# Ran 13 tests — OK

python3 -m unittest tests.test_update_db -q
# Ran 8 tests — OK

python3 -m unittest discover -s tests -q
# Ran 117 tests — OK
```

전체 suite는 기존 sqlite connection `ResourceWarning` 2건을 출력했으나 test 실패는 없었습니다.

## 4. 구조 개선

`ExactMemoryMatchStrategy`는 canonical prompt와 paraphrase의 exact match를, `SubjectTokenOverlapStrategy`는
subject token이 있는 query의 lexical overlap을 `MemoryMatchCandidate`로 반환합니다. candidate는 edit,
근거 prompt, score, matched-by route를 함께 보존합니다.

`MemoryMatchPolicy`는 strategy 순서를 edit마다 적용하고, score가 큰 candidate만 선택해 기존 input-order
tie behavior를 유지합니다. `MemoryEditStore`는 선택된 candidate에 common threshold와 answer-type gate를
적용하고 verifier·strict fallback을 수행합니다.

`None` Strategy 설정은 default exact → overlap order를 사용하고, 빈 tuple은 explicit trigger disable을
뜻합니다. custom Strategy는 raw query와 각 edit을 받지만 final correction safety를 직접 우회할 수
없습니다.

## 5. 변경 규모와 범위

| 항목 | 결과 |
| --- | ---: |
| 추가 | 176줄 |
| 삭제 | 33줄 |
| 합계 | 209줄 |
| 파일 | 2개 |
| 허용 목록 외 변경 | 없음 |

변경 파일은 `brainwash/algorithms/memory_edit.py`, `tests/test_memory_edit_runtime.py`입니다. 209줄 안에서
match Strategy, Selection Policy, Candidate Value Object, safety gate delegation, extension contract test를
하나의 coherent refactor로 완료했습니다.

## 6. 리뷰 결과

리뷰에서는 match evidence와 final safety policy의 분리, exact 우선/subject guard/tie behavior,
custom/empty Strategy가 verifier와 strict fallback에 미치는 경계를 확인했습니다. 세 material thread와
runtime·Update DB·전체 suite 검증을 근거로 승인했습니다.
