# R-A1 PR 대화 — semantic matcher 설정 경계 검토

> Synthetic GitHub artifact: true  
> 최초 검토 head: `7e62a05` · 최종 head: `c82fa23`

## 스레드 1 — 설정을 잃는 cache key

**위치** `brainwash/semantic.py` (`MatcherSettings.resolve`, 초기 PR 207-223행)
**심각도** `blocking` · **Change Request**

**리뷰어 · 댓글 :thinking:**

`MatcherSettings`는 `embedding_model`과 `llm_provider`를 명시적으로 들고 있지만,
`_env_matcher()`에는 각각 `"embedding"`, `"llm"`만 key로 넘깁니다.

같은 process에서 `BRAINWASH_EMBED_MODEL=model/a`로 한 번 조회한 뒤 `model/b`로 바꾸면
두 번째 호출도 `model/a` 인스턴스를 받습니다. provider도 같은 문제입니다. 설정 객체를
도입한 목적과 반대라서, cache key에 실제 생성 입력을 포함해 주세요.

**작업자 · 답변 :speech_balloon:**

맞습니다. 기존 cache는 backend별 재사용만 생각했고, 이번에 model/provider를 설정 객체로
드러내면서 그 전제가 더 이상 맞지 않게 됐습니다. embedding은 model 이름, LLM은 provider를
key에 포함하도록 수정하고 회귀 테스트도 추가하겠습니다.

**리뷰어 · 후속 :mag:**

`auto`와 명시적 `embedding`은 같은 model이면 같은 인스턴스를 공유해도 괜찮습니다. 둘은
생성 정책만 다르고 실제 backend 입력은 같기 때문입니다. 그 경우에도 key를 `choice`가 아니라
backend 종류와 model로 표현하면 의도가 더 분명할 것 같습니다.

**작업자 · 답변 :+1:**

동의합니다. `(backend, setting)` tuple로 바꾸겠습니다. 그래서 `auto`와 `embedding`은 같은
model이면 공유하고, model이나 provider가 다르면 반드시 새 인스턴스를 만듭니다.

**작업자 · 반영**

`c82fa23 fix(review-01): isolate matcher setting cache`

- embedding key: `("embedding", embedding_model)`
- LLM key: `("llm", llm_provider)`
- `auto`는 동일 model의 embedding cache를 재사용

`python3 -m unittest tests.test_semantic -q` 15건과 전체 93건 통과를 확인했습니다.

**리뷰어 · 확인 :white_check_mark:**

확인했습니다. cache 수명은 유지하면서도 인스턴스의 실제 생성 입력을 잃지 않습니다.
Change Request를 해결한 것으로 보입니다.

## 스레드 2 — 새 환경 경계의 회귀 테스트

**위치** `brainwash/semantic.py` (`MatcherSettings.from_environment`, 초기 PR 197-205행)
**심각도** `important` · **Change Request**

**리뷰어 · 댓글 :test_tube:**

이번 PR은 새로 읽는 세 환경 변수와 cache의 조합을 추가했지만, 최초 PR에는 이를 직접
검증하는 테스트가 없습니다. 현재 default test는 실행 머신의 `os.environ` 상태에 간접적으로
의존합니다.

model/provider가 바뀌는 두 호출과 빈 환경의 기본값을 각각 고정해 주세요. 테스트 중 환경을
바꿀 때는 [Python `unittest.mock.patch.dict` 설명](https://docs.python.org/3/library/unittest.mock.html#patch.dict)처럼
아래 블록 안에서만 환경 값을 바꾸고, 블록을 벗어나면 실행 전 `os.environ` 상태로 자동 복구되게
작성하면 다른 test에 설정이 새지 않습니다.

```python
with patch.dict(os.environ, {"BRAINWASH_MATCHER": "embedding"}, clear=True):
    matcher = get_matcher()
# 여기서는 원래 os.environ으로 복구됨
```

**작업자 · 답변 :memo:**

좋은 지적입니다. 설정 object만 추가하고 환경 계약을 명세로 남기지 않았습니다. `patch.dict`로
빈 환경과 backend별 환경을 분리하고, test 전후 `_ENV_CACHE`와 explicit matcher도 정리하겠습니다.

**리뷰어 · 후속 :eyes:**

LLM test는 실제 API import가 아니라 `_make_llm_matcher` seam을 대체하면 provider가 factory에
전달되는지까지 확인할 수 있겠습니다. 외부 환경 없이 cache 계약만 검증할 수 있어요.

**작업자 · 답변 :smile:**

그 방식으로 작성했습니다. embedding은 실제 `EmbeddingMatcher`의 `model_name`을 확인하고,
LLM은 factory 호출 목록이 `claude`, `openai` 순서인지 확인합니다. 네트워크나 optional package는
사용하지 않습니다.

**작업자 · 반영**

`58354ca test(review-01): cover matcher setting isolation`

세 regression test를 추가했습니다. 이 commit은 아직 shared default와 cache key 수정 전이라
Red 상태였고, 다음 `c82fa23`에서 함께 Green으로 만들었습니다.

**리뷰어 · 확인 :white_check_mark:**

환경 복구, cache 정리, model/provider 두 경로가 모두 드러납니다. host configuration에 우연히
의존하지 않는 테스트가 되어 좋습니다.

## 스레드 3 — 기본 model 값의 두 출처

**위치** `brainwash/semantic.py` (`EmbeddingMatcher.__init__` 66-74행, `MatcherSettings.from_environment` 197-205행)
**심각도** `important`

**리뷰어 · 댓글 :warning:**

기본 embedding model 문자열이 `EmbeddingMatcher`와 `MatcherSettings`에 각각 있습니다. 지금은
같지만, 한 곳만 바꾸면 explicit construction과 environment resolution이 다른 model을 선택하게
됩니다. 이 값은 module-level constant 하나로 공유하는 편이 안전해 보입니다.

**작업자 · 답변 :+1:**

맞습니다. 설정 객체를 도입하면서 기존 constructor의 default를 복제했습니다. 두 진입점이
같은 default를 쓴다는 계약을 constant와 test로 명시하겠습니다.

**리뷰어 · 후속 :bulb:**

그렇게 하면 future default 변경도 한 줄에서 끝나고, thread 2의 빈 환경 test가 두 경로의
일관성도 보호하게 되겠네요.

**작업자 · 반영**

`c82fa23 fix(review-01): isolate matcher setting cache`

`DEFAULT_EMBEDDING_MODEL`을 추가해 constructor와 `MatcherSettings`가 공유하도록 했습니다.

**리뷰어 · 확인 :white_check_mark:**

확인했습니다. 설정값의 단일 출처가 생겼고 빈 환경 회귀 test도 그 계약을 검증합니다.

## 스레드 4 — 알 수 없는 선택값의 호환성

**위치** `brainwash/semantic.py` (`MatcherSettings.resolve`, 초기 PR 207-223행)
**심각도** `question`

**리뷰어 · 질문 :question:**

`BRAINWASH_MATCHER=typo`는 `None`으로 처리됩니다. 설정 오류를 예외로 바꾸는 선택지도 있지만,
기존 `get_matcher()`도 unknown 값을 조용히 비활성화했습니다. 이번 refactor에서 의도적으로
그 계약을 유지한 것이 맞나요?

**작업자 · 답변 :speech_balloon:**

맞습니다. matcher는 opt-in 보조 기능이라 기본 retrieval을 막지 않는 것이 기존 정책입니다.
unknown 값을 예외로 바꾸면 배포 환경의 오타가 retrieval 전체 실패로 바뀌므로, 이번 구조 변경의
범위를 넘는다고 판단했습니다.

**리뷰어 · 후속 :+1:**

동의합니다. 이 PR의 목표는 configuration object 추출이지 운영 정책 변경이 아니므로, `None`
fallback을 유지하는 것이 호환성 측면에서 더 안전합니다. 변경 요청은 없습니다.

## 스레드 5 — explicit matcher와 cached backend의 우선순위

**위치** `brainwash/semantic.py` (`get_matcher` 232-236행, `set_matcher` 226-229행)
**심각도** `question`

**리뷰어 · 질문 :eyes:**

`MatcherSettings.from_environment()`를 호출하기 전에 `_ACTIVE`를 확인합니다. test fixture나
embedded caller가 explicit matcher를 설치했을 때 environment cache가 영향을 주지 않는다는
경계가 유지되는지 확인하고 싶습니다.

**작업자 · 답변 :speech_balloon:**

`get_matcher()`는 `_ACTIVE is not None`이면 즉시 반환합니다. 따라서 explicit matcher가 있는
동안에는 환경을 읽거나 cache factory를 호출하지 않습니다. `set_matcher(None)` 뒤에만 다시
environment resolution으로 돌아갑니다.

**리뷰어 · 후속 :white_check_mark:**

확인했습니다. override가 configuration object보다 앞에 있어 injected test double과 운영 설정의
책임이 섞이지 않습니다. 이 부분은 기존 계약을 유지한 것으로 보입니다.

## 최종 승인

**리뷰어 · Approve :tada:**

초기 PR의 설정 cache 누수와 테스트 공백을 구체적인 실행 경로로 확인했고, 후속 commit이
model/provider별 cache key, 기본값 단일화, 환경 격리 regression test로 해결했습니다.

호환성을 위한 unknown-value fallback과 explicit override 우선순위도 논의로 확인했습니다.
`tests.test_semantic` 15건과 전체 93건이 통과한 것을 확인하여 승인합니다.
