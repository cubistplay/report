# R-A3 최종 리뷰 보고서 — benchmark provenance metadata

## 1. 검토 배경

대상 PR은 benchmark adapter가 만든 request에 source dataset, source case, rewrite 위치를 남기는
provenance 기능을 추가했습니다. 초기 head는 전체 test 96건을 통과했지만, metadata ownership과
원본 source coordinate를 확인하는 test는 없었습니다.

리뷰에서는 provenance가 단순한 부가 정보가 아니라 report와 audit log에서 원본 sample을 다시
찾는 식별 정보라는 점을 기준으로, merge 우선순위와 MQuAKE multi-rewrite 좌표를 검토했습니다.

## 2. Commit 및 PR 경계

- base: `9ed3dea463f66d7bfb0a61c65fac329e90fb1a89`
- 최초 PR branch/head: `review/review-03-benchmark-provenance` /
  `b82ec0aaaeea827fedf4be1306af786e8a3a5763`
- 리뷰 반영 테스트: `1763aeb9cc30b58fc8299fb3bacc381c809f17c7`
  `test(review-03): specify benchmark provenance contracts`
- 리뷰 반영 수정 및 최종 head: `43064678c1ccf8126493b12908b586fb5b780910`
  `fix(review-03): preserve benchmark provenance`

리뷰 시작 뒤 최초 PR commit은 바꾸지 않았습니다. test와 fix를 새 commit으로 누적해 `main`의
선형 이력을 유지했습니다.

## 3. 발견 사항과 반영 결과

| 심각도 | 발견 사항 | 영향 | 반영 |
| --- | --- | --- | --- |
| blocking | extra metadata가 provenance key를 덮음 | audit source를 위조하거나 잃을 수 있음 | provenance를 마지막에 merge |
| important | MQuAKE index에 output count 사용 | skipped rewrite 뒤 원본 row 위치를 잃음 | `enumerate()`의 source index 사용 |
| important | provenance contract test 부재 | metadata 표현 회귀를 탐지하기 어려움 | factory·loader regression test 2건 추가 |
| question | single-rewrite의 null index | dataset별 schema 처리 방식 | null을 명시적 N/A로 유지 |
| question | request ID와 provenance reference | origin 식별 책임 중복 가능성 | 실행 ID와 source 좌표의 역할 분리 확인 |

## 4. 리뷰 품질과 협업

첫 Change Request는 merge 순서라는 코드 근거와 `{"benchmark_source": "not-the-source"}` 재현
입력을 함께 제시했습니다. provenance는 adapter가 임의로 지정하는 부가 metadata가 아니라 source
of truth이므로, factory가 마지막에 적용해야 한다는 수정 방향도 구체적으로 전달했습니다.

정보 제공으로 Python dictionary unpacking의 같은 key 처리도 코드 예시로 공유했습니다. 뒤쪽
unpacking 값이 앞쪽 값을 대체하므로 아래 순서는 source를 잃고, provenance를 마지막에 두어야 합니다.

```python
{**{"benchmark_source": "counterfact"}, **{"benchmark_source": "not-the-source"}}
# {"benchmark_source": "not-the-source"}
```

이 설명은 [Python dictionary display 문서](https://docs.python.org/3/reference/expressions.html#dictionary-displays)의
unpacking 규칙을 근거로 했으며, 링크를 열지 않아도 문제와 수정 방향을 확인할 수 있게 작성했습니다.

두 번째 Change Request는 `len(requests)`와 `enumerate()`의 `index`가 의미하는 값이 다르다는 점을
설명했습니다. 앞 rewrite가 skip되면 output 순서와 source row index가 갈라지는 입력을 test로
고정해, 단순한 naming 지적이 아니라 source traceability 오류임을 검증했습니다.

null index와 request ID 중복 여부는 즉시 결함으로 단정하지 않고 data schema와 운영 추적의
역할을 질문으로 확인했습니다. 불필요한 변경 요청을 만들지 않으면서 provenance 계약을 명확히
했습니다.

## 5. 검증

초기 PR head에서 기존 full suite를 실행했습니다.

```bash
python3 -m unittest discover -s tests -q
# Ran 96 tests — OK
```

리뷰 반영 후에는 focused test와 전체 suite를 다시 실행했습니다.

```bash
python3 -m unittest tests.test_benchmark_adapters -q
# Ran 11 tests — OK

python3 -m unittest discover -s tests -q
# Ran 98 tests — OK
```

전체 suite는 기존 sqlite connection `ResourceWarning` 2건을 출력했으나 test 실패는 없었습니다.

## 6. 변경 범위

| 구간 | 파일 | 추가 | 삭제 | 합계 |
| --- | --- | ---: | ---: | ---: |
| 최초 PR | `brainwash/benchmarks.py` | 49 | 1 | 50 |
| 리뷰 반영 | `brainwash/benchmarks.py`, `tests/test_benchmark_adapters.py` | 55 | 5 | 60 |

초기 변경량 52줄은 Review PR의 일반적인 50~200줄 범위 안입니다. provenance Value Object와
Factory metadata 경계라는 하나의 기능을 검토했고, 반영 범위도 발견한 ownership·source coordinate·test
문제에만 한정했습니다.

## Black 포맷 검증

각 코드 커밋 직전에 Black 26.5.1을 적용했습니다. 최종 변경 파일은 `black --check`를 통과했고, 재작성 전후 변경 Python 파일의 AST가 동일함을 확인했습니다. `#` 주석과 inline comment는 코드에서 제거했으며, 새 docstring은 추가하지 않았습니다.

## Black 포맷 검증

각 코드 커밋 직전에 Black 26.5.1을 적용했습니다. 최종 변경 파일은 `black --check`를 통과했고, 재작성 전후 변경 Python 파일의 AST가 동일함을 확인했습니다. `#` 주석과 inline comment는 코드에서 제거했으며, 새 docstring은 추가하지 않았습니다.

## Black 포맷 검증

각 코드 커밋 직전에 Black 26.5.1을 적용했습니다. 최종 변경 파일은 `black --check`를 통과했고, 재작성 전후 변경 Python 파일의 AST가 동일함을 확인했습니다. `#` 주석과 inline comment는 코드에서 제거했으며, 새 docstring은 추가하지 않았습니다.

