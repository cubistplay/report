# I-A5 PR 대화 — preference export 전략 분리

> Synthetic GitHub artifact: true  
> 최초 검토 head: `f7256bc` · Change Request 없음

## 스레드 1 — pair와 binary 행의 의미

**위치** `brainwash/algorithms/preference.py` (`PairedPreferenceRecords`, `BinaryPreferenceRecords`)

**리뷰어 · 질문 🔎**

paired 전략은 chosen/rejected가 모두 있어야 한 행을 만들고, binary 전략은 하나만 있어도
행을 만듭니다. 이 차이를 adapter 분기 대신 Strategy로 분리한 이유가 있나요?

**작업자 · 답변 💬**

두 방식은 같은 request를 받아도 학습 예시의 의미가 다릅니다. paired objective는 두 응답의
상대 비교가 필요하지만, KTO의 binary label은 각 응답을 독립적으로 사용합니다. 그래서
조건문을 공통 export에 두지 않고 행 변환 정책으로 분리했습니다.

**리뷰어 · 후속 질문 💭**

불완전한 pair를 skip한 수는 SimPO/DPO config에 남는데, binary 전략에는 같은 수가 없는
이유도 그 정책 차이 때문인가요?

**작업자 · 답변**

네. paired dataset에서는 요청 하나가 통째로 제외되므로 skip count가 데이터 손실을
설명합니다. binary dataset에서는 chosen 또는 rejected가 있으면 각각 유효한 행이므로
같은 의미의 skip count를 만들지 않았습니다.

**리뷰어 · 확인 ✅**

학습 목적에 따른 행 규칙과 관찰 지표가 strategy 안에 함께 있어 경계가 명확합니다.

## 스레드 2 — SimPO와 DPO가 공유하는 asset 범위

**위치** `brainwash/algorithms/preference.py` (`PairedPreferenceExportAdapter`)

**리뷰어 · 질문 🔎**

DPO가 SimPO script와 `simpo_config.json`을 계속 만들지만 command는 빈 목록으로
반환합니다. 공유 base class로 옮기면서 DPO를 SimPO 실행으로 바꾸지는 않았는지
확인하고 싶습니다.

**작업자 · 답변 💬**

공유한 것은 paired dataset과 기존 SimPO asset 생성 절차뿐입니다. DPO adapter는 그 asset을
준비한 뒤 `dpo_config.json`을 추가하고, 이전처럼 command를 비워 DPO trainer 선택을
호출자에게 남깁니다.

**리뷰어 · 후속 질문 💭**

그러면 DPO의 `simpo_config.json` 안 algorithm 값도 이전처럼 DPO여야겠네요.

**작업자 · 답변**

맞습니다. shared asset 메서드가 hard-coded SimPO 이름이 아니라 `self.name`을 사용합니다.
새 DPO 통합 테스트는 train file, DPO config, 빈 command를 확인하고 기존 artifact 형식도
전후 비교했습니다.

**리뷰어 · 확인 👍**

중복은 제거했지만 adapter별 실행 책임과 기존 config 계약은 유지됩니다.

## 스레드 3 — 공통 export의 변경 범위

**위치** `PreferenceExportAdapter._export_dataset`

**리뷰어 · 질문 🔎**

공통 base adapter가 directory 생성과 JSONL 기록을 맡습니다. KTO까지 같은 export 경로를
쓰게 되면서 paired training script가 잘못 생성될 가능성은 없나요?

**작업자 · 답변 💬**

base adapter는 strategy가 만든 rows를 `train_filename`에 쓰는 역할만 합니다. script와
paired config는 `PairedPreferenceExportAdapter`에만 있어 KTO는 `train_kto.jsonl`과
`kto_config.json`만 만듭니다.

**리뷰어 · 후속 질문 💭**

구조 분리 후 실제 output 전체도 비교했나요?

**작업자 · 답변**

네. SimPO·DPO·KTO의 manifest와 JSON/JSONL artifact를 임시 output 경로만 정규화한 뒤
변경 전과 비교했습니다. `diff` 출력은 없었고, KTO 통합 테스트도 SimPO script 부재를
명시적으로 확인합니다.

**리뷰어 · 확인 📌**

공통화한 범위와 adapter별로 남긴 asset 생성 책임이 테스트와 출력 비교로 확인됩니다.

## 승인

**리뷰어 · Approve ✅**

paired/binary 데이터 의미를 Strategy로 분리했고, 공통 export와 shared paired asset
생성 범위가 명확합니다. 세 adapter의 artifact 동등성까지 확인되어 승인합니다.
