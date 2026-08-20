# 프롬프트 히스토리 — R-A3

> 실제 대화 export가 아니라, 이 리뷰 PR이 나오도록 구성한 가상 기록입니다.
> 리뷰어 요청과 그에 대한 AI의 판단 요약·응답만 기록합니다. 숨은 추론 과정은 포함하지 않습니다.

## PR 리뷰 요청

**Reviewer**

> `review/review-03-benchmark-provenance`에 PR 올렸는데 리뷰 좀 해줘.
> 변경사항 보고 문제 있으면 지적하고, 애매한 건 질문으로 남겨줘. 고쳐야 하는 건 왜 그런지랑
> 어떻게 수정하면 되는지도 같이 적어줘. 중요한 것 위주로 봐줘.

**AI 판단 요약**

- PR은 adapter가 만든 request metadata에 benchmark source, case ID, rewrite index, reference를 추가합니다.
- factory metadata merge 순서, MQuAKE의 source coordinate, raw case ID 표현, 신규 provenance test 공백을 확인했습니다.

**AI 응답**

> 봤습니다. 고쳐야 할 건 두 가지입니다.
>
> 1. **blocking — provenance가 adapter extra metadata에 덮입니다.** factory가 provenance를 먼저
>    넣고 `extra_metadata`를 나중에 merge해서 `{"benchmark_source": "wrong"}` 같은 값이 source를
>    바꿀 수 있습니다. provenance는 원본 추적용이라 caller가 덮으면 안 됩니다. extra metadata를
>    먼저 합치고 provenance를 마지막에 적용해야 합니다.
>
> 2. **important — MQuAKE rewrite index가 source index가 아닙니다.** 현재 global request 개수인
>    `len(requests)`를 넣고 있어서 앞 rewrite가 skip되거나 앞 case가 있으면 원본 `requested_rewrite`
>    위치와 달라집니다. 이미 loop의 `enumerate()`가 주는 `index`를 써야 합니다.
>
> test는 provenance key override가 막히는지, 첫 rewrite를 skip한 뒤 두 번째 rewrite가 `#1`로
> 기록되는지, numeric case ID가 stable string과 reference로 남는지를 추가하면 됩니다. case ID를
> string으로 정규화하면 source reference도 dataset마다 같은 타입으로 다룰 수 있습니다.
>
> `benchmark_rewrite_index=None`을 single-rewrite dataset에도 남기는 건 JSON에서 명시적인 “해당 없음”
> 표현이라 문제로 보진 않았습니다. source coordinate가 request ID와 별개로 남는 설계도 운영 추적에는
> 적절해 보이므로 질문으로만 확인하면 충분합니다.
