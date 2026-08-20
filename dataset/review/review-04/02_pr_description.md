# feat: report SFT dataset preparation

> Synthetic GitHub artifact: true  
> 최초 PR 시점의 설명입니다. 이후 리뷰 대화와 반영 commit은 포함하지 않습니다.

## 요약

QLoRA/LoRA SFT adapter가 training JSONL을 만들 때 usable row와 skip 수를 `sft_dataset_summary.json`에
기록합니다. config와 run note도 같은 summary를 사용해 training data 상태를 보여 줍니다.

## 주요 변경사항

- `SftDatasetSummary`가 input count, training row count, skip count, usable ratio를 표현합니다.
- empty dataset은 usable ratio `0.0`과 명시적인 note를 남깁니다.
- QLoRA/LoRA adapter가 summary JSON을 생성하고 config에 summary 값을 포함합니다.
- run note가 `Prepared 12/20 usable SFT rows.` 형태로 data preparation 결과를 보여 줍니다.

## 설계 의도

training script를 실행하기 전에도 data preparation에서 몇 개의 요청이 실제 SFT 행이 되었는지
확인할 수 있어야 합니다. count 계산과 note 형식을 Value Object에 모아 QLoRA와 LoRA가 같은
summary 계약을 쓰도록 했습니다.

## Review Points

1. **count 의미** — total input, emitted row, skipped request의 책임과 usable ratio 분모가
   training data 손실을 충분히 설명하는지 검토 부탁드립니다.

2. **artifact discoverability** — summary JSON을 만들기만 하는 것이 아니라 pipeline manifest에서
   찾을 수 있도록 `AlgorithmRunResult`에 노출해야 하는지 확인 부탁드립니다.

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

기존 전체 test 98건이 통과했습니다. 최초 PR에는 summary count와 manifest artifact를 직접
검증하는 test는 포함하지 않았습니다.

## Todos

- [ ] 리뷰 의견 반영
