# 프롬프트 히스토리 — R-A6 lifecycle audit import 리뷰

> Synthetic conversation artifact: true  
> 이 기록은 실제 대화 export가 아니라, 이 리뷰 결과를 만들기 위해 리뷰어가 입력할 법한 가상 요청과 응답 요약입니다.

## 리뷰 요청

**Reviewer**

> `review/review-06-lifecycle-audit`에 lifecycle audit import PR 올렸어. 변경사항 보고 문제점은 지적하고, 애매한 건 질문으로 남겨줘. 고쳐야 하면 왜 그런지랑 수정 방향도 같이 적어줘. 중요한 것 위주로 봐줘.

## AI 판단 요약

- 기준 commit은 `8291841`, 최초 검토 대상은 `8c56c1e`입니다.
- 변경은 lifecycle event table 추가와 `memory_lifecycle.jsonl` import 경로입니다.
- `active` 값은 외부 JSON에서 들어오지만 현재 truthiness로 SQLite 정수로 바뀝니다. 문자열 `"false"`도 참으로 처리될 수 있습니다.
- `ingest_run_artifacts()`가 lifecycle row를 적재하지만 `UpdateDbIngestReport`에는 개수가 남지 않습니다.
- memory ledger를 먼저 import한 뒤 lifecycle row를 넣는 순서, 중복 import, timestamp 정렬은 현재 schema와 호출 순서를 함께 확인할 질문입니다.

## AI 응답

> 전체 흐름은 자연스럽습니다. ledger를 먼저 적재하고 lifecycle audit를 넣어서 foreign key 순서도 맞습니다. 다만 외부 artifact를 받는 경계에 두 가지는 수정이 필요해 보여요.
>
> 1. `active`를 `if row.get("active")`로 처리하면 JSON producer가 `"false"`를 보냈을 때 active=1로 저장됩니다. boolean/0·1/문자열 true·false를 명시적으로 파싱하고, 그 밖의 값은 오류로 처리하는 편이 안전합니다.
> 2. lifecycle row를 몇 건 import했는지 report에 안 남습니다. 실행 결과를 받는 쪽에서는 성공했는지 확인할 방법이 없으니 `UpdateDbIngestReport`와 `to_dict()`에 count를 포함시키는 게 좋겠습니다.
>
> 나머지는 `INSERT OR REPLACE`의 재실행 의미, ISO timestamp 정렬, source별 unique key가 호출자 의도와 맞는지만 질문으로 확인하겠습니다.
