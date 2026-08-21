# 프롬프트 히스토리 — R-A10 routing lane 리뷰

> Synthetic conversation artifact: true  
> 실제 대화 export가 아니라 이 리뷰 결과를 만들기 위한 가상 리뷰 요청과 응답 요약입니다.

## 리뷰 요청

**Reviewer**

> review/review-10-routing-lanes에 mixed batch를 lane별로 나눠 route하는 PR 올렸어. fact, behavior, domain이 섞일 수 있으니까 실제로 위험한 부분 위주로 봐줘. 고칠 건 이유랑 test 방향까지 같이 적어주고, 애매한 건 질문으로 남겨줘.

## AI 판단 요약

- 기준 commit은 cd5e0db, 최초 검토 대상은 f0b3a83입니다.
- 초기 lane 분류는 behavior와 memory 두 가지뿐입니다.
- domain 또는 broad request가 memory lane에 섞이면 BroadScopeRoutingPolicy가 lane 전체를 QLoRA SFT로 route해 factual correction까지 Memory Store를 쓰지 못할 수 있습니다.
- lane 결과는 grouping 순서로 반환되지만 original input position을 남기지 않아 caller가 input-order report를 복원할 수 없습니다.
- empty input, behavior/style/safety 분류, lane ordering의 의미는 질문으로 확인할 수 있습니다.

## AI 응답

> 구조 자체는 괜찮습니다. 다만 broad/domain correction을 memory lane에 넣는 건 위험합니다. lane 전체가 broad policy로 가면 작은 fact patch까지 training route로 바뀔 수 있어요. broad를 별도 lane으로 분리하는 게 맞겠습니다.
>
> 그리고 route_lanes는 입력을 재배열하므로 request ID만으로는 original order report를 복원하기 어렵습니다. enumerate 기반 position을 lane metadata에 넣어 두면 consumer가 결과를 원래 request 순서와 연결할 수 있습니다.
