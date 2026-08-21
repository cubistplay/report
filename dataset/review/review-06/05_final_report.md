# R-A6 lifecycle audit import 리뷰 결과 보고서

> Synthetic GitHub artifact: true

## 1. 검토 대상

`feat(review-06): import memory lifecycle audits`는 run directory의 `memory_lifecycle.jsonl`을
`memory_lifecycle_events`에 적재하고 memory별 audit event를 조회하는 기능입니다.

최초 PR은 `3611312`이며, `brainwash/memory/schema.sql`과 `brainwash/memory/update_db.py`에
57줄을 추가했습니다. 이 시점의 전체 test 120건은 통과했습니다.

## 2. 주요 검토 및 반영

### 2.1 외부 boolean 값의 명시적 변환

최초 구현은 `1 if row.get("active") else 0`을 사용했습니다. 이 방식은 문자열 `"false"`를 참으로
처리할 수 있어 artifact schema의 의미를 보존하지 못했습니다.

[Python Truth Value Testing](https://docs.python.org/3/library/stdtypes.html#truth-value-testing) 문서는 빈 문자열만
false로 분류하므로, 아래 결과가 나옵니다.

```python
bool("false")  # True
```

이에 따라 `_coerce_lifecycle_active()`가 boolean, `0/1`, `"true"/"false"`만 허용하도록 반영했습니다.
그 밖의 값은 `ValueError`로 실패하여 잘못된 artifact를 조용히 상태 변경으로 저장하지 않습니다.

### 2.2 import 결과의 관찰 가능성

초기 PR은 lifecycle row를 저장했지만 그 개수를 `UpdateDbIngestReport`에 제공하지 않았습니다. 반영 후에는
`lifecycle_events` count와 `to_dict()` 출력이 추가되어 caller가 lifecycle artifact 처리 결과를 확인할 수 있습니다.

## 3. 커밋 및 검증

| 단계 | Commit | 내용 | 검증 |
| --- | --- | --- | --- |
| 최초 PR | `3611312` | lifecycle audit table 및 import | 전체 120건 통과 |
| 리뷰 명세 | `b1c13ac` | 문자열 boolean과 report count 회귀 test | 초기 구현에서 1 error, 1 failure 확인 |
| 리뷰 반영 | `581ee00` | 값 정규화와 report count | update DB 10건, 전체 122건 통과 |

최초 PR 이후에는 rebase나 force push 없이 test commit과 fix commit을 순서대로 누적했습니다.

## 4. 결론

lifecycle event를 current memory record와 분리한 Event Log 구조는 유지하면서, 외부 artifact의 타입 계약과
실행 결과 report를 보완했습니다. 재실행 시의 unique identity와 foreign key 선행 조건도 대화에서 확인해
운영 데이터의 중복 및 orphan event 위험을 명확히 했습니다.

## Black 포맷 검증

각 코드 커밋 직전에 Black 26.5.1을 적용했습니다. 최종 변경 파일은 `black --check`를 통과했고, 재작성 전후 변경 Python 파일의 AST가 동일함을 확인했습니다. `#` 주석과 inline comment는 코드에서 제거했으며, 새 docstring은 추가하지 않았습니다.

## Black 포맷 검증

각 코드 커밋 직전에 Black 26.5.1을 적용했습니다. 최종 변경 파일은 `black --check`를 통과했고, 재작성 전후 변경 Python 파일의 AST가 동일함을 확인했습니다. `#` 주석과 inline comment는 코드에서 제거했으며, 새 docstring은 추가하지 않았습니다.

## Black 포맷 검증

각 코드 커밋 직전에 Black 26.5.1을 적용했습니다. 최종 변경 파일은 `black --check`를 통과했고, 재작성 전후 변경 Python 파일의 AST가 동일함을 확인했습니다. `#` 주석과 inline comment는 코드에서 제거했으며, 새 docstring은 추가하지 않았습니다.

