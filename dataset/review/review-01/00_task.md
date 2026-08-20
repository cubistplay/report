# Review-01 — semantic matcher 설정 경계 검토

## 검토 요청

`BRAINWASH_MATCHER`, embedding model, LLM provider를 `MatcherSettings`로 모은
refactor PR을 검토해 주세요. 이 PR은 기존 전체 테스트만 통과하며, 새 설정 객체에 대한
테스트는 아직 없습니다.

## 검토 범위

- `brainwash/semantic.py`의 환경 변수 해석, matcher 생성, 프로세스 캐시 경계
- 설정 변경이 long-lived process와 테스트 실행에서 어떤 결과를 내는지
- 기본 모델 값의 단일 출처와 override matcher 우선순위

## 완료 조건

- 발견 사항마다 코드 위치, 영향, 구체적인 개선 방향을 제시합니다.
- Change Request는 회귀 테스트와 별도 후속 commit으로 해결합니다.
- 설정이 다른 embedding model과 LLM provider를 캐시에서 구분하는지 검증합니다.
- focused semantic test와 전체 test suite를 다시 통과시킵니다.

## 제한

- 변경 파일은 `brainwash/semantic.py`, `tests/test_semantic.py`에 한정합니다.
- 초기 PR은 테스트 없이도 기존 전체 suite를 통과하는 완결된 refactor여야 합니다.
- 리뷰 시작 뒤에는 history를 고치지 않고 response commit을 선형으로 쌓습니다.
