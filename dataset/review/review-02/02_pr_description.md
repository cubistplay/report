# feat: record conversation resolutions

> Synthetic GitHub artifact: true  
> 최초 PR 시점의 설명입니다. 이후 리뷰 대화와 반영 commit은 포함하지 않습니다.

## 요약

`ConversationResolver`가 follow-up query를 해석할 때 원문, rewritten query, 상태, 당시
subject context를 기록하도록 했습니다. `clear_history()`는 resolution 기록만 비우고 다음
follow-up 해석에 필요한 subject context는 유지합니다.

## 주요 변경사항

- `ConversationResolutionRecord`가 resolution 결과와 known subject 목록을 담습니다.
- 모든 `resolve()` 결과(`unchanged`, `unresolved`, `ambiguous`, `resolved`)를 발생 순서대로
  history에 기록합니다.
- `history` property로 기록을 조회할 수 있습니다.
- `to_dict()`로 이후 JSON export가 가능한 형태를 제공합니다.
- `clear_history()`로 로그만 비울 수 있습니다.

## 설계 의도

이력은 resolver가 결정을 내린 시점의 context를 설명하는 audit data입니다. 따라서 record 생성은
각 분기에서 흩어지지 않도록 `_record()` Template Method에 모으고, `resolve()`는 기존 판단과
rewrite 책임을 유지합니다.

## Review Points

1. **history API 경계** — 외부 caller가 조회한 history와 record를 어떤 수준까지 변경하지
   못하게 해야 하는지, audit snapshot의 책임이 적절한지 확인 부탁드립니다.

2. **log와 context 분리** — `clear_history()`가 기록만 지우고 subject context는 유지하는
   계약이 interactive conversation에서 자연스러운지 검토 부탁드립니다.

## PR Type

- [x] ✨ Feature
- [ ] 🐛 Bugfix
- [ ] ♻️ Refactoring (no functional changes, no api changes)
- [ ] 🎨 Code style update
- [ ] 📚 Docs
- [ ] 🔧 Other

## 테스트

```bash
python3 -m unittest discover -s tests -q
```

기존 전체 test 93건이 통과했습니다. 최초 PR에는 새 history API의 mutation·snapshot 계약을
직접 검증하는 test는 포함하지 않았습니다.

## Formatting

각 코드 커밋 직전에 Black 26.5.1을 적용했습니다. 변경된 Python 파일의 comment token과 module/class/function docstring은 제거했으며, SQL·script template·test fixture 같은 실행용 multiline string 값은 보존했습니다. 최초 PR snapshot을 포함한 최종 변경 파일은 `black --check`를 통과했고, 원본과 재작성본의 실행 AST도 동일합니다.

## Todos

- [ ] 리뷰 의견 반영
