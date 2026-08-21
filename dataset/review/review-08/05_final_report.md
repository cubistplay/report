# R-A8 memory snapshot export 리뷰 결과 보고서

> Synthetic GitHub artifact: true

## 1. 현황 및 이슈

`feat(review-08): export memory snapshots`는 `as_of`를 받아 memory artifact bundle과
`memory_snapshot.json`을 생성합니다. 최초 PR `d49ba6b`은 `brainwash/memory/ledger.py`에 51줄을
추가하고 5줄을 수정했으며, 기존 전체 test 124건을 통과했습니다.

snapshot은 특정 시점의 상태를 재현하는 artifact인데, 최초 구현은 일부 derived artifact만 `as_of`를
받았습니다. 같은 directory의 active memory, index, conflict, promotion report가 서로 다른 시점을
보일 수 있었고 naive datetime은 실행 환경 timezone에 따라 다르게 해석될 수 있었습니다. artifact
bundle의 일관성과 audit 재현성이 코드리뷰의 핵심인 변경이므로 시간 경계와 입력 계약을 검토 대상으로
선정했습니다.

## 2. 주요 검토 및 반영

### 2.1 모든 derived artifact의 동일한 시간 경계

초기 구현은 active_memory와 memory_lifecycle에만 as_of를 전달했습니다. index, conflicts, promotion report는 현재 시각으로 계산되어 같은 snapshot directory에서 서로 다른 record set을 보일 수 있었습니다.

반영 후 write_artifacts는 하나의 validated snapshot_as_of를 active snapshot, lifecycle decision, conflicts, index, promotion report에 모두 전달합니다. 과거 snapshot에서 현재는 active인 record가 index와 promotion cluster에 나타나지 않는 test를 추가했습니다.

### 2.2 timezone-aware snapshot input

[Python datetime 공식 문서](https://docs.python.org/3/library/datetime.html#aware-and-naive-objects)는 naive datetime의 timezone 해석이 프로그램에 맡겨진다고 설명하며, aware datetime은 특정 시점을 표현합니다.

    datetime(2026, 8, 21, 9, 0)                 # timezone 미지정
    datetime(2026, 8, 21, 9, 0, tzinfo=UTC)     # 특정 시점

snapshot artifact는 audit 재현에 쓰이므로 naive as_of를 ValueError로 거절하도록 반영했습니다.

## 3. 활동 내용

리뷰 의도는 snapshot Value Object를 다시 설계하는 것이 아니라, 하나의 bundle이 하나의 시간 경계를
공유하도록 보장하는 것이었습니다. 리뷰어는 과거 `as_of`와 future record 예시를 들어 active memory는
비어 있는데 index와 promotion report에는 record가 남는 재현 사례를 제시했습니다. Python datetime
공식 문서를 근거로 naive datetime의 모호성도 설명하고, timezone-aware input만 허용하도록 요청했습니다.

작업자는 bundle boundary와 naive datetime을 먼저 실패시키는 test commit을 추가했습니다. 이후
`write_artifacts()`가 검증된 `snapshot_as_of` 하나를 모든 derived view에 전달하고, timezone 정보가 없는
값은 `ValueError`로 거절하도록 fix commit을 쌓았습니다. 리뷰 대화는 코드 위치, 시간 예시, 문서 근거,
기대 artifact를 함께 남겨 다른 팀원도 snapshot contract를 바로 확인할 수 있게 했습니다.

## 4. 기대 효과

동일한 export directory 안의 artifact가 같은 시점의 record set을 표현하므로 audit와 backfill 비교가
신뢰할 수 있게 됩니다. timezone-aware contract는 실행 호스트의 local timezone에 따라 결과가 달라지는
문제를 차단하고, snapshot manifest의 `as_of`를 실제 재현 좌표로 사용할 수 있게 합니다.

팀은 snapshot을 파일 묶음이 아니라 “하나의 validated time boundary를 공유하는 bundle”로 인식하게 됩니다.
이후 export 기능을 검토할 때는 모든 derived output에 동일한 filter가 전파되는지, 시간 입력이 명시적인지,
과거 상태를 재현하는 test가 있는지를 공통 기준으로 확인할 수 있습니다.

## 5. 커밋 및 검증

| 단계 | Commit | 내용 | 검증 |
| --- | --- | --- | --- |
| 최초 PR | d49ba6b | snapshot manifest 및 as_of export | 전체 124건 통과 |
| 리뷰 명세 | 604d638 | bundle boundary·naive datetime test | 초기 구현에서 2 failures 확인 |
| 리뷰 반영 | cf3f29b | shared as_of와 timezone validation | memory ledger 12건, 전체 126건 통과 |

최초 PR 이후에는 test commit과 fix commit을 순서대로 누적했고, rebase나 force push를 사용하지 않았습니다.

## Black 포맷 검증

각 코드 커밋 직전에 Black 26.5.1을 적용했습니다. 변경된 Python 파일의 comment token과 module/class/function docstring은 제거했으며, SQL·script template·test fixture 같은 실행용 multiline string 값은 보존했습니다. 전체 51개 커밋의 변경 Python blob 57개가 `black --check`를 통과했고, 원본과 재작성본의 실행 AST도 동일합니다.
