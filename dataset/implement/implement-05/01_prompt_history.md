# 프롬프트 히스토리 — I-A5

> 실제 대화 export가 아니라, 이 PR을 만들기 위한 가상 개발 프롬프트 시나리오입니다.
> 구현과 테스트 결과를 기준으로 작성했습니다.

## Prompt 1 — preference 행 규칙 정리

**Developer**

> SimPO랑 DPO는 chosen/rejected pair가 있어야 하고 KTO는 좋은 예시나 나쁜 예시 하나만
> 있어도 행을 만들잖아. 지금은 adapter마다 직접 돌고 있는데, 공통 export는 묶되 이
> 차이는 안 섞이게 정리해줘.

**Codex**

> 행 생성 규칙을 paired/binary Strategy로 분리하고, adapter는 전략이 만든 dataset을
> 공통 export 메서드로 쓰게 하겠습니다. SimPO/DPO가 공유하는 script·config 생성도
> 별도 Template Method로 만들겠습니다.

## Prompt 2 — 테스트 우선 계약

**Developer**

> pair가 하나라도 없으면 SimPO/DPO는 skip count가 올라가야 하고, KTO는 chosen만 있어도
> positive row가 나와야 해. DPO는 simpo asset을 만들되 command는 비워 두는 지금 동작도
> 그대로 확인해줘.

**작업 단계 — Red 테스트**

- paired record의 완전한 pair 필수와 skip count
- binary record의 chosen/rejected 독립 행 생성
- DPO의 paired asset 재사용과 KTO의 paired asset 미생성

```bash
python3 -m unittest tests.test_pipeline_preference_export -q
# ImportError: 아직 PairedPreferenceRecords 등이 없습니다.
```

```bash
python3 -m unittest tests.test_pipeline -q
# Ran 5 tests — OK
```

Commit: `4148780 test(implement-05): specify preference record strategies`

## Prompt 3 — 동작 보존 확인

**Developer**

> 이건 export 구조만 바꾸는 거니까 SimPO, DPO, KTO 각각 output 파일과 manifest를 전후로
> 비교해줘. 경로만 달라지는 부분은 빼고 JSON/JSONL 내용이 같아야 해.

**작업 단계 — Green 검증**

```bash
python3 -m unittest tests.test_pipeline_preference_export -q
# Ran 4 tests — OK

python3 -m unittest tests.test_pipeline -q
# Ran 5 tests — OK

python3 -m unittest discover -s tests -q
# Ran 90 tests — OK
```

SimPO·DPO·KTO의 manifest와 JSON/JSONL artifact에서 임시 output 경로만 정규화해
변경 전 `067cbce`와 비교했고 `diff` 출력은 없었습니다.

Commit: `cd8199f refactor(implement-05): separate preference export strategies`
