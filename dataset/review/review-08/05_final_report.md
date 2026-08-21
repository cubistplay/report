# R-A8 memory snapshot export 리뷰 결과 보고서

> Synthetic GitHub artifact: true

## 1. 검토 대상

feat(review-08): export memory snapshots는 lifecycle as_of를 받아 memory artifact bundle과 memory_snapshot.json manifest를 생성합니다. 최초 PR 7b91748은 brainwash/memory/ledger.py에 53줄을 추가하고 5줄을 수정했으며, 기존 전체 test 124건을 통과했습니다.

## 2. 주요 검토 및 반영

### 2.1 모든 derived artifact의 동일한 시간 경계

초기 구현은 active_memory와 memory_lifecycle에만 as_of를 전달했습니다. index, conflicts, promotion report는 현재 시각으로 계산되어 같은 snapshot directory에서 서로 다른 record set을 보일 수 있었습니다.

반영 후 write_artifacts는 하나의 validated snapshot_as_of를 active snapshot, lifecycle decision, conflicts, index, promotion report에 모두 전달합니다. 과거 snapshot에서 현재는 active인 record가 index와 promotion cluster에 나타나지 않는 test를 추가했습니다.

### 2.2 timezone-aware snapshot input

[Python datetime 공식 문서](https://docs.python.org/3/library/datetime.html#aware-and-naive-objects)는 naive datetime의 timezone 해석이 프로그램에 맡겨진다고 설명하며, aware datetime은 특정 시점을 표현합니다.

    datetime(2026, 8, 21, 9, 0)                 # timezone 미지정
    datetime(2026, 8, 21, 9, 0, tzinfo=UTC)     # 특정 시점

snapshot artifact는 audit 재현에 쓰이므로 naive as_of를 ValueError로 거절하도록 반영했습니다.

## 3. 커밋 및 검증

| 단계 | Commit | 내용 | 검증 |
| --- | --- | --- | --- |
| 최초 PR | 7b91748 | snapshot manifest 및 as_of export | 전체 124건 통과 |
| 리뷰 명세 | 6b31cc5 | bundle boundary·naive datetime test | 초기 구현에서 2 failures 확인 |
| 리뷰 반영 | 6950e7a | shared as_of와 timezone validation | memory ledger 12건, 전체 126건 통과 |

최초 PR 이후에는 test commit과 fix commit을 순서대로 누적했고, rebase나 force push를 사용하지 않았습니다.

## 4. 결론

Snapshot Manifest Value Object 구조를 유지하면서, bundle 내부의 시간 일관성과 재현 가능한 timezone contract를 보완했습니다. 원본 ledger는 audit 입력으로 유지하고, filtered artifact는 같은 snapshot boundary를 공유합니다.

## Black 포맷 검증

각 코드 커밋 직전에 Black 26.5.1을 적용했습니다. 최종 변경 파일은 `black --check`를 통과했고, 재작성 전후 변경 Python 파일의 AST가 동일함을 확인했습니다. `#` 주석과 inline comment는 코드에서 제거했으며, 새 docstring은 추가하지 않았습니다.

