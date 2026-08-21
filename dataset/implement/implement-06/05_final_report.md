# I-A6 개발 활동 보고서 — memory promotion Policy 분리

## 1. 배경

Memory Ledger는 active record를 subject·relation·scope로 묶어 promotion report를 만들었지만,
기존 판정은 record 수가 `many_min` 이상인지만 확인했습니다. target이 충돌하거나 high-risk record가
포함돼도 training candidate처럼 보일 수 있었고, 판단 기준이 ledger persistence 로직 안에 섞여
있었습니다.

이번 변경에서는 promotion eligibility를 `PromotionPolicy`와 세 Rule Strategy로 분리했습니다.
ledger는 active record 수집과 artifact export를 유지하고, Policy는 target 일관성·최소 건수·risk
검토 필요성을 `PromotionDecision`으로 설명합니다.

## 2. Commit 및 PR 경계

- base: `main` / `52b7d93fd18947ec1e4d45f98a1341c7bb9d7827`
- Red 테스트: `c080187739c4b9fe3c2e77bd42343120b213c64b`
  `test(implement-06): specify memory promotion policy`
- 최초 PR 및 최종 head: `39dbc7a6db3580d255a596592503e63ee680d89d`
  `refactor(implement-06): separate memory promotion policy`
- 최종 `main`: `39dbc7a6db3580d255a596592503e63ee680d89d`

최초 head에서 책임 경계, high-risk override의 안전성, 기존 ledger/Update DB 계약을 검토했습니다.
코드 결함은 발견되지 않아 Change Request나 후속 commit은 만들지 않았습니다.

## 3. TDD 및 검증

Red 테스트는 아직 없는 `brainwash.memory.promotion.PromotionPolicy` import에서 실패했습니다.
일관된 target cluster, minimum record 부족, conflicting target, high-risk review, reviewed workflow의
explicit override를 먼저 명세로 고정했습니다.

구현 후 아래 검증을 완료했습니다.

```bash
python3 -m unittest tests.test_memory_ledger -q
# Ran 7 tests — OK

python3 -m unittest tests.test_update_db -q
# Ran 4 tests — OK

python3 -m unittest discover -s tests -q
# Ran 106 tests — OK
```

전체 suite는 기존 sqlite connection `ResourceWarning` 2건을 출력했으나 test 실패는 없었습니다.

## 4. 구조 개선

`PromotionRule`은 cluster를 promotion할 수 없는 사유 하나를 판단하는 Strategy 인터페이스입니다.
`MinimumRecordsRule`은 evidence 수를, `ConsistentTargetRule`은 conflicting target을,
`HighRiskReviewRule`은 manual review가 필요한 high-risk record를 확인합니다.

`PromotionPolicy`는 세 Rule의 finding을 조합해 stable한 `PromotionDecision`을 만듭니다.
Decision은 record ID·target·reason·manual review 여부를 함께 직렬화하므로 report consumer가
`ready_for_training`만 보고 안전하지 않은 cluster를 오해하지 않습니다.

`MemoryLedger.promotion_report(many_min)`는 기존 호출 형식을 보존하며 default Policy로 위임합니다.
reviewed workflow는 별도 `PromotionPolicy`를 명시적으로 전달할 때만 high-risk 검토 gate를 해제할 수
있습니다.

## 5. 변경 규모와 범위

| 항목 | 결과 |
| --- | ---: |
| 추가 | 345줄 |
| 삭제 | 28줄 |
| 합계 | 373줄 |
| 파일 | 3개 |
| 허용 목록 외 변경 | 없음 |

변경 파일은 `brainwash/memory/ledger.py`, `brainwash/memory/promotion.py`,
`tests/test_memory_ledger.py`입니다. 300줄을 넘었지만 Policy·Rule Strategy, Decision Value Object,
ledger delegation, 다섯 policy boundary test를 한 coherent unit으로 완결한 결과입니다. 가독성을
해치며 쪼개지 않았습니다.

## 6. 리뷰 결과

리뷰에서는 ledger와 promotion의 책임 경계, conflict/high-risk가 automatic training에 미치는 영향,
기존 `promotion_report(many_min)` 및 Update DB ingestion 보존을 확인했습니다. 세 개의 설계·동작
스레드에서 정책의 이유와 override 범위를 검토했고, test와 전체 suite 검증 결과를 근거로 승인했습니다.

## Black 포맷 검증

각 코드 커밋 직전에 Black 26.5.1을 적용했습니다. 최종 변경 파일은 `black --check`를 통과했고, 재작성 전후 변경 Python 파일의 AST가 동일함을 확인했습니다. `#` 주석과 inline comment는 코드에서 제거했으며, 새 docstring은 추가하지 않았습니다.

