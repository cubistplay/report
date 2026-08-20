# 프롬프트 히스토리 — R-A5

> 실제 대화 export가 아니라, 이 리뷰 PR이 나오도록 구성한 가상 기록입니다.
> 리뷰어 요청과 그에 대한 AI의 판단 요약·응답만 기록합니다. 숨은 추론 과정은 포함하지 않습니다.

## PR 리뷰 요청

**Reviewer**

> `review/review-05-support-retrieval-trace`에 PR 올렸어. 변경사항 보고 문제 있으면 지적하고,
> 애매한 건 질문으로 남겨줘. 고쳐야 하는 건 왜 그런지랑 어떻게 수정하면 되는지도 같이 적어줘.
> 중요한 것 위주로 봐줘.

**AI 판단 요약**

- PR은 support retrieval 결과를 `SupportRetrieval`로 남기고 기존 `retrieve()`는 answer만 반환하게
  위임합니다.
- semantic acceptance threshold와 trace에 기록하는 observed score가 같은 값인지, 새 trace가 기존 API
  동작을 바꾸지 않는지를 우선 확인했습니다.

**AI 응답**

> 두 가지는 반영이 필요합니다.
>
> 1. **blocking — semantic trace에 실제 similarity가 아니라 threshold `0.6`이 기록됩니다.** `0.93`으로
>    매치돼도 trace에는 `0.6`이 남아서 운영자가 결과 강도를 판단할 수 없습니다. threshold는 “수락할지”를
>    정하는 기준이고, trace score는 실제 matcher가 낸 값을 보존해야 합니다. `_semantic_retrieve()`가
>    answer와 best score를 함께 반환하도록 바꾸고 그 값을 trace에 넣어 주세요.
>
> 2. **important — 새 trace 계약을 고정하는 test가 없습니다.** 고정 score matcher로 semantic trace가
>    `0.93`을 그대로 보존하는지, 기존 `retrieve()`도 같은 answer를 반환하는지 확인해 주세요. exact match와
>    no-match를 함께 넣은 batch가 입력마다 trace 하나를 만드는 test도 있으면 batch API의 길이·정규화 계약이
>    분명해집니다.
>
> `score=None`인 no-match에 lexical best score를 남기는 현재 정책은 디버깅에 쓸 수 있으니 바로 바꿀 문제는
> 아닙니다. 다만 caller가 `source == "none"`과 `score`를 어떻게 해석하는지 문서화할 필요가 있는지는
> 질문으로 확인해 보면 좋겠습니다.
