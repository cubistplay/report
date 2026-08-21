# 프롬프트 히스토리 — R-A8 memory snapshot export 리뷰

> Synthetic conversation artifact: true  
> 실제 대화 export가 아니라 이 리뷰 결과를 만들기 위한 가상 리뷰 요청과 응답 요약입니다.

## 리뷰 요청

**Reviewer**

> review/review-08-memory-snapshots에 PR 올렸어. 특정 시점 기준으로 memory artifact를 뽑는 기능이야. 문제 있는 부분이랑 확인만 하면 되는 부분 구분해서 리뷰해줘. 고쳐야 하면 이유랑 테스트 방향도 같이 적어줘.

## AI 판단 요약

- 기준 commit은 9234442, 최초 검토 대상은 7b91748입니다.
- active_memory와 memory_lifecycle은 as_of를 받지만 conflicts, index, promotion report는 현재 시각으로 계산합니다.
- 따라서 하나의 export directory 안에서 manifest는 과거 시점을 말하는데 index와 promotion 결과는 현재 state가 될 수 있습니다.
- naive datetime은 시스템 timezone에 따라 의미가 달라지는 boundary인데 snapshot API가 이를 허용합니다.
- 원본 ledger를 그대로 보존하는 이유, 빈 ledger manifest, conflict artifact의 역할은 질문으로 확인할 수 있습니다.

## AI 응답

> snapshot manifest를 별도 Value Object로 둔 건 좋습니다. 다만 snapshot이라고 부르려면 모든 derived artifact가 같은 as_of를 써야 합니다. 지금은 active/lifecycle만 과거 기준이고 index·promotion·conflicts는 현재 기준이라 bundle 내부가 모순될 수 있어요.
>
> 또 naive datetime은 UTC인지 local time인지 해석이 정해져 있지 않습니다. export 결과가 실행 머신 timezone에 따라 달라질 수 있으니, 이 public API에서는 timezone-aware datetime만 받게 하는 편이 안전합니다.
