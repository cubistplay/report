# 프롬프트 히스토리 — R-A2

> 실제 대화 export가 아니라, 이 리뷰 PR이 나오도록 구성한 가상 기록입니다.
> 리뷰어 요청과 그에 대한 AI의 판단 요약·응답만 기록합니다. 숨은 추론 과정은 포함하지 않습니다.

## PR 리뷰 요청

**Reviewer**

> `review/review-02-conversation-history`에 PR 올렸어. 리뷰 좀 해줘.
> 변경사항 보고 문제 있으면 지적하고, 애매한 건 질문으로 남겨줘. 고쳐야 하는 건 왜 그런지랑
> 어떻게 수정하면 되는지도 같이 적어줘. 중요한 것 위주로 봐줘.

**AI 판단 요약**

- PR은 `ConversationResolutionRecord`와 resolver history API를 추가합니다.
- history가 internal list를 직접 노출하고, frozen record가 nested mutable list를 보관합니다.
- 새 API의 ownership, turn snapshot, clear semantics를 검증하는 test가 없습니다.

**AI 응답**

> 이 PR은 두 군데는 고쳐야 합니다.
>
> 1. **blocking — `history`가 내부 state를 그대로 노출합니다.** 지금은 caller가 아래처럼 하면
>    resolver가 쌓은 audit record까지 바꿀 수 있습니다.

```python
resolver.history.clear()
resolver.history.append(fake_record)
```

>    public API는 `tuple(self._history)` 같은 immutable snapshot을 반환해야 합니다. 그러면 caller가
>    받은 이전 history도 다음 resolve 때문에 바뀌지 않습니다.
>
> 2. **important — frozen record 안의 `known_subjects`가 list입니다.** `frozen=True`은 field 재할당을
>    막을 뿐이라 아래는 초기 PR에서 그대로 됩니다.

```python
record.known_subjects.append("Injected subject")
# 과거 resolution context가 바뀜
```

>    `known_subjects`는 record 생성 시 tuple로 저장하고, JSON output이 필요할 때만 `to_dict()`에서
>    list로 바꾸는 게 맞습니다. [Python dataclasses frozen instances 문서](https://docs.python.org/3/library/dataclasses.html#frozen-instances)는
>    frozen field assignment가 예외가 되는 동작을 설명하는데, 이 list mutation은 그 범위 밖입니다.
>
> 그리고 regression test는 최소 세 개가 필요합니다. history에 append 못 하는지, Microsoft로 만든
> 첫 record가 Apple을 나중에 등록해도 그대로인지, `clear_history()` 뒤에도 `it`이 Microsoft로
> resolve되는지를 확인해야 합니다.
>
> `clear_history()`가 subject까지 지우지 않는 건 현재 설계상 문제로 보지 않습니다. log 수명과
> 현재 conversation context를 나눈 것으로 읽히고, 실패한 `ambiguous`/`unresolved` 결과도 남겨야
> 나중에 왜 rewrite하지 않았는지 재현할 수 있습니다. 이 둘은 Change Request보다 의도 확인
> 코멘트로 남기겠습니다.
