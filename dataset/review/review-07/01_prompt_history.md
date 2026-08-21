# 프롬프트 히스토리 — R-A7 promotion candidate 리뷰

> Synthetic conversation artifact: true  
> 실제 대화 export가 아니라 이 리뷰 결과를 만들기 위한 가상 리뷰 요청과 응답 요약입니다.

## 리뷰 요청

**Reviewer**

> review/review-07-promotion-candidates에 PR 올렸어. promotion 후보 조회 로직이야. 변경사항 보고 문제 있으면 지적하고, 애매한 부분은 질문으로 남겨줘. 수정이 필요하면 이유랑 방향도 같이 적어줘. 너무 사소한 건 빼고 중요한 것만 봐줘.

## AI 판단 요약

- 기준 commit은 d99d223, 최초 검토 대상은 5be9d0c입니다.
- 초기 PR은 active memory를 SQL aggregate로 묶어 threshold를 넘긴 PromotionCandidate를 만듭니다.
- GROUP_CONCAT(id)와 GROUP_CONCAT(DISTINCT target)에 aggregate 내부 순서가 없어 exported record_ids가 결정적이지 않습니다.
- raw target의 DISTINCT 비교는 기존 PromotionPolicy의 공백·대소문자 정규화와 다르게 conflict를 만들 수 있습니다.
- threshold validation, high-risk candidate의 수동 검토 의미, lifecycle audit와 persisted status의 책임은 확인 질문으로 남길 수 있습니다.

## AI 응답

> 후보를 별도 Value Object로 만든 방향은 괜찮습니다. 다만 training 전 단계의 결과라서 두 계약은 고쳐야 할 것 같아요.
>
> 1. GROUP_CONCAT의 ID 순서는 SQL에서 보장하지 않습니다. 같은 memory 집합인데 결과 JSON의 record_ids가 달라지면 audit diff와 downstream dataset이 불안정해집니다. 정렬된 row를 기준으로 후보를 만들고 regression test를 추가하는 게 좋겠습니다.
> 2. target은 "Seoul"과 " seoul "을 다른 값으로 봅니다. 기존 promotion policy는 이를 같은 target으로 보므로 candidate API가 더 보수적인 conflict를 만들어 버립니다. 같은 정규화 규칙을 재사용해 주세요.
>
> many_min, high-risk, lifecycle event의 의미는 구현 의도를 확인한 뒤 추가 변경 여부를 결정하겠습니다.
