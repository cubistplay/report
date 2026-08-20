# refactor: centralize matcher settings

> Synthetic GitHub artifact: true  
> 최초 PR 시점의 설명입니다. 이후 리뷰 대화와 반영 commit은 포함하지 않습니다.

## 요약

semantic matcher를 고르는 환경 변수와 backend별 입력값을 `MatcherSettings`로 모았습니다.
직접 설치한 matcher가 환경 설정보다 우선하는 기존 규칙은 유지합니다.

## 주요 변경사항

- `MatcherSettings.from_environment()`가 matcher 선택, embedding model, LLM provider를
  하나의 immutable 설정값으로 읽습니다.
- `MatcherSettings.resolve()`가 `lexical`, `embedding`, `llm`, `auto`, `off` 선택을
  해석합니다.
- `get_matcher()`는 explicit matcher 우선순위를 확인한 뒤 설정 객체에 해석을 위임합니다.
- backend 생성은 기존처럼 process cache를 사용해 매 호출마다 무거운 모델을 만들지 않도록
  유지합니다.

## 설계 의도

설정값을 읽는 코드와 backend 선택 분기를 하나의 Value Object에 모아, 환경 변수 계약을
호출자가 아닌 semantic module의 해석 경계에서 확인할 수 있게 했습니다. 이 변경은 새
backend를 추가할 때 `get_matcher()`의 제어 흐름을 계속 늘리지 않기 위한 작은
Configuration Object refactor입니다.

## Review Points

1. **설정과 cache의 책임 경계** — `MatcherSettings`가 backend 생성까지 맡는 구조와
   process cache의 key가 설정값을 충분히 표현하는지 검토 부탁드립니다.

2. **기존 선택 계약 보존** — explicit matcher 우선순위, `off`/`none`, `auto` fallback,
   optional dependency를 lazy하게 다루는 기존 동작이 유지되는지 확인 부탁드립니다.

## PR Type

- [ ] ✨ Feature
- [ ] 🐛 Bugfix
- [x] ♻️ Refactoring (no functional changes, no api changes)
- [ ] 🎨 Code style update
- [ ] 📚 Docs
- [ ] 🔧 Other

## 테스트

```bash
python3 -m unittest discover -s tests -q
```

기존 전체 테스트 90건이 통과했습니다. 이번 최초 PR에는 `MatcherSettings`의 환경·cache
계약을 직접 확인하는 신규 테스트는 포함하지 않았습니다.

## Todos

- [ ] 리뷰 의견 반영
