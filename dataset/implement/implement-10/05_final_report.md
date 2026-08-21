# I-A10 개발 활동 보고서 — memory lifecycle Policy 통합

## 1. 현황 및 이슈

Memory DB의 `active_memory` view는 status와 validity window를 함께 적용했지만, in-memory Memory Ledger는
status만 보고 active record를 선택했습니다. 같은 record가 DB에서는 future 또는 expired로 제외되면서
active snapshot, conflict, index, promotion report에는 포함될 수 있었습니다.

가독성 측면에서는 “active”라는 동일한 용어가 DB와 Ledger에서 서로 다른 조건을 뜻했고, record가
제외된 이유도 결과에 남지 않았습니다. 유지보수성 측면에서는 timestamp parsing과 boundary 비교가
각 consumer로 퍼질 가능성이 있어 start/end semantics가 다시 달라질 수 있었습니다. 시간 기준을 한
Policy로 통합하고, 판정 결과와 이유를 명시적인 값으로 남길 필요가 있었습니다.

## 2. Commit 및 PR 경계

- base: `main` / `365a6a66b7f44dc81a537dd5531aace966e4b388`
- Red 테스트: `f930e71b9b08bb62b2565f091fb48a444815e92a`
  `test(implement-10): specify memory lifecycle policy`
- 최초 PR 및 최종 head: `6fcf422241c92467db2d0636affc4c8f82e1f9e8`
  `refactor(implement-10): centralize memory lifecycle policy`
- 최종 `main`: `6fcf422241c92467db2d0636affc4c8f82e1f9e8`

최초 head에서 DB/in-memory window semantics, invalid timestamp의 안전한 처리, lifecycle audit과
existing ingestion boundary를 검토했습니다. 코드 결함은 발견되지 않아 Change Request나 후속 commit은
만들지 않았습니다.

## 3. 활동 내용

구현 의도는 DB view와 Ledger가 status, valid_from, valid_until에 대해 같은 boundary semantics를
사용하도록 하고, time-dependent 결과를 재현 가능한 audit 정보로 만드는 것이었습니다. 먼저 존재하지
않는 `MemoryLifecyclePolicy`를 사용하는 Red 테스트로 injected `as_of`에서 active, future, expired,
retired, invalid timestamp 판정과 lifecycle artifact·active snapshot 일치를 고정했습니다.

`MemoryLifecyclePolicy`는 start boundary를 inclusive, end boundary를 exclusive로 평가해 DB view와 같은
조건을 적용합니다. timestamp parsing 실패는 default active로 숨기지 않고 explicit inactive reason으로
남깁니다. `MemoryLifecycleDecision`은 record ID, active 여부, reason, as-of timestamp를 직렬화합니다.

`MemoryLedger`의 active snapshot, conflicts, exact index, promotion report는 모두 Policy가 만든 active
record를 사용합니다. optional `as_of`와 constructor-level Policy injection으로 test·backfill·historical
audit의 clock을 고정하면서도 기본 호출은 UTC now를 사용해 기존 API를 보존했습니다. 구현 후 다음
검증을 완료했습니다.

```bash
python3 -m unittest tests.test_memory_ledger -q
# Ran 10 tests — OK

python3 -m unittest tests.test_update_db -q
# Ran 8 tests — OK

python3 -m unittest discover -s tests -q
# Ran 120 tests — OK
```

전체 suite는 기존 sqlite connection `ResourceWarning` 2건을 출력했으나 test 실패는 없었습니다.

## 4. 기대 효과

DB 조회와 in-memory artifact가 동일한 lifecycle 의미를 사용하므로 future·expired record가 downstream
결과에 섞이는 drift를 줄일 수 있습니다. inactive reason과 as-of가 남아 운영자가 snapshot 차이를
재현할 수 있고, timestamp 오류도 원본 record를 다시 추적하기 전에 report에서 식별할 수 있습니다.

리뷰 과정에서는 active 여부를 단순 status field가 아니라 “특정 시점에서 Policy가 내린 결정”으로
인식하게 됐습니다. 팀원은 시간 관련 변경을 start/end boundary, timezone, evaluation clock이라는 공통
기준으로 검토할 수 있고, 새로운 consumer도 직접 날짜 비교를 구현하지 않고 lifecycle decision을
재사용할 수 있습니다. 이는 시간 의존 코드의 중복과 환경별 해석 차이를 줄이는 효과가 있습니다.

## 5. 변경 규모와 범위

| 항목 | 결과 |
| --- | ---: |
| 추가 | 220줄 |
| 삭제 | 3줄 |
| 합계 | 223줄 |
| 파일 | 2개 |
| 허용 목록 외 변경 | 없음 |

변경 파일은 `brainwash/memory/ledger.py`, `tests/test_memory_ledger.py`입니다. 223줄 안에서 lifecycle
Policy, Decision Value Object, active filtering, audit artifact, deterministic temporal test를 하나의
coherent refactor로 완료했습니다.

## 6. 리뷰 결과

리뷰에서는 DB view와 in-memory active window semantics, inactive reason/audit 책임, injected time과
existing Update DB ingestion 보존을 확인했습니다. 세 material thread와 ledger·Update DB·전체 suite
검증을 근거로 승인했습니다.

## Black 포맷 검증

각 코드 커밋 직전에 Black 26.5.1을 적용했습니다. 변경된 Python 파일의 comment token과 module/class/function docstring은 제거했으며, SQL·script template·test fixture 같은 실행용 multiline string 값은 보존했습니다. 전체 51개 커밋의 변경 Python blob 57개가 `black --check`를 통과했고, 원본과 재작성본의 실행 AST도 동일합니다.
