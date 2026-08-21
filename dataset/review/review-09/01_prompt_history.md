# 프롬프트 히스토리 — R-A9 memory trigger batch 리뷰

> Synthetic conversation artifact: true  
> 실제 대화 export가 아니라 이 리뷰 결과를 만들기 위한 가상 리뷰 요청과 응답 요약입니다.

## 리뷰 요청

**Reviewer**

> review/review-09-memory-trigger-batch에 batch trigger API 올렸어. evaluation에서 쓸 거라 입력이랑 결과를 맞춰 보기 쉬워야 해. 문제 있으면 지적하고, 꼭 고쳐야 하는 거랑 확인만 하면 되는 건 나눠서 봐줘.

## AI 판단 요약

- 기준 commit은 cf3f29b, 최초 검토 대상은 5ae605a입니다.
- evaluate_many가 prompt를 dict key로 사용해 같은 prompt가 반복되면 한 result로 합쳐집니다.
- triggered_count가 실제 memory_triggered 여부 대신 trigger mapping 길이를 반환합니다.
- retrieve_many도 동일 mapping을 사용하므로 입력 cardinality를 보장하지 못합니다.
- duplicate evaluation 비용, no-match reason 집계, JSON serialization 형태는 질문으로 확인할 수 있습니다.

## AI 응답

> 두 부분은 수정이 필요해 보여요.
>
> 1. batch API는 evaluation input과 zip해서 쓸 가능성이 높은데 dict는 동일 prompt를 한 key로 합칩니다. list 또는 tuple 기반 결과로 입력 순서와 중복을 보존해야 합니다.
> 2. triggered_count는 batch에 result가 몇 개인지가 아니라 memory_triggered가 true인 result 수여야 합니다. no-match batch가 1건인데 triggered_count=1로 나오는 건 summary가 잘못된 상태입니다.
>
> 나머지는 duplicate prompt를 매번 계산하는지, reason aggregation이 positional result와 별개인지 확인하겠습니다.
