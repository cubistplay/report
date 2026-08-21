# R-A7 promotion candidate 리뷰 결과 보고서

> Synthetic GitHub artifact: true

## 1. 검토 대상

feat(review-07): list promotion candidates는 Update DB의 active memory를 key별로 묶어 promotion threshold를 충족한 후보를 PromotionCandidate로 반환합니다. 최초 PR 58a5609은 brainwash/memory/update_db.py에 59줄을 추가했고, 기존 전체 test 122건을 통과했습니다.

## 2. 주요 검토 및 반영

### 2.1 결정적 record ID 순서

초기 구현은 GROUP_CONCAT(id) 결과를 그대로 반환했습니다. [SQLite aggregate 함수 문서](https://www.sqlite.org/lang_aggfunc.html)는 ORDER BY가 없는 aggregate 입력 순서는 임의이며 호출마다 달라질 수 있다고 설명합니다. 특히 group_concat()은 연결된 요소의 순서가 임의라고 명시합니다.

    GROUP_CONCAT(id)  -- 출력 순서가 정해지지 않음

반영 후에는 key와 ID로 정렬된 active row를 읽어 Python에서 group을 구성했습니다. 따라서 insert 순서와 관계없이 candidate의 record_ids가 항상 같은 순서로 반환됩니다.

### 2.2 target conflict 계약 정렬

초기 PR은 raw target에 DISTINCT를 적용해 "Seoul"과 " seoul "을 충돌로 판단했습니다. 이는 공백·대소문자를 정규화하는 기존 PromotionPolicy와 달랐습니다. 반영 후에는 같은 _normalize_key()를 기준으로 target을 묶고, 정렬된 첫 label을 표시용 값으로 사용합니다.

## 3. 커밋 및 검증

| 단계 | Commit | 내용 | 검증 |
| --- | --- | --- | --- |
| 최초 PR | 58a5609 | promotion candidate Value Object 및 조회 | 전체 122건 통과 |
| 리뷰 명세 | 932c788 | stable record order·normalized target test | 초기 구현에서 2 failures 확인 |
| 리뷰 반영 | e071fe4 | 정렬 기반 grouping 및 target canonicalization | update DB 12건, 전체 124건 통과 |

최초 PR 이후에는 test commit과 fix commit을 순서대로 누적했으며, rebase나 force push를 사용하지 않았습니다.

## 4. 결론

후보 조회를 training 실행과 분리한 Value Object 구조는 유지했습니다. 다만 audit·export consumer가 의존하는 output의 결정성과 policy의 target 의미가 보완되어, 같은 active memory snapshot이 일관된 promotion candidate를 반환하게 됐습니다.

## Black 포맷 검증

각 코드 커밋 직전에 Black 26.5.1을 적용했습니다. 변경된 Python 파일의 comment token과 module/class/function docstring은 제거했으며, SQL·script template·test fixture 같은 실행용 multiline string 값은 보존했습니다. 전체 51개 커밋의 변경 Python blob 57개가 `black --check`를 통과했고, 원본과 재작성본의 실행 AST도 동일합니다.
