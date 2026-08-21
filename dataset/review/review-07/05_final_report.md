# R-A7 리뷰를 통한 promotion candidate 비결정성 위험 제거

> Synthetic GitHub artifact: true

## 1. 현황 및 이슈

`feat(review-07): list promotion candidates`는 active memory를 key별로 묶어 training 후보를 반환합니다.
최초 PR `58a5609`은 `brainwash/memory/update_db.py`에 59줄을 추가했고, 기존 전체 test 122건을
통과했습니다.

하지만 candidate는 audit·export·training 입력으로 사용될 수 있어 출력 순서와 target 의미가 안정적이어야
합니다. 최초 구현은 `GROUP_CONCAT(id)`의 순서를 보장하지 않았고, target 비교도 기존 PromotionPolicy와
다른 raw 문자열 기준이었습니다. 같은 snapshot이 실행마다 다른 JSON을 만들거나 policy와 다른 candidate를
만들 수 있는 문제이므로, 결정성·기존 정책 정렬·회귀 테스트 관점에서 이 리뷰를 선정했습니다.

## 2. 주요 검토 및 반영

### 2.1 결정적 record ID 순서

초기 구현은 GROUP_CONCAT(id) 결과를 그대로 반환했습니다. [SQLite aggregate 함수 문서](https://www.sqlite.org/lang_aggfunc.html)는 ORDER BY가 없는 aggregate 입력 순서는 임의이며 호출마다 달라질 수 있다고 설명합니다. 특히 group_concat()은 연결된 요소의 순서가 임의라고 명시합니다.

    GROUP_CONCAT(id)  -- 출력 순서가 정해지지 않음

반영 후에는 key와 ID로 정렬된 active row를 읽어 Python에서 group을 구성했습니다. 따라서 insert 순서와 관계없이 candidate의 record_ids가 항상 같은 순서로 반환됩니다.

### 2.2 target conflict 계약 정렬

초기 PR은 raw target에 DISTINCT를 적용해 "Seoul"과 " seoul "을 충돌로 판단했습니다. 이는 공백·대소문자를 정규화하는 기존 PromotionPolicy와 달랐습니다. 반영 후에는 같은 _normalize_key()를 기준으로 target을 묶고, 정렬된 첫 label을 표시용 값으로 사용합니다.

## 3. 활동 내용

리뷰 의도는 candidate API의 Value Object 구조는 유지하면서 downstream consumer가 의존하는 출력 계약을
명확히 하는 것이었습니다. 리뷰어는 SQLite aggregate 공식 문서의 ORDER BY 없는 입력 순서 규칙을
제시하고, insert 순서와 무관한 record ID ordering을 테스트로 요청했습니다. 또한 기존
PromotionPolicy의 정규화 규칙을 비교 근거로 삼아 raw `DISTINCT`가 만드는 false conflict를 지적했습니다.

작업자는 먼저 stable order와 normalized target을 재현하는 test commit을 쌓고, fix commit에서 active row를
key와 ID 순으로 읽어 Python에서 grouping하도록 변경했습니다. target은 `_normalize_key()`로 비교하고
표시용 label은 정렬된 첫 값을 사용하도록 해 판단 key와 display 값을 분리했습니다. 리뷰 스레드는 문제,
근거, 기대 output, 회귀 test를 연결해 변경 이유를 추적 가능하게 만들었습니다.

## 4. 기대 효과

같은 active memory snapshot은 insert 순서와 관계없이 같은 candidate JSON을 반환하므로 audit diff와
training 입력이 안정됩니다. promotion candidate의 target 의미도 기존 Policy와 일치해, 조회 API와
promotion 판단이 서로 다른 결과를 내는 위험이 줄어듭니다.

팀은 aggregate 결과를 API contract로 노출할 때 순서 보장이 별도 설계 대상이라는 점을 공유하게 됩니다.
또한 기존 정책과 새 조회 API의 의미를 비교하는 리뷰 방식이 정착되면, 단순 SQL 동작 확인을 넘어
consumer 관점의 결정성과 도메인 계약까지 함께 검토할 수 있습니다.

## 5. 커밋 및 검증

| 단계 | Commit | 내용 | 검증 |
| --- | --- | --- | --- |
| 최초 PR | 58a5609 | promotion candidate Value Object 및 조회 | 전체 122건 통과 |
| 리뷰 명세 | 932c788 | stable record order·normalized target test | 초기 구현에서 2 failures 확인 |
| 리뷰 반영 | e071fe4 | 정렬 기반 grouping 및 target canonicalization | update DB 12건, 전체 124건 통과 |

최초 PR 이후에는 test commit과 fix commit을 순서대로 누적했으며, rebase나 force push를 사용하지 않았습니다.

## Black 포맷 검증

각 코드 커밋 직전에 Black 26.5.1을 적용했습니다. 변경된 Python 파일의 comment token과 module/class/function docstring은 제거했으며, SQL·script template·test fixture 같은 실행용 multiline string 값은 보존했습니다. 전체 51개 커밋의 변경 Python blob 57개가 `black --check`를 통과했고, 원본과 재작성본의 실행 AST도 동일합니다.
