# 프롬프트 히스토리 — R-A4

> 실제 대화 export가 아니라, 이 리뷰 PR이 나오도록 구성한 가상 기록입니다.
> 리뷰어 요청과 그에 대한 AI의 판단 요약·응답만 기록합니다. 숨은 추론 과정은 포함하지 않습니다.

## PR 리뷰 요청

**Reviewer**

> `review/review-04-sft-summary`에 PR 올렸는데 리뷰 좀 해줘.
> 변경사항 보고 문제 있으면 지적하고, 애매한 건 질문으로 남겨줘. 고쳐야 하는 건 왜 그런지랑
> 어떻게 수정하면 되는지도 같이 적어줘. 중요한 것 위주로 봐줘.

**AI 판단 요약**

- PR은 SFT preparation의 input/output count를 `SftDatasetSummary` JSON과 config·note에 기록합니다.
- summary가 무엇을 분모로 계산하는지와 file이 result manifest에서 발견되는지를 우선 확인했습니다.

**AI 응답**

> 두 가지는 고쳐야 합니다.
>
> 1. **blocking — `total_requests`가 전체 입력이 아니라 training row 수입니다.** target 없는 request를
>    skip하면 `rows`에는 안 남는데, 지금 summary는 `len(rows)`를 total로 씁니다. 요청 2개 중 1개를
>    skip해도 `1/1`, ratio `1.0`으로 보여 data loss를 숨깁니다. summary 생성에는 `len(requests)`를
>    넘겨야 합니다.
>
> 2. **important — summary file을 만들고도 manifest artifact에 안 넣습니다.** file은 output directory에
>    있지만 pipeline consumer는 `result.artifacts`와 manifest만 보고 run output을 찾습니다. summary
>    path를 `dataset_summary` artifact로 반환해야 합니다.
>
> regression test는 target 있는 request 하나, 없는 request 하나로 summary가 `2 / 1 / 1 / 0.5`를
> 기록하는지 보고, manifest의 `dataset_summary`가 실제 summary path인지까지 확인하면 됩니다.
> empty dataset에서 ratio를 `0.0`으로 두는 건 division-by-zero를 피하고 report consumer가 처리하기
> 쉬운 정책이라 문제로 보진 않습니다. config 안 summary와 note가 같은 summary object를 쓰는지도
> 질문으로 확인하면 충분합니다.
