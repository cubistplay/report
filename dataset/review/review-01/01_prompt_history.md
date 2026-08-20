# 프롬프트 히스토리 — R-A1

> 실제 대화 export가 아니라, 이 리뷰 PR이 나오도록 구성한 가상 기록입니다.
> 리뷰어 요청과 그에 대한 AI의 판단 요약·응답만 기록합니다. 숨은 추론 과정은 포함하지 않습니다.

## PR 리뷰 요청

**Reviewer**

> `review/review-01-matcher-settings`에 PR 올렸는데 리뷰 좀 해줘.
> 변경사항 보고 문제 있으면 지적하고, 애매한 건 질문으로 남겨줘. 고쳐야 하는 건 왜 그런지랑
> 어떻게 수정하면 되는지도 같이 적어줘. 중요한 것 위주로 봐줘.

**AI 판단 요약**

- PR은 `MatcherSettings`로 environment resolution을 모으고 matcher cache를 계속 사용하는 구조입니다.
- cache key가 model/provider를 구분하지 않고, 새 environment contract의 test가 없습니다.
- 기본 model string은 두 곳에 중복되어 있으며, unknown value fallback과 explicit override 우선순위는 기존 정책과 비교가 필요합니다.

**AI 응답**

> 봤습니다. 바로 막아야 할 건 두 가지입니다.
>
> 1. **blocking — cache key가 설정을 잃습니다.** `EmbeddingMatcher`는 model name을, LLM matcher는
>    provider를 instance 생성에 쓰는데 cache key는 `embedding`/`llm`뿐입니다. 그래서 같은 process에서
>    `model/a` 다음 `model/b`를 선택하면 두 번째도 첫 matcher를 받습니다. key를 `(backend, setting)`
>    형태로 바꿔야 합니다. `auto`와 명시적 embedding은 같은 model일 때만 같은 instance를 공유하면 됩니다.
>
> 2. **important — 새 설정 경로에 regression test가 없습니다.** model 변경, provider 변경, 빈
>    environment default를 고정해야 합니다. test에서는 아래처럼 block 안에서만 환경을 바꾸면 끝난
>    뒤 기존 `os.environ`으로 자동 복구돼 다른 test에 안 샙니다.

```python
with patch.dict(os.environ, {"BRAINWASH_MATCHER": "embedding"}, clear=True):
    matcher = get_matcher()
# 여기서는 원래 os.environ으로 복구됨
```

> LLM은 실제 API를 부르지 말고 `_make_llm_matcher` factory를 stub으로 바꾸면 provider가 제대로
> 전달되는지도 확인할 수 있습니다. [Python `patch.dict` 문서](https://docs.python.org/3/library/unittest.mock.html#patch.dict)도
> 이 scoped dictionary patch 방식을 설명합니다.
>
> 그리고 기본 embedding model 문자열이 settings와 constructor에 각각 있어서 한쪽만 바뀌면
> 달라질 수 있습니다. `DEFAULT_EMBEDDING_MODEL` 하나로 묶는 건 같이 반영하면 좋겠습니다.
>
> 반대로 unknown matcher 값을 `None`으로 처리하는 건 기존 fallback 정책이라 Change Request로
> 보진 않았습니다. `set_matcher()`로 주입한 matcher가 environment보다 먼저 반환되는 것도 유지되고
> 있습니다. 이 두 개는 의도 확인 코멘트 정도면 충분합니다.
