# feat: record benchmark request provenance

> Synthetic GitHub artifact: true  
> 최초 PR 시점의 설명입니다. 이후 리뷰 대화와 반영 commit은 포함하지 않습니다.

## 요약

CounterFact, KnowEdit, MQuAKE, RippleEdits adapter가 생성한 request에 원본 benchmark 좌표를
남깁니다. source dataset, source case, optional rewrite 위치를 metadata와 사람이 읽는 reference로
기록합니다.

## 주요 변경사항

- `BenchmarkProvenance`가 source, case ID, rewrite index를 표현합니다.
- `BenchmarkProvenance.from_row()`가 dataset row의 case ID 또는 fallback을 사용합니다.
- `BenchmarkRequestFactory`가 request metadata에 provenance를 넣습니다.
- CounterFact·KnowEdit·MQuAKE·RippleEdits loader가 provenance를 전달합니다.
- `benchmark_reference`는 `mquake:17#1`처럼 report와 audit log에서 읽을 수 있는 좌표를 제공합니다.

## 설계 의도

adapter 내부 row 형식을 downstream pipeline에 노출하지 않으면서, request가 어느 benchmark
sample에서 왔는지 추적할 수 있도록 Value Object를 도입했습니다. 각 loader는 row에서 provenance를
만들고, Factory가 `CorrectionRequest` metadata로 옮깁니다.

## Review Points

1. **metadata ownership** — benchmark provenance와 adapter-specific metadata가 같은 key를 가질 때
   어느 쪽이 권위를 가져야 하는지 검토 부탁드립니다.

2. **source coordinate 보존** — MQuAKE의 multi-rewrite row에서 skipped rewrite가 있어도 provenance가
   원본 row 위치를 설명하는지 확인 부탁드립니다.

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

기존 전체 test 96건이 통과했습니다. 최초 PR에는 provenance metadata의 우선순위와 source
coordinate를 직접 검증하는 test는 포함하지 않았습니다.

## Formatting

각 코드 커밋 직전에 Black 26.5.1을 적용했습니다. 변경된 Python 파일의 comment token과 module/class/function docstring은 제거했으며, SQL·script template·test fixture 같은 실행용 multiline string 값은 보존했습니다. 최초 PR snapshot을 포함한 최종 변경 파일은 `black --check`를 통과했고, 원본과 재작성본의 실행 AST도 동일합니다.

## Todos

- [ ] 리뷰 의견 반영
