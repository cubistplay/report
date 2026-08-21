# feat: split routing lanes

> Synthetic GitHub artifact: true  
> 최초 PR 시점의 설명입니다. 이후 리뷰 대화와 반영 commit은 포함하지 않습니다.

## 요약

mixed correction batch를 behavioral lane과 memory-routing lane으로 분리하고, 각 lane에 독립적인 AlgorithmPlan을 생성합니다.

## 주요 변경사항

- RoutingLane Value Object가 lane name, request IDs, routing plan을 표현합니다.
- RoutingLaneBatch가 lane collection과 전체 request count를 직렬화합니다.
- BrainwashRouter.route_lanes()가 mixed input을 lane별로 나눈 뒤 기존 route()를 재사용합니다.
- empty input은 기존 route()와 같은 ValueError 계약을 유지합니다.

## 설계 의도

Composite Result 패턴으로 하나의 mixed batch의 lane별 routing 결과를 표현했습니다. 기존 RoutingPolicy Strategy는 그대로 유지하고, route_lanes는 policy 선택 전에 입력만 분리하는 Orchestrator 역할을 맡습니다.

~~~mermaid
flowchart LR
  B[mixed requests] --> R[route]
  R --> P[one mixed AlgorithmPlan]
~~~

~~~mermaid
flowchart LR
  B[mixed requests] --> L[route_lanes]
  L --> H[behavior lane]
  L --> M[memory lane]
  H --> P1[AlgorithmPlan]
  M --> P2[AlgorithmPlan]
~~~

## Review Points

1. **lane ownership** — behavioral, factual, domain/broad correction이 서로의 routing threshold에 영향을 주지 않도록 분리됐는지 검토 부탁드립니다.
2. **reassembly contract** — lane 결과를 original input과 다시 연결할 수 있는 metadata가 충분한지 확인 부탁드립니다.

## PR Type

- [x] ✨ Feature
- [ ] 🐛 Bugfix
- [ ] ♻️ Refactoring (no functional changes, no api changes)
- [ ] 🎨 Code style update
- [ ] 📚 Docs
- [ ] 🔧 Other

## 테스트

    python3 -m unittest discover -s tests -q

기존 전체 test 128건이 통과했습니다. 최초 PR에는 domain/fact mixed lane의 plan 분리와 original request position을 직접 검증하는 test는 포함하지 않았습니다.

## Formatting

각 코드 커밋 직전에 Black 26.5.1을 적용했습니다. 변경된 Python 파일의 comment token과 module/class/function docstring은 제거했으며, SQL·script template·test fixture 같은 실행용 multiline string 값은 보존했습니다. 최초 PR snapshot을 포함한 최종 변경 파일은 `black --check`를 통과했고, 원본과 재작성본의 실행 AST도 동일합니다.

## Todos

- [ ] 리뷰 의견 반영
