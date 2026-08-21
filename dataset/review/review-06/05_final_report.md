# R-A6 lifecycle audit import 리뷰 결과 보고서

> Synthetic GitHub artifact: true

## 1. 현황 및 이슈

`feat(review-06): import memory lifecycle audits`는 외부 `memory_lifecycle.jsonl`을 DB event로
적재하는 변경입니다. 최초 PR은 `b61a2d0`이며 schema와 import 경로에 56줄을 추가했고, 전체 test
120건은 통과했습니다.

다만 전체 test 통과만으로 외부 artifact의 타입 계약과 import 결과의 관찰 가능성이 보장되지는
않았습니다. 문자열 `"false"`를 Python truthiness로 처리하면 active event로 저장될 수 있고, lifecycle
처리 건수가 report에 없으면 정상 처리와 누락을 구분할 수 없습니다. 외부 데이터가 DB 상태를 바꾸는
경로라 코드리뷰에서 타입·관찰성·재실행 계약을 우선 확인할 필요가 있어 이 활동을 선정했습니다.

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

## 3. 활동 내용

리뷰 의도는 구현의 기본 구조를 바꾸기보다, 실제 운영 입력이 지나가는 경계에서 모호한 값을
명시적인 계약으로 바꾸는 것이었습니다. 리뷰어는 `bool("false")`의 실제 동작과 Python 공식 문서를
근거로 제시하고, 허용할 boolean 표현과 실패해야 할 값을 구체적으로 요청했습니다. 또한 import가
성공했는지 caller가 알 수 있도록 `UpdateDbIngestReport`에 lifecycle count를 남기도록 제안했습니다.

리뷰 문화 측면에서는 “문제가 있습니다”에 그치지 않고 재현 가능한 입력, 기대 저장값, 필요한
회귀 테스트를 함께 제시했습니다. 작업자는 문자열 boolean·잘못된 문자열·report count를 test로 먼저
고정하고, 별도 fix commit에서 정규화와 관찰성을 반영했습니다. 이 과정으로 schema, importer, report가
서로 어떤 계약을 갖는지 스레드에서 검토 가능해졌습니다.

## 4. 기대 효과

잘못된 lifecycle artifact가 조용히 active event로 저장되는 위험이 줄고, 운영자는 report만으로
lifecycle import 처리량을 확인할 수 있습니다. unique identity와 foreign key 순서도 함께 확인돼
재실행·적재 순서 변경 시의 중복과 orphan event 위험을 조기에 발견할 수 있습니다.

팀은 외부 JSON 입력을 Python의 일반 truthiness에 맡기지 않고, 허용값·거절값·저장값을 test로
명시해야 한다는 기준을 공유하게 됩니다. 앞으로 유사한 importer 리뷰에서도 타입 변환, 결과 report,
재실행 idempotency를 공통 체크 항목으로 사용할 수 있습니다.

## 5. 커밋 및 검증

| 단계 | Commit | 내용 | 검증 |
| --- | --- | --- | --- |
| 최초 PR | `b61a2d0` | lifecycle audit table 및 import | 전체 120건 통과 |
| 리뷰 명세 | `ab61715` | 문자열 boolean과 report count 회귀 test | 초기 구현에서 1 error, 1 failure 확인 |
| 리뷰 반영 | `8e32f99` | 값 정규화와 report count | update DB 10건, 전체 122건 통과 |

최초 PR 이후에는 rebase나 force push 없이 test commit과 fix commit을 순서대로 누적했습니다.

## Black 포맷 검증

각 코드 커밋 직전에 Black 26.5.1을 적용했습니다. 변경된 Python 파일의 comment token과 module/class/function docstring은 제거했으며, SQL·script template·test fixture 같은 실행용 multiline string 값은 보존했습니다. 전체 51개 커밋의 변경 Python blob 57개가 `black --check`를 통과했고, 원본과 재작성본의 실행 AST도 동일합니다.
