# R-A2 최종 리뷰 보고서 — conversation resolution history

## 1. 검토 배경

대상 PR은 follow-up query의 resolution 결과와 당시 subject context를 resolver 내부 history에
남기는 기능을 추가했습니다. 초기 head는 기존 전체 test 93건을 통과했지만, 새 audit API의
불변성·turn snapshot·clear semantics를 검증하는 test는 없었습니다.

리뷰는 “frozen dataclass를 썼는가”가 아니라, 외부 caller가 실제로 과거 record와 resolver의
내부 collection을 바꿀 수 있는지를 코드로 재현하는 데 집중했습니다.

## 2. Commit 및 PR 경계

- base: `c82fa2342fb4b121d3a26fec89201e8898a20a5c`
  `fix(review-01): isolate matcher setting cache`
- 최초 PR head: `fa05516b87c31c46b05c9cfe0dc09b49aafa20b4`
  `feat(review-02): record conversation resolutions`
- 리뷰 반영 테스트: `dcc37005b6ed6bae3997e61956c394e5b3cf922f`
  `test(review-02): specify resolution history snapshots`
- 리뷰 반영 수정 및 최종 head: `9e19a339f4408c79c1d1b526fafac5a95990af45`
  `fix(review-02): protect resolution history snapshots`

리뷰 시작 뒤 최초 PR commit을 수정하지 않았습니다. regression test와 response code를 새
commit으로 누적해 `main`의 선형 이력을 유지했습니다.

## 3. 발견 사항과 반영 결과

| 심각도 | 발견 사항 | 영향 | 반영 |
| --- | --- | --- | --- |
| blocking | `history`가 내부 list를 직접 반환 | caller가 audit log를 clear/append 가능 | tuple snapshot 반환 |
| important | frozen record 내부의 `known_subjects`가 list | 과거 turn의 context를 외부에서 변경 가능 | tuple 보관, export 때만 list 변환 |
| important | 신규 history 계약 test 부재 | mutation·snapshot·clear 회귀를 탐지하기 어려움 | regression test 3건 추가 |
| question | clear와 context reset의 차이 | interactive caller가 state 수명을 오해할 위험 | history만 비우는 계약 확인 |
| question | 실패 상태 기록 | ambiguous/unresolved 원인 추적 범위 | 모든 resolution 상태 기록 확인 |

## 4. 리뷰 근거와 협업 방식

첫 지적에서는 `resolver.history.clear()`와 `resolver.history.append(fake_record)`라는 짧은
재현 code를 제시해, private attribute가 아니라 public API가 내부 state를 노출한다는 점을
분명히 했습니다.

두 번째 지적은 [Python dataclasses의 frozen instances 문서](https://docs.python.org/3/library/dataclasses.html#frozen-instances)가
**field assignment를 예외로 처리하는 동작**을 설명한다는 점을 근거로 삼았습니다. 이를
“frozen이면 내부도 모두 immutable하다”로 과장하지 않고, 아래처럼 nested list mutation이
실제로 가능한 code를 함께 제시했습니다.

```python
record.known_subjects.append("Injected subject")
# 초기 PR에서는 성공: 과거 resolution의 context가 바뀜
```

작업자는 tuple snapshot과 JSON export 경계를 제안했고, 리뷰에서는 `to_dict()`에서만 list로
변환하는 책임 분리를 확인했습니다. clear semantics와 실패 상태 기록은 결함으로 단정하지 않고
질문으로 다뤄 불필요한 Change Request를 만들지 않았습니다.

## 5. 검증

초기 PR head의 기존 full suite 결과입니다.

```bash
python3 -m unittest discover -s tests -q
# Ran 93 tests — OK
```

리뷰 반영 후에는 focused test와 전체 suite를 다시 실행했습니다.

```bash
python3 -m unittest tests.test_benchmark_adapters -q
# Ran 9 tests — OK

python3 -m unittest discover -s tests -q
# Ran 96 tests — OK
```

전체 suite는 기존 sqlite connection `ResourceWarning` 2건을 출력했으나 test 실패는 없었습니다.
이번 history 변경과 무관한 기존 경고이므로 별도 범위로 분리했습니다.

## 6. 변경 범위

| 구간 | 파일 | 추가 | 삭제 | 합계 |
| --- | --- | ---: | ---: | ---: |
| 최초 PR | `brainwash/conversation.py` | 65 | 4 | 69 |
| 리뷰 반영 | `brainwash/conversation.py`, `tests/test_benchmark_adapters.py` | 43 | 5 | 48 |

초기 변경량 81줄은 Review PR의 일반적인 50~200줄 범위 안이며, in-memory conversation audit
history라는 하나의 검토 단위를 완결합니다. 리뷰 반영은 발견한 mutability와 테스트 공백만
해결했고, 허용된 source/test 파일 밖으로 범위를 넓히지 않았습니다.

## Black 포맷 검증

각 코드 커밋 직전에 Black 26.5.1을 적용했습니다. 변경된 Python 파일의 comment token과 module/class/function docstring은 제거했으며, SQL·script template·test fixture 같은 실행용 multiline string 값은 보존했습니다. 전체 51개 커밋의 변경 Python blob 57개가 `black --check`를 통과했고, 원본과 재작성본의 실행 AST도 동일합니다.
