# Review-02 — conversation resolution history 검토

## 검토 요청

follow-up query 해석 결과와 당시 subject context를 in-memory history로 남기는 PR을 검토해
주세요. 초기 PR은 기존 resolver 동작을 유지하고 전체 test를 통과하지만, 새 history API의
불변성·snapshot·clear 의미를 직접 검증하는 test는 아직 없습니다.

## 검토 범위

- `brainwash/conversation.py`의 resolution record와 history 반환 계약
- frozen dataclass와 내부 mutable value의 경계
- `clear_history()`가 conversation subject와 log를 구분하는지
- JSON export와 in-memory immutable representation의 책임 분리

## 완료 조건

- 외부 호출자가 과거 resolution record나 resolver 내부 history를 변경할 수 없는지 검토합니다.
- resolution 시점의 subject 목록이 이후 대화에 의해 바뀌지 않는지 고정합니다.
- Change Request는 regression test와 후속 code commit으로 해결합니다.
- focused benchmark adapter test와 전체 test suite를 통과시킵니다.

## 제한

- 변경 파일은 `brainwash/conversation.py`, `tests/test_benchmark_adapters.py`에 한정합니다.
- 최초 PR은 기존 full suite를 통과하는 완결된 기능이어야 합니다.
- 리뷰 시작 뒤에는 reviewed commit을 고치지 않고 선형 response commit만 추가합니다.
